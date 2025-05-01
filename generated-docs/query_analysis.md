# SQL Server Query Plan Analysis Framework

## Query Store Analysis

### Query Performance Regression Detection
```sql
CREATE TABLE dbo.QueryRegressionHistory
(
    RegressionId bigint IDENTITY(1,1) PRIMARY KEY,
    QueryId bigint,
    PlanId bigint,
    QueryText nvarchar(max),
    PreviousPlanXML xml,
    CurrentPlanXML xml,
    PreviousAvgDuration decimal(18,2),
    CurrentAvgDuration decimal(18,2),
    PreviousAvgCPU decimal(18,2),
    CurrentAvgCPU decimal(18,2),
    PreviousAvgLogicalReads bigint,
    CurrentAvgLogicalReads bigint,
    RegressionType varchar(20),
    DetectionTime datetime2,
    ResolutionTime datetime2,
    Resolution varchar(100)
);

CREATE PROCEDURE dbo.DetectQueryRegressions
    @PerformanceThreshold decimal(5,2) = 20.0,  -- 20% degradation
    @MinimumExecutions int = 10
AS
BEGIN
    -- Detect performance regressions
    WITH QueryMetrics AS (
        SELECT 
            q.query_id,
            qt.query_sql_text,
            p.plan_id,
            p.query_plan as current_plan_xml,
            rs.avg_duration,
            rs.avg_cpu_time,
            rs.avg_logical_io_reads,
            rs.count_executions,
            LAG(p.query_plan) OVER (
                PARTITION BY q.query_id 
                ORDER BY p.last_execution_time
            ) as previous_plan_xml,
            LAG(rs.avg_duration) OVER (
                PARTITION BY q.query_id 
                ORDER BY p.last_execution_time
            ) as previous_duration,
            LAG(rs.avg_cpu_time) OVER (
                PARTITION BY q.query_id 
                ORDER BY p.last_execution_time
            ) as previous_cpu,
            LAG(rs.avg_logical_io_reads) OVER (
                PARTITION BY q.query_id 
                ORDER BY p.last_execution_time
            ) as previous_reads
        FROM sys.query_store_query q
        JOIN sys.query_store_query_text qt 
            ON q.query_text_id = qt.query_text_id
        JOIN sys.query_store_plan p 
            ON q.query_id = p.query_id
        JOIN sys.query_store_runtime_stats rs 
            ON p.plan_id = rs.plan_id
        WHERE rs.count_executions >= @MinimumExecutions
    )
    INSERT INTO dbo.QueryRegressionHistory
    (
        QueryId, PlanId, QueryText, 
        PreviousPlanXML, CurrentPlanXML,
        PreviousAvgDuration, CurrentAvgDuration,
        PreviousAvgCPU, CurrentAvgCPU,
        PreviousAvgLogicalReads, CurrentAvgLogicalReads,
        RegressionType, DetectionTime
    )
    SELECT 
        qm.query_id,
        qm.plan_id,
        qm.query_sql_text,
        qm.previous_plan_xml,
        qm.current_plan_xml,
        qm.previous_duration,
        qm.avg_duration,
        qm.previous_cpu,
        qm.avg_cpu_time,
        qm.previous_reads,
        qm.avg_logical_io_reads,
        CASE 
            WHEN (qm.avg_duration - qm.previous_duration) * 100.0 / 
                 NULLIF(qm.previous_duration, 0) > @PerformanceThreshold
            THEN 'Duration'
            WHEN (qm.avg_cpu_time - qm.previous_cpu) * 100.0 / 
                 NULLIF(qm.previous_cpu, 0) > @PerformanceThreshold
            THEN 'CPU'
            WHEN (qm.avg_logical_io_reads - qm.previous_reads) * 100.0 / 
                 NULLIF(qm.previous_reads, 0) > @PerformanceThreshold
            THEN 'IO'
        END,
        GETUTCDATE()
    FROM QueryMetrics qm
    WHERE 
        (qm.avg_duration - qm.previous_duration) * 100.0 / 
            NULLIF(qm.previous_duration, 0) > @PerformanceThreshold
        OR (qm.avg_cpu_time - qm.previous_cpu) * 100.0 / 
            NULLIF(qm.previous_cpu, 0) > @PerformanceThreshold
        OR (qm.avg_logical_io_reads - qm.previous_reads) * 100.0 / 
            NULLIF(qm.previous_reads, 0) > @PerformanceThreshold;

    -- Analyze regressions
    SELECT 
        QueryText,
        RegressionType,
        (CurrentAvgDuration - PreviousAvgDuration) * 100.0 / 
            NULLIF(PreviousAvgDuration, 0) as DurationDegradation,
        (CurrentAvgCPU - PreviousAvgCPU) * 100.0 / 
            NULLIF(PreviousAvgCPU, 0) as CPUDegradation,
        (CurrentAvgLogicalReads - PreviousAvgLogicalReads) * 100.0 / 
            NULLIF(PreviousAvgLogicalReads, 0) as ReadsDegradation,
        DetectionTime,
        CASE 
            WHEN CurrentAvgDuration > PreviousAvgDuration * 2 
            THEN 'Critical'
            WHEN CurrentAvgCPU > PreviousAvgCPU * 2 
            THEN 'Critical'
            ELSE 'Warning'
        END as Severity,
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM sys.dm_exec_query_stats qs
                CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
                WHERE qp.query_plan.exist(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    //p:MissingIndex'
                ) = 1
            ) THEN 'Missing Index Detected'
            WHEN EXISTS (
                SELECT 1 
                FROM sys.dm_exec_query_stats qs
                CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
                WHERE qp.query_plan.exist(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    //p:NestedLoops[@Parallel="1"]'
                ) = 1
            ) THEN 'Parallelism Issue'
            ELSE 'Plan Changed'
        END as RegressionCause
    FROM dbo.QueryRegressionHistory
    WHERE ResolutionTime IS NULL
    ORDER BY DetectionTime DESC;
END;
```

### Plan Cache Analysis
```sql
CREATE PROCEDURE dbo.AnalyzePlanCache
    @MinExecutionCount int = 100
AS
BEGIN
    -- Analyze plan cache patterns
    SELECT TOP 50
        qt.text as QueryText,
        qs.execution_count,
        qs.total_worker_time / 1000000.0 as TotalCPUSeconds,
        qs.total_elapsed_time / 1000000.0 as TotalDurationSeconds,
        qs.total_logical_reads / qs.execution_count as AvgLogicalReads,
        qs.total_physical_reads / qs.execution_count as AvgPhysicalReads,
        qs.total_logical_writes / qs.execution_count as AvgLogicalWrites,
        CAST(qp.query_plan as xml) as QueryPlan,
        CASE 
            WHEN qp.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:MissingIndex'
            ) = 1 THEN 'Missing Index'
            WHEN qp.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:NestedLoops[@Parallel="1"]'
            ) = 1 THEN 'Parallel Nested Loops'
            WHEN qp.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:Warnings'
            ) = 1 THEN 'Plan Warnings'
            ELSE 'Normal'
        END as PlanStatus,
        CASE 
            WHEN qs.total_worker_time / qs.execution_count > 1000000 -- 1 second
            THEN 'High CPU'
            WHEN qs.total_logical_reads / qs.execution_count > 1000
            THEN 'High IO'
            ELSE 'Normal'
        END as PerformanceStatus
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE qs.execution_count >= @MinExecutionCount
    ORDER BY qs.total_worker_time DESC;

    -- Analyze plan reuse
    SELECT TOP 20
        qt.text as QueryText,
        qs.execution_count,
        qs.cached_time,
        qs.last_execution_time,
        qp.query_plan,
        DATEDIFF(MINUTE, qs.cached_time, GETDATE()) as CacheMinutes,
        qs.total_worker_time / qs.execution_count as AvgCPUMicroseconds,
        CASE 
            WHEN qs.execution_count = 1 
            THEN 'Single Use'
            WHEN qs.execution_count < 10 
            THEN 'Low Reuse'
            ELSE 'High Reuse'
        END as ReusePattern
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    ORDER BY qs.execution_count ASC;
END;
```

### Parameter Sniffing Detection
```sql
CREATE PROCEDURE dbo.DetectParameterSniffing
AS
BEGIN
    -- Analyze parameter sensitivity
    WITH PlanMetrics AS (
        SELECT 
            qt.text as QueryText,
            p.query_id,
            p.plan_id,
            rs.avg_duration,
            rs.avg_cpu_time,
            rs.avg_logical_io_reads,
            rs.count_executions,
            STDEV(rs.avg_duration) OVER (
                PARTITION BY p.query_id
            ) as duration_stdev,
            STDEV(rs.avg_cpu_time) OVER (
                PARTITION BY p.query_id
            ) as cpu_stdev,
            STDEV(rs.avg_logical_io_reads) OVER (
                PARTITION BY p.query_id
            ) as reads_stdev
        FROM sys.query_store_plan p
        JOIN sys.query_store_query q 
            ON p.query_id = q.query_id
        JOIN sys.query_store_query_text qt 
            ON q.query_text_id = qt.query_text_id
        JOIN sys.query_store_runtime_stats rs 
            ON p.plan_id = rs.plan_id
    )
    SELECT 
        QueryText,
        COUNT(DISTINCT plan_id) as PlanCount,
        AVG(avg_duration) as AvgDuration,
        MAX(avg_duration) as MaxDuration,
        MIN(avg_duration) as MinDuration,
        duration_stdev as DurationStdDev,
        cpu_stdev as CPUStdDev,
        reads_stdev as ReadsStdDev,
        CASE 
            WHEN duration_stdev > AVG(avg_duration) 
            THEN 'High Duration Variance'
            WHEN cpu_stdev > AVG(avg_cpu_time) 
            THEN 'High CPU Variance'
            WHEN reads_stdev > AVG(avg_logical_io_reads) 
            THEN 'High IO Variance'
            ELSE 'Normal'
        END as VarianceStatus,
        CASE 
            WHEN COUNT(DISTINCT plan_id) > 5 
            THEN 'Multiple Plans'
            WHEN duration_stdev > AVG(avg_duration) 
            THEN 'Parameter Sensitive'
            ELSE 'Stable'
        END as PlanStability
    FROM PlanMetrics
    GROUP BY 
        QueryText,
        duration_stdev,
        cpu_stdev,
        reads_stdev
    HAVING COUNT(DISTINCT plan_id) > 1
    ORDER BY duration_stdev DESC;
END;
```

This query analysis framework provides comprehensive tools for:
1. Query performance regression detection using Query Store
2. Plan cache analysis for resource utilization patterns
3. Parameter sniffing detection and plan stability analysis
4. Automated recommendations for query optimization

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
