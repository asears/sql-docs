# SQL Server Compatibility Mode Monitoring

## Query Performance Analysis

### Compatibility Level Impact

1. Performance Regression Detection
```sql
CREATE TABLE dbo.CompatibilityPerformance
(
    MonitorId bigint IDENTITY(1,1) PRIMARY KEY,
    QueryId bigint,
    QueryText nvarchar(max),
    CompatibilityLevel int,
    ExecutionCount int,
    AvgCPUTime bigint,
    AvgDuration bigint,
    AvgLogicalReads bigint,
    PlanHash binary(32),
    CollectionDate datetime2,
    HasRegressed bit
);

-- Create monitoring procedure
CREATE PROCEDURE dbo.MonitorCompatibilityImpact
    @MinExecutionCount int = 10,
    @RegressionThreshold decimal(5,2) = 20.0
AS
BEGIN
    -- Capture current performance metrics
    INSERT INTO dbo.CompatibilityPerformance
    (QueryId, QueryText, CompatibilityLevel, 
     ExecutionCount, AvgCPUTime, AvgDuration,
     AvgLogicalReads, PlanHash, CollectionDate)
    SELECT 
        q.query_id,
        qt.query_sql_text,
        qp.compatibility_level,
        rs.count_executions,
        rs.avg_cpu_time,
        rs.avg_duration,
        rs.avg_logical_io_reads,
        HASHBYTES('SHA2_256', qp.query_plan),
        GETUTCDATE()
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
    ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan qp 
    ON q.query_id = qp.query_id
    JOIN sys.query_store_runtime_stats rs 
    ON qp.plan_id = rs.plan_id
    WHERE rs.count_executions >= @MinExecutionCount;
    
    -- Detect regressions
    WITH BaselinePerf AS (
        SELECT 
            QueryId,
            CompatibilityLevel,
            AVG(AvgDuration) as BaselineDuration
        FROM dbo.CompatibilityPerformance
        WHERE CompatibilityLevel < 160  -- Pre-SQL 2022
        GROUP BY QueryId, CompatibilityLevel
    )
    UPDATE cp
    SET HasRegressed = 1
    FROM dbo.CompatibilityPerformance cp
    JOIN BaselinePerf bp
    ON cp.QueryId = bp.QueryId
    WHERE cp.CompatibilityLevel = 160  -- SQL 2022
    AND ((cp.AvgDuration - bp.BaselineDuration) * 100.0) / 
        bp.BaselineDuration > @RegressionThreshold;
END;
```

2. Automated Resolution
```sql
CREATE PROCEDURE dbo.AutomateMitigationStrategy
    @QueryId bigint,
    @ActionType varchar(20)  -- 'ForceOldPlan', 'AdjustStats', 'Rewrite'
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @ActionType = 'ForceOldPlan'
    BEGIN
        -- Find and force the best performing legacy plan
        DECLARE @PlanId bigint;
        
        SELECT TOP 1 @PlanId = p.plan_id
        FROM sys.query_store_plan p
        JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
        WHERE p.query_id = @QueryId
        AND p.compatibility_level < 160
        ORDER BY rs.avg_duration ASC;
        
        IF @PlanId IS NOT NULL
        BEGIN
            EXEC sp_query_store_force_plan 
                @query_id = @QueryId,
                @plan_id = @PlanId;
        END;
    END
    ELSE IF @ActionType = 'AdjustStats'
    BEGIN
        -- Update statistics with increased sampling
        DECLARE @SQL nvarchar(max);
        SELECT @SQL = 'UPDATE STATISTICS ' + 
                     OBJECT_SCHEMA_NAME(q.object_id) + '.' +
                     OBJECT_NAME(q.object_id) + 
                     ' WITH FULLSCAN'
        FROM sys.query_store_query q
        WHERE query_id = @QueryId;
        
        IF @SQL IS NOT NULL
            EXEC sp_executesql @SQL;
    END;
END;
```

## Resource Utilization Tracking

### Memory Pressure Analysis
```sql
CREATE TABLE dbo.MemoryPressureLog
(
    LogId bigint IDENTITY(1,1) PRIMARY KEY,
    CollectionTime datetime2,
    CompatibilityLevel int,
    TotalServerMemory_MB int,
    TargetServerMemory_MB int,
    PlanCacheSize_MB int,
    BufferCacheSize_MB int,
    PageLifeExpectancy int,
    MemoryGrantsPending int
);

-- Monitor memory metrics
CREATE PROCEDURE dbo.TrackMemoryUtilization
AS
BEGIN
    INSERT INTO dbo.MemoryPressureLog
    SELECT 
        GETUTCDATE(),
        d.compatibility_level,
        (SELECT cntr_value 
         FROM sys.dm_os_performance_counters
         WHERE counter_name = 'Total Server Memory (KB)') / 1024,
        (SELECT cntr_value 
         FROM sys.dm_os_performance_counters
         WHERE counter_name = 'Target Server Memory (KB)') / 1024,
        (SELECT SUM(size_in_bytes) / 1048576.0
         FROM sys.dm_exec_cached_plans) as PlanCache_MB,
        (SELECT SUM(pages_kb) / 1024.0
         FROM sys.dm_os_memory_clerks
         WHERE type = 'MEMORYCLERK_SQLBUFFERPOOL') as BufferCache_MB,
        (SELECT cntr_value 
         FROM sys.dm_os_performance_counters
         WHERE counter_name = 'Page life expectancy'),
        (SELECT cntr_value 
         FROM sys.dm_os_performance_counters
         WHERE counter_name = 'Memory Grants Pending')
    FROM sys.databases d
    WHERE d.database_id = DB_ID();
END;
```

### CPU Utilization Analysis
```sql
CREATE TABLE dbo.CPUPressureLog
(
    LogId bigint IDENTITY(1,1) PRIMARY KEY,
    CollectionTime datetime2,
    CompatibilityLevel int,
    SQLProcessCPU int,
    SystemIdleCPU int,
    OtherProcessCPU int,
    CompileTime_ms bigint,
    ExecutionTime_ms bigint,
    QueryCount int
);

-- Monitor CPU metrics
CREATE PROCEDURE dbo.TrackCPUUtilization
AS
BEGIN
    INSERT INTO dbo.CPUPressureLog
    SELECT 
        GETUTCDATE(),
        d.compatibility_level,
        (SELECT ProcessUtilization
         FROM (
             SELECT TOP(1)
                 SystemIdle,
                 100 - SystemIdle - ProcessUtilization
                 AS OtherProcessUtilization,
                 ProcessUtilization
             FROM (
                 SELECT
                     record.value('(./Record/@id)[1]', 'int') AS record_id,
                     record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
                     AS SystemIdle,
                     record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
                     AS ProcessUtilization
                 FROM (
                     SELECT TOP(1) CONVERT(xml, record) AS record
                     FROM sys.dm_os_ring_buffers
                     WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                     ORDER BY timestamp DESC
                 ) AS rb
             ) AS y
         ) AS z),
        (SELECT SystemIdle FROM (
             SELECT TOP(1) SystemIdle
             FROM (
                 SELECT
                     record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
                     AS SystemIdle
                 FROM (
                     SELECT TOP(1) CONVERT(xml, record) AS record
                     FROM sys.dm_os_ring_buffers
                     WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                     ORDER BY timestamp DESC
                 ) AS rb
             ) AS y
         ) AS z),
        100 - ProcessUtilization - SystemIdle,
        SUM(total_compile_duration),
        SUM(total_execution_time),
        COUNT(*)
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY (
        SELECT TOP(1)
            SystemIdle,
            ProcessUtilization
        FROM (
            SELECT
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
                AS SystemIdle,
                record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
                AS ProcessUtilization
            FROM (
                SELECT TOP(1) CONVERT(xml, record) AS record
                FROM sys.dm_os_ring_buffers
                WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                ORDER BY timestamp DESC
            ) AS rb
        ) AS y
    ) AS cpu
    CROSS JOIN sys.databases d
    WHERE d.database_id = DB_ID()
    GROUP BY 
        d.compatibility_level,
        ProcessUtilization,
        SystemIdle;
END;
```

## Alert Configuration

### Performance Thresholds
```sql
CREATE TABLE dbo.PerformanceAlerts
(
    AlertId int IDENTITY(1,1) PRIMARY KEY,
    MetricName nvarchar(100),
    ThresholdValue decimal(18,2),
    Operator char(2),  -- GT, LT, EQ, GE, LE
    Severity tinyint,
    IsEnabled bit,
    LastTriggered datetime2,
    AlertCount int
);

-- Configure default thresholds
INSERT INTO dbo.PerformanceAlerts
(MetricName, ThresholdValue, Operator, Severity, IsEnabled)
VALUES
('CPU_Pressure', 90.0, 'GT', 1, 1),
('Memory_Grants_Pending', 5, 'GT', 1, 1),
('Page_Life_Expectancy', 300, 'LT', 2, 1),
('Query_Regression_Pct', 20.0, 'GT', 1, 1);

-- Create alert monitoring procedure
CREATE PROCEDURE dbo.CheckPerformanceAlerts
AS
BEGIN
    -- CPU alerts
    INSERT INTO dbo.AlertLog
    SELECT 
        'CPU Pressure',
        'CPU utilization exceeded threshold',
        GETUTCDATE(),
        1
    FROM dbo.CPUPressureLog
    WHERE SQLProcessCPU > (
        SELECT ThresholdValue 
        FROM dbo.PerformanceAlerts
        WHERE MetricName = 'CPU_Pressure'
    )
    AND CollectionTime > DATEADD(MINUTE, -5, GETUTCDATE());
    
    -- Memory alerts
    INSERT INTO dbo.AlertLog
    SELECT 
        'Memory Pressure',
        'Memory grants pending exceeded threshold',
        GETUTCDATE(),
        1
    FROM dbo.MemoryPressureLog
    WHERE MemoryGrantsPending > (
        SELECT ThresholdValue 
        FROM dbo.PerformanceAlerts
        WHERE MetricName = 'Memory_Grants_Pending'
    )
    AND CollectionTime > DATEADD(MINUTE, -5, GETUTCDATE());
    
    -- Update alert counts
    UPDATE dbo.PerformanceAlerts
    SET AlertCount = AlertCount + 1,
        LastTriggered = GETUTCDATE()
    WHERE AlertId IN (
        SELECT DISTINCT AlertId
        FROM dbo.AlertLog
        WHERE LogTime > DATEADD(MINUTE, -5, GETUTCDATE())
    );
END;
```

This comprehensive monitoring framework will help track and analyze the performance impact of running in different compatibility modes, enabling quick identification and resolution of issues during the migration process.
