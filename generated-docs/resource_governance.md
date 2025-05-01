# SQL Server Resource Governance Framework

## Workload Classification

### Workload Group Configuration
```sql
-- Create workload tracking
CREATE TABLE dbo.WorkloadGroups
(
    GroupId int IDENTITY(1,1) PRIMARY KEY,
    GroupName nvarchar(128),
    MaxDOP int,
    MinMemoryPercent decimal(5,2),
    MaxMemoryPercent decimal(5,2),
    RequestMaxMemoryGrantPercent decimal(5,2),
    MaxCPUPercent decimal(5,2),
    GroupPriority int,
    IsEnabled bit
);

-- Create workload classification
CREATE TABLE dbo.WorkloadClassification
(
    ClassificationId int IDENTITY(1,1) PRIMARY KEY,
    GroupId int,
    ApplicationName nvarchar(128),
    UserName nvarchar(128),
    DatabaseName sysname,
    ClassificationRule nvarchar(max),
    Priority int,
    CONSTRAINT FK_Classification_Group 
        FOREIGN KEY (GroupId) 
        REFERENCES dbo.WorkloadGroups(GroupId)
);

-- Initialize default groups
INSERT INTO dbo.WorkloadGroups
(GroupName, MaxDOP, MinMemoryPercent, MaxMemoryPercent, 
 RequestMaxMemoryGrantPercent, MaxCPUPercent, GroupPriority, IsEnabled)
VALUES
('ETL_Workload', 4, 20, 40, 25, 50, 100, 1),
('OLTP_Workload', 2, 30, 60, 15, 70, 200, 1),
('Reporting_Workload', 6, 10, 30, 20, 40, 50, 1),
('Maintenance_Workload', 8, 5, 20, 30, 30, 25, 1);
```

### Resource Pool Management
```sql
CREATE TABLE dbo.ResourcePoolHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    PoolName nvarchar(128),
    ResourceType varchar(20),
    AllocatedValue decimal(18,2),
    UsedValue decimal(18,2),
    WaitingTasks int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorResourcePools
AS
BEGIN
    -- Capture current resource usage
    INSERT INTO dbo.ResourcePoolHistory
    SELECT 
        p.name as PoolName,
        'CPU' as ResourceType,
        p.cap_cpu_percent as AllocatedValue,
        s.cpu_usage_ms * 100.0 / 
            NULLIF(s.cpu_usage_ms + s.cpu_idle_ms, 0) as UsedValue,
        s.active_requests_count as WaitingTasks,
        GETUTCDATE()
    FROM sys.dm_resource_governor_resource_pools p
    JOIN sys.dm_resource_governor_workload_groups g
        ON p.pool_id = g.pool_id
    JOIN sys.dm_resource_governor_resource_pool_stats s
        ON p.pool_id = s.pool_id;

    -- Memory metrics
    INSERT INTO dbo.ResourcePoolHistory
    SELECT 
        p.name as PoolName,
        'Memory' as ResourceType,
        p.max_memory_percent as AllocatedValue,
        s.used_memory_kb * 100.0 / 
            NULLIF(s.target_memory_kb, 0) as UsedValue,
        s.pending_memory_kb as WaitingTasks,
        GETUTCDATE()
    FROM sys.dm_resource_governor_resource_pools p
    JOIN sys.dm_resource_governor_workload_groups g
        ON p.pool_id = g.pool_id
    JOIN sys.dm_resource_governor_resource_pool_stats s
        ON p.pool_id = s.pool_id;
END;
```

## Dynamic Resource Management

### Automated Resource Adjustment
```sql
CREATE PROCEDURE dbo.AdjustResourceAllocation
    @ResourceType varchar(20),
    @HighUtilizationThreshold decimal(5,2) = 85.0,
    @LowUtilizationThreshold decimal(5,2) = 30.0
AS
BEGIN
    -- Analyze resource usage patterns
    WITH PoolUtilization AS (
        SELECT 
            PoolName,
            AVG(UsedValue) as AvgUtilization,
            MAX(UsedValue) as PeakUtilization,
            AVG(WaitingTasks) as AvgWaitingTasks
        FROM dbo.ResourcePoolHistory
        WHERE 
            ResourceType = @ResourceType
            AND CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY PoolName
    )
    SELECT 
        pu.PoolName,
        pu.AvgUtilization,
        pu.PeakUtilization,
        pu.AvgWaitingTasks,
        CASE 
            WHEN pu.AvgUtilization > @HighUtilizationThreshold 
                 AND pu.AvgWaitingTasks > 0
            THEN 'Increase'
            WHEN pu.PeakUtilization < @LowUtilizationThreshold
            THEN 'Decrease'
            ELSE 'NoChange'
        END as AdjustmentNeeded
    FROM PoolUtilization pu;

    -- Generate adjustment commands
    SELECT 
        'ALTER RESOURCE GOVERNOR RESOURCE POOL ' + 
        QUOTENAME(PoolName) + 
        CASE @ResourceType
            WHEN 'CPU' THEN 
                ' WITH (CAP_CPU_PERCENT = ' + 
                CAST(CEILING(current_cap * 1.2) as varchar(3)) + ')'
            WHEN 'Memory' THEN 
                ' WITH (MAX_MEMORY_PERCENT = ' + 
                CAST(CEILING(current_max * 1.2) as varchar(3)) + ')'
        END as AdjustmentCommand
    FROM sys.dm_resource_governor_resource_pools
    WHERE name IN (
        SELECT PoolName 
        FROM PoolUtilization 
        WHERE AdjustmentNeeded = 'Increase'
    );
END;
```

### Workload Balancing
```sql
CREATE PROCEDURE dbo.BalanceWorkloads
AS
BEGIN
    -- Analyze workload distribution
    WITH WorkloadStats AS (
        SELECT 
            g.name as GroupName,
            COUNT(*) as RequestCount,
            AVG(r.cpu_time) as AvgCPUTime,
            AVG(r.logical_reads) as AvgLogicalReads,
            AVG(r.writes) as AvgWrites,
            SUM(mg.requested_memory_kb) as TotalMemoryRequested
        FROM sys.dm_exec_requests r
        JOIN sys.dm_resource_governor_workload_groups g
            ON r.group_id = g.group_id
        LEFT JOIN sys.dm_exec_query_memory_grants mg
            ON r.session_id = mg.session_id
        GROUP BY g.name
    )
    SELECT 
        ws.*,
        CASE 
            WHEN AvgCPUTime > 
                 2 * (SELECT AVG(AvgCPUTime) FROM WorkloadStats)
            THEN 'High CPU'
            WHEN TotalMemoryRequested > 
                 2 * (SELECT AVG(TotalMemoryRequested) FROM WorkloadStats)
            THEN 'High Memory'
            ELSE 'Balanced'
        END as WorkloadStatus
    FROM WorkloadStats ws;

    -- Generate rebalancing recommendations
    SELECT 
        CASE 
            WHEN WorkloadStatus = 'High CPU'
            THEN 'Consider reducing MAX_DOP or increasing CAP_CPU_PERCENT'
            WHEN WorkloadStatus = 'High Memory'
            THEN 'Consider adjusting REQUEST_MAX_MEMORY_GRANT_PERCENT'
            ELSE 'No action needed'
        END as Recommendation,
        GroupName,
        RequestCount,
        AvgCPUTime,
        TotalMemoryRequested
    FROM WorkloadStats
    WHERE WorkloadStatus != 'Balanced';
END;
```

### Performance Analysis Integration
```sql
CREATE PROCEDURE dbo.AnalyzeResourceImpact
    @StartTime datetime2,
    @EndTime datetime2
AS
BEGIN
    -- Correlate resource usage with performance
    WITH ResourceMetrics AS (
        SELECT 
            rh.PoolName,
            rh.ResourceType,
            rh.UsedValue,
            qm.avg_duration,
            qm.avg_logical_io_reads,
            qm.avg_logical_io_writes,
            qm.avg_cpu_time,
            qm.avg_query_max_used_memory
        FROM dbo.ResourcePoolHistory rh
        CROSS APPLY (
            SELECT 
                AVG(rs.avg_duration) as avg_duration,
                AVG(rs.avg_logical_io_reads) as avg_logical_io_reads,
                AVG(rs.avg_logical_io_writes) as avg_logical_io_writes,
                AVG(rs.avg_cpu_time) as avg_cpu_time,
                AVG(rs.avg_query_max_used_memory) as avg_query_max_used_memory
            FROM sys.query_store_runtime_stats rs
            JOIN sys.query_store_plan p
                ON rs.plan_id = p.plan_id
            WHERE rs.last_execution_time 
                BETWEEN rh.CollectionTime 
                AND DATEADD(MINUTE, 5, rh.CollectionTime)
        ) qm
        WHERE rh.CollectionTime 
            BETWEEN @StartTime AND @EndTime
    )
    SELECT 
        PoolName,
        ResourceType,
        AVG(UsedValue) as AvgResourceUsage,
        AVG(avg_duration) as AvgQueryDuration,
        AVG(avg_cpu_time) as AvgCPUTime,
        AVG(avg_logical_io_reads) as AvgReads,
        AVG(avg_logical_io_writes) as AvgWrites,
        CORR(UsedValue, avg_duration) as ResourceDurationCorrelation
    FROM ResourceMetrics
    GROUP BY PoolName, ResourceType
    ORDER BY AvgResourceUsage DESC;
END;
```

This resource governance framework provides detailed control and monitoring of SQL Server resources, helping to maintain optimal performance across different workload types. Would you like me to add more specific components for handling particular resource types or focus on another aspect of the implementation?
