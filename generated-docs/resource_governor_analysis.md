# SQL Server Resource Governor Analysis Framework

## Resource Governor Monitoring

### Resource Pool Analysis
```sql
CREATE TABLE dbo.ResourcePoolMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    PoolName nvarchar(128),
    WorkloadGroupName nvarchar(128),
    CPUUsagePercent decimal(5,2),
    MemoryUsageMB decimal(18,2),
    RequestCount int,
    ActiveRequestCount int,
    QueuedRequestCount int,
    AverageCPULatencyMs decimal(18,2),
    AverageMemoryGrantLatencyMs decimal(18,2),
    MaxMemoryGrantTimeoutCount int,
    MaxConcurrentQueries int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorResourcePools
    @HighCPUThreshold decimal(5,2) = 80.0,
    @HighMemoryThreshold decimal(5,2) = 90.0
AS
BEGIN
    -- Capture Resource Governor metrics
    INSERT INTO dbo.ResourcePoolMetrics
    SELECT 
        rp.name as PoolName,
        wg.name as WorkloadGroupName,
        rs.cpu_usage_percent as CPUUsagePercent,
        rs.memory_usage_kb / 1024.0 as MemoryUsageMB,
        rs.request_count,
        rs.active_request_count,
        rs.queued_request_count,
        rs.cpu_delayed_ms * 1.0 / 
            NULLIF(rs.request_count, 0) as AverageCPULatencyMs,
        rs.max_memory_grant_timeout_ms * 1.0 / 
            NULLIF(rs.request_count, 0) as AverageMemoryGrantLatencyMs,
        rs.max_memory_grant_timeout_count,
        wg.max_dop,
        GETUTCDATE()
    FROM sys.dm_resource_governor_resource_pools rp
    JOIN sys.dm_resource_governor_workload_groups wg 
        ON rp.pool_id = wg.pool_id
    JOIN sys.dm_resource_governor_resource_pool_stats rs 
        ON rp.pool_id = rs.pool_id;

    -- Analyze resource utilization patterns
    WITH PoolMetrics AS (
        SELECT 
            PoolName,
            WorkloadGroupName,
            CPUUsagePercent,
            MemoryUsageMB,
            RequestCount,
            ActiveRequestCount,
            QueuedRequestCount,
            AverageCPULatencyMs,
            AverageMemoryGrantLatencyMs,
            MaxMemoryGrantTimeoutCount,
            MaxConcurrentQueries,
            LAG(CPUUsagePercent) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime
            ) as PreviousCPUUsage,
            LAG(MemoryUsageMB) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime
            ) as PreviousMemoryUsage
        FROM dbo.ResourcePoolMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        PoolName,
        WorkloadGroupName,
        CPUUsagePercent,
        MemoryUsageMB,
        RequestCount,
        ActiveRequestCount,
        QueuedRequestCount,
        AverageCPULatencyMs,
        AverageMemoryGrantLatencyMs,
        MaxMemoryGrantTimeoutCount,
        CASE 
            WHEN CPUUsagePercent > @HighCPUThreshold 
                 AND MemoryUsageMB > @HighMemoryThreshold 
            THEN 'Critical Resource Pressure'
            WHEN CPUUsagePercent > @HighCPUThreshold 
            THEN 'High CPU Usage'
            WHEN MemoryUsageMB > @HighMemoryThreshold 
            THEN 'High Memory Usage'
            WHEN QueuedRequestCount > 10 
            THEN 'Request Queuing'
            ELSE 'Normal'
        END as PoolStatus,
        CASE 
            WHEN CPUUsagePercent > @HighCPUThreshold 
                 AND MemoryUsageMB > @HighMemoryThreshold 
            THEN 'Consider:
                  1. Adjusting pool limits
                  2. Moving workloads
                  3. Adding resources'
            WHEN CPUUsagePercent > @HighCPUThreshold 
            THEN 'Review CPU allocation'
            WHEN MemoryUsageMB > @HighMemoryThreshold 
            THEN 'Review memory allocation'
            WHEN QueuedRequestCount > 10 
            THEN 'Check concurrent query limits'
            ELSE 'No action needed'
        END as Recommendation
    FROM PoolMetrics
    WHERE CPUUsagePercent > @HighCPUThreshold
    OR MemoryUsageMB > @HighMemoryThreshold
    OR QueuedRequestCount > 10
    OR CPUUsagePercent > COALESCE(PreviousCPUUsage, 0) * 1.5
    OR MemoryUsageMB > COALESCE(PreviousMemoryUsage, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN CPUUsagePercent > @HighCPUThreshold 
                 AND MemoryUsageMB > @HighMemoryThreshold THEN 1
            WHEN CPUUsagePercent > @HighCPUThreshold THEN 2
            WHEN MemoryUsageMB > @HighMemoryThreshold THEN 3
            ELSE 4
        END,
        CPUUsagePercent DESC;
END;
```

### Workload Classification Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeWorkloadClassification
AS
BEGIN
    -- Analyze workload classification patterns
    SELECT 
        wc.name as ClassifierName,
        wgm.name as MemberName,
        wg.name as WorkloadGroupName,
        rp.name as PoolName,
        COUNT(DISTINCT s.session_id) as ActiveSessions,
        AVG(r.cpu_time) as AvgCPUTime,
        AVG(r.reads) as AvgReads,
        AVG(r.writes) as AvgWrites,
        AVG(r.logical_reads) as AvgLogicalReads,
        COUNT(
            CASE 
                WHEN r.wait_type IS NOT NULL 
                THEN 1 
            END
        ) as WaitingRequests,
        STRING_AGG(
            DISTINCT r.wait_type, 
            ', '
        ) as WaitTypes,
        CASE 
            WHEN COUNT(DISTINCT s.session_id) > 100 
            THEN 'High Session Count'
            WHEN AVG(r.cpu_time) > 1000000 
            THEN 'High CPU Usage'
            WHEN COUNT(
                CASE 
                    WHEN r.wait_type IS NOT NULL 
                    THEN 1 
                END
            ) > 10 
            THEN 'High Wait Count'
            ELSE 'Normal'
        END as ClassificationPattern,
        CASE 
            WHEN COUNT(DISTINCT s.session_id) > 100 
            THEN 'Review session distribution'
            WHEN AVG(r.cpu_time) > 1000000 
            THEN 'Analyze workload patterns'
            WHEN COUNT(
                CASE 
                    WHEN r.wait_type IS NOT NULL 
                    THEN 1 
                END
            ) > 10 
            THEN 'Investigate wait causes'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.resource_governor_workload_groups wg
    JOIN sys.resource_governor_resource_pools rp 
        ON wg.pool_id = rp.pool_id
    JOIN sys.resource_governor_workload_group_members wgm 
        ON wg.group_id = wgm.group_id
    JOIN sys.resource_governor_configuration_windows wc 
        ON wgm.classifier_id = wc.classifier_id
    LEFT JOIN sys.dm_exec_sessions s 
        ON s.group_id = wg.group_id
    LEFT JOIN sys.dm_exec_requests r 
        ON s.session_id = r.session_id
    GROUP BY 
        wc.name,
        wgm.name,
        wg.name,
        rp.name
    HAVING COUNT(DISTINCT s.session_id) > 50
    OR AVG(r.cpu_time) > 1000000
    OR COUNT(
        CASE 
            WHEN r.wait_type IS NOT NULL 
            THEN 1 
        END
    ) > 10
    ORDER BY 
        CASE 
            WHEN COUNT(DISTINCT s.session_id) > 100 THEN 1
            WHEN AVG(r.cpu_time) > 1000000 THEN 2
            ELSE 3
        END,
        ActiveSessions DESC;
END;
```

### Resource Usage Trending
```sql
CREATE PROCEDURE dbo.AnalyzeResourceTrends
    @TimeWindowMinutes int = 60
AS
BEGIN
    -- Analyze resource usage trends
    WITH ResourceTrends AS (
        SELECT 
            PoolName,
            WorkloadGroupName,
            CollectionTime,
            CPUUsagePercent,
            MemoryUsageMB,
            RequestCount,
            LAG(CPUUsagePercent, 6) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime
            ) as CPUUsage10MinAgo,
            LAG(MemoryUsageMB, 6) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime
            ) as MemoryUsage10MinAgo,
            AVG(CPUUsagePercent) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime 
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) as AvgCPULast10Min,
            AVG(MemoryUsageMB) OVER (
                PARTITION BY PoolName 
                ORDER BY CollectionTime 
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) as AvgMemoryLast10Min
        FROM dbo.ResourcePoolMetrics
        WHERE CollectionTime >= DATEADD(
            MINUTE, 
            -@TimeWindowMinutes, 
            GETUTCDATE()
        )
    )
    SELECT 
        PoolName,
        WorkloadGroupName,
        MAX(CPUUsagePercent) as PeakCPUUsage,
        MAX(MemoryUsageMB) as PeakMemoryUsageMB,
        AVG(CPUUsagePercent) as AvgCPUUsage,
        AVG(MemoryUsageMB) as AvgMemoryUsageMB,
        MAX(RequestCount) as PeakRequestCount,
        MAX(AvgCPULast10Min) - MIN(AvgCPULast10Min) as CPUVariation,
        MAX(AvgMemoryLast10Min) - MIN(AvgMemoryLast10Min) as MemoryVariation,
        CASE 
            WHEN MAX(CPUUsagePercent) > 
                 AVG(CPUUsagePercent) * 2 
            THEN 'CPU Spikes'
            WHEN MAX(MemoryUsageMB) > 
                 AVG(MemoryUsageMB) * 2 
            THEN 'Memory Spikes'
            WHEN MAX(AvgCPULast10Min) - 
                 MIN(AvgCPULast10Min) > 30 
            THEN 'High CPU Variation'
            WHEN MAX(AvgMemoryLast10Min) - 
                 MIN(AvgMemoryLast10Min) > 1024 
            THEN 'High Memory Variation'
            ELSE 'Stable'
        END as TrendPattern,
        CASE 
            WHEN MAX(CPUUsagePercent) > 
                 AVG(CPUUsagePercent) * 2 
            THEN 'Investigate CPU spike causes'
            WHEN MAX(MemoryUsageMB) > 
                 AVG(MemoryUsageMB) * 2 
            THEN 'Review memory usage patterns'
            WHEN MAX(AvgCPULast10Min) - 
                 MIN(AvgCPULast10Min) > 30 
            THEN 'Analyze CPU variations'
            WHEN MAX(AvgMemoryLast10Min) - 
                 MIN(AvgMemoryLast10Min) > 1024 
            THEN 'Monitor memory fluctuations'
            ELSE 'No action needed'
        END as Recommendation
    FROM ResourceTrends
    GROUP BY 
        PoolName,
        WorkloadGroupName
    HAVING MAX(CPUUsagePercent) > AVG(CPUUsagePercent) * 2
    OR MAX(MemoryUsageMB) > AVG(MemoryUsageMB) * 2
    OR MAX(AvgCPULast10Min) - MIN(AvgCPULast10Min) > 30
    OR MAX(AvgMemoryLast10Min) - MIN(AvgMemoryLast10Min) > 1024
    ORDER BY 
        CASE 
            WHEN MAX(CPUUsagePercent) > 
                 AVG(CPUUsagePercent) * 2 THEN 1
            WHEN MAX(MemoryUsageMB) > 
                 AVG(MemoryUsageMB) * 2 THEN 2
            ELSE 3
        END,
        PeakCPUUsage DESC;
END;
```

This Resource Governor analysis framework provides comprehensive tools for:
1. Monitoring resource pool utilization and performance
2. Analyzing workload classification effectiveness
3. Tracking resource usage trends and patterns
4. Optimizing workload management

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
