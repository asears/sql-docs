# SQL Server CPU Analysis Framework

## CPU Usage Monitoring

### Scheduler Analysis
```sql
CREATE TABLE dbo.SchedulerHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    SchedulerId int,
    ParentNodeId int,
    CurrentTasks int,
    RunnableTasks int,
    ActiveWorkers int,
    LoadFactor decimal(5,2),
    YieldCount bigint,
    PreemptiveSwitches int,
    ContextSwitches int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorSchedulers
    @HighLoadThreshold decimal(5,2) = 75.0,
    @CriticalLoadThreshold decimal(5,2) = 90.0
AS
BEGIN
    -- Capture scheduler metrics
    INSERT INTO dbo.SchedulerHistory
    SELECT 
        scheduler_id,
        parent_node_id,
        current_tasks_count,
        runnable_tasks_count,
        active_workers_count,
        load_factor,
        yield_count,
        preemptive_switches_count,
        context_switches_count,
        GETUTCDATE()
    FROM sys.dm_os_schedulers
    WHERE scheduler_id < 255;  -- Exclude internal schedulers

    -- Analyze scheduler load patterns
    WITH SchedulerMetrics AS (
        SELECT 
            ParentNodeId,
            AVG(LoadFactor) as AvgLoad,
            MAX(LoadFactor) as PeakLoad,
            SUM(CurrentTasks) as TotalTasks,
            SUM(RunnableTasks) as RunnableTasks,
            AVG(ContextSwitches) as AvgContextSwitches
        FROM dbo.SchedulerHistory
        WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        GROUP BY ParentNodeId
    )
    SELECT 
        ParentNodeId as NUMANode,
        AvgLoad,
        PeakLoad,
        TotalTasks,
        RunnableTasks,
        AvgContextSwitches,
        CASE 
            WHEN AvgLoad >= @CriticalLoadThreshold THEN 'Critical'
            WHEN AvgLoad >= @HighLoadThreshold THEN 'Warning'
            ELSE 'Normal'
        END as LoadStatus,
        CASE 
            WHEN AvgLoad >= @CriticalLoadThreshold 
            THEN 'Consider:
                  1. MAXDOP reduction
                  2. Resource Governor limits
                  3. Query optimization'
            WHEN AvgLoad >= @HighLoadThreshold 
            THEN 'Monitor for further increase'
            ELSE 'No action needed'
        END as Recommendation
    FROM SchedulerMetrics
    ORDER BY AvgLoad DESC;
END;
```

### Query CPU Analysis
```sql
CREATE TABLE dbo.QueryCPUHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    QueryHash binary(8),
    PlanHash binary(8),
    QueryText nvarchar(max),
    ExecutionCount bigint,
    TotalCPUMs bigint,
    AvgCPUMs decimal(18,2),
    MaxCPUMs bigint,
    TotalDurationMs bigint,
    CPUToElapsedRatio decimal(5,2),
    ParallelismDegree int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeQueryCPU
    @MinExecutions int = 10,
    @HighCPUThresholdMs int = 1000
AS
BEGIN
    -- Capture query CPU metrics
    INSERT INTO dbo.QueryCPUHistory
    SELECT 
        qs.query_hash,
        qs.plan_handle,
        st.text,
        qs.execution_count,
        qs.total_worker_time,
        qs.total_worker_time * 1.0 / qs.execution_count,
        qs.max_worker_time,
        qs.total_elapsed_time,
        CAST(qs.total_worker_time * 100.0 / 
             NULLIF(qs.total_elapsed_time, 0) as decimal(5,2)),
        qp.DegreeOfParallelism,
        GETUTCDATE()
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
    CROSS APPLY (
        SELECT MAX(p.DegreeOfParallelism) as DegreeOfParallelism
        FROM (
            SELECT CAST(qp.query_plan as xml) as PlanXML
            FROM sys.dm_exec_query_plan(qs.plan_handle) qp
        ) as qp
        CROSS APPLY qp.PlanXML.nodes('//RelOp') rel(p)
        CROSS APPLY (
            SELECT 
                p.value('@DegreeOfParallelism', 'int') 
                as DegreeOfParallelism
        ) as p
    ) as qp
    WHERE qs.execution_count >= @MinExecutions;

    -- Analyze CPU patterns
    WITH CPUMetrics AS (
        SELECT 
            QueryHash,
            QueryText,
            ExecutionCount,
            AvgCPUMs,
            MaxCPUMs,
            CPUToElapsedRatio,
            ParallelismDegree,
            LAG(AvgCPUMs) OVER (
                PARTITION BY QueryHash 
                ORDER BY CollectionTime
            ) as PreviousAvgCPU
        FROM dbo.QueryCPUHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        QueryText,
        ExecutionCount,
        AvgCPUMs / 1000.0 as AvgCPUSeconds,
        MaxCPUMs / 1000.0 as MaxCPUSeconds,
        CPUToElapsedRatio,
        ParallelismDegree,
        CASE 
            WHEN AvgCPUMs > @HighCPUThresholdMs THEN 'High CPU'
            WHEN CPUToElapsedRatio > 80 THEN 'CPU Bound'
            ELSE 'Normal'
        END as CPUStatus,
        CASE 
            WHEN AvgCPUMs > @HighCPUThresholdMs 
            THEN 'Consider:
                  1. Index optimization
                  2. Query rewrite
                  3. MAXDOP adjustment'
            WHEN CPUToElapsedRatio > 80 
            THEN 'Review for CPU optimization'
            ELSE 'No action needed'
        END as Recommendation
    FROM CPUMetrics
    WHERE AvgCPUMs > @HighCPUThresholdMs
    OR CPUToElapsedRatio > 80
    ORDER BY AvgCPUMs DESC;
END;
```

### Parallel Query Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeParallelQueries
AS
BEGIN
    -- Analyze currently running parallel queries
    SELECT 
        r.session_id,
        r.command,
        r.status,
        r.cpu_time,
        r.total_elapsed_time,
        r.parallel_worker_count,
        st.text as QueryText,
        qp.query_plan,
        CASE 
            WHEN r.parallel_worker_count > 8 
            THEN 'High Parallelism'
            WHEN r.parallel_worker_count > 4 
            THEN 'Moderate Parallelism'
            ELSE 'Normal'
        END as ParallelismStatus,
        CASE 
            WHEN r.parallel_worker_count > 8 
            THEN 'Consider MAXDOP reduction'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_requests r
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
    CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) qp
    WHERE r.parallel_worker_count > 0
    ORDER BY r.parallel_worker_count DESC;

    -- Analyze parallel wait stats
    SELECT 
        wait_type,
        waiting_tasks_count,
        wait_time_ms,
        max_wait_time_ms,
        signal_wait_time_ms,
        CASE 
            WHEN wait_time_ms > 10000 THEN 'Critical'
            WHEN wait_time_ms > 1000 THEN 'Warning'
            ELSE 'Normal'
        END as WaitStatus
    FROM sys.dm_os_wait_stats
    WHERE wait_type LIKE 'CXPACKET%'
    OR wait_type LIKE 'EXCHANGE%'
    ORDER BY wait_time_ms DESC;
END;
```

### CPU Signal Wait Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeCPUSignalWaits
AS
BEGIN
    -- Analyze signal wait times
    WITH SignalWaits AS (
        SELECT 
            wait_type,
            signal_wait_time_ms,
            wait_time_ms,
            100.0 * signal_wait_time_ms / wait_time_ms 
                as signal_wait_percent,
            ROW_NUMBER() OVER (
                ORDER BY signal_wait_time_ms DESC
            ) as rn
        FROM sys.dm_os_wait_stats
        WHERE wait_time_ms > 0
        AND wait_type NOT IN (
            'SLEEP_TASK',
            'BROKER_TASK_STOP',
            'BROKER_TO_FLUSH',
            'SQLTRACE_BUFFER_FLUSH',
            'CLR_AUTO_EVENT',
            'CLR_MANUAL_EVENT'
        )
    )
    SELECT 
        wait_type,
        signal_wait_time_ms,
        wait_time_ms,
        signal_wait_percent,
        CASE 
            WHEN signal_wait_percent > 20 THEN 'Critical'
            WHEN signal_wait_percent > 10 THEN 'Warning'
            ELSE 'Normal'
        END as SignalWaitStatus,
        CASE 
            WHEN signal_wait_percent > 20 
            THEN 'Consider:
                  1. CPU upgrade
                  2. Workload distribution
                  3. Query optimization'
            WHEN signal_wait_percent > 10 
            THEN 'Monitor for increase'
            ELSE 'No action needed'
        END as Recommendation
    FROM SignalWaits
    WHERE rn <= 10
    ORDER BY signal_wait_time_ms DESC;

    -- Calculate overall signal wait ratio
    SELECT 
        SUM(signal_wait_time_ms) * 1.0 / 
            SUM(wait_time_ms) * 100 as overall_signal_wait_percent,
        CASE 
            WHEN SUM(signal_wait_time_ms) * 1.0 / 
                 SUM(wait_time_ms) * 100 > 20 
            THEN 'Critical CPU Pressure'
            WHEN SUM(signal_wait_time_ms) * 1.0 / 
                 SUM(wait_time_ms) * 100 > 10 
            THEN 'Moderate CPU Pressure'
            ELSE 'Normal CPU Utilization'
        END as CPUPressureStatus
    FROM sys.dm_os_wait_stats
    WHERE wait_type NOT IN (
        'SLEEP_TASK',
        'BROKER_TASK_STOP',
        'BROKER_TO_FLUSH',
        'SQLTRACE_BUFFER_FLUSH',
        'CLR_AUTO_EVENT',
        'CLR_MANUAL_EVENT'
    );
END;
```

This CPU analysis framework provides detailed tools for:
1. Scheduler load monitoring and NUMA node analysis
2. Query CPU utilization tracking and optimization recommendations
3. Parallel query analysis with resource usage metrics
4. CPU signal wait analysis for detecting processor pressure

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
