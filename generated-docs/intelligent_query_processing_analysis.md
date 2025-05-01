# SQL Server Intelligent Query Processing Analysis Framework

## Adaptive Query Processing Monitoring

### Batch Mode Analysis
```sql
CREATE TABLE dbo.BatchModeMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    QueryID bigint,
    DatabaseName sysname,
    ObjectName sysname,
    BatchModeName nvarchar(60),
    ExecutionCount int,
    BatchRowCount bigint,
    MemoryGrantKB bigint,
    DegreeOfParallelism int,
    AdaptiveThreshold int,
    AdaptiveFeedback xml,
    LastExecutionTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorBatchModeProcessing
    @HighMemoryThresholdMB decimal(18,2) = 1024.0,
    @LowBatchSizeThreshold int = 1000
AS
BEGIN
    -- Capture batch mode metrics
    INSERT INTO dbo.BatchModeMetrics
    SELECT 
        qs.query_id,
        DB_NAME(qt.dbid) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qp.query_plan.value(
            'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            (/p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/p:StmtSimple/
             p:QueryPlan/p:RelOp/@BatchMode)[1]',
            'nvarchar(60)'
        ) as BatchModeName,
        qs.execution_count,
        qp.query_plan.value(
            'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            sum(/p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/p:StmtSimple/
                p:QueryPlan/p:RelOp/@EstimateRows)',
            'bigint'
        ) as BatchRowCount,
        qp.query_plan.value(
            'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            (/p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/p:StmtSimple/
             p:QueryPlan/@MemoryGrant)[1]',
            'bigint'
        ) as MemoryGrantKB,
        qs.max_dop as DegreeOfParallelism,
        qp.query_plan.value(
            'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            (/p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/p:StmtSimple/
             p:QueryPlan/@AdaptiveThreshold)[1]',
            'int'
        ) as AdaptiveThreshold,
        qp.query_plan.query(
            'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            /p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/p:StmtSimple/
            p:QueryPlan/p:AdaptiveFeedback'
        ) as AdaptiveFeedback,
        qs.last_execution_time,
        GETUTCDATE()
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE qp.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:RelOp[@BatchMode]'
    ) = 1;

    -- Analyze batch mode patterns
    WITH BatchMetrics AS (
        SELECT 
            QueryID,
            DatabaseName,
            ObjectName,
            BatchModeName,
            ExecutionCount,
            BatchRowCount,
            MemoryGrantKB / 1024.0 as MemoryGrantMB,
            DegreeOfParallelism,
            AdaptiveThreshold,
            AdaptiveFeedback,
            LastExecutionTime,
            LAG(BatchRowCount) OVER (
                PARTITION BY QueryID 
                ORDER BY CollectionTime
            ) as PreviousBatchRows
        FROM dbo.BatchModeMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        QueryID,
        DatabaseName,
        ObjectName,
        BatchModeName,
        ExecutionCount,
        BatchRowCount,
        MemoryGrantMB,
        DegreeOfParallelism,
        AdaptiveThreshold,
        AdaptiveFeedback,
        CASE 
            WHEN MemoryGrantMB > @HighMemoryThresholdMB 
                 AND BatchRowCount < @LowBatchSizeThreshold 
            THEN 'Inefficient Memory Usage'
            WHEN BatchRowCount < @LowBatchSizeThreshold 
            THEN 'Small Batch Size'
            WHEN DegreeOfParallelism > 4 
                 AND BatchRowCount < 10000 
            THEN 'Excessive Parallelism'
            ELSE 'Normal'
        END as BatchStatus,
        CASE 
            WHEN MemoryGrantMB > @HighMemoryThresholdMB 
                 AND BatchRowCount < @LowBatchSizeThreshold 
            THEN 'Review memory grant feedback'
            WHEN BatchRowCount < @LowBatchSizeThreshold 
            THEN 'Consider row mode processing'
            WHEN DegreeOfParallelism > 4 
                 AND BatchRowCount < 10000 
            THEN 'Adjust MAXDOP settings'
            ELSE 'No action needed'
        END as Recommendation
    FROM BatchMetrics
    WHERE MemoryGrantMB > @HighMemoryThresholdMB
    OR BatchRowCount < @LowBatchSizeThreshold
    OR (DegreeOfParallelism > 4 AND BatchRowCount < 10000)
    ORDER BY 
        CASE 
            WHEN MemoryGrantMB > @HighMemoryThresholdMB 
                 AND BatchRowCount < @LowBatchSizeThreshold THEN 1
            WHEN BatchRowCount < @LowBatchSizeThreshold THEN 2
            ELSE 3
        END,
        MemoryGrantMB DESC;
END;
```

### Memory Grant Feedback Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeMemoryGrantFeedback
AS
BEGIN
    -- Analyze memory grant feedback patterns
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        MIN(rs.min_duration) as MinDurationMs,
        MAX(rs.max_duration) as MaxDurationMs,
        AVG(rs.avg_query_max_used_memory) as AvgMemoryGrantKB,
        MAX(rs.max_query_max_used_memory) as MaxMemoryGrantKB,
        MIN(rs.min_query_max_used_memory) as MinMemoryGrantKB,
        COUNT(DISTINCT rs.memory_grant_feedback_loop_id) 
            as FeedbackLoopCount,
        STRING_AGG(
            CAST(rs.memory_grant_feedback_loop_id as varchar) + 
            ':' + 
            CAST(rs.avg_query_max_used_memory as varchar),
            ', '
        ) as GrantHistory,
        CASE 
            WHEN MAX(rs.max_query_max_used_memory) > 
                 MIN(rs.min_query_max_used_memory) * 2 
            THEN 'High Grant Variance'
            WHEN COUNT(
                DISTINCT rs.memory_grant_feedback_loop_id
            ) > 5 
            THEN 'Unstable Grants'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Performance Variance'
            ELSE 'Normal'
        END as FeedbackStatus,
        CASE 
            WHEN MAX(rs.max_query_max_used_memory) > 
                 MIN(rs.min_query_max_used_memory) * 2 
            THEN 'Review memory usage patterns'
            WHEN COUNT(
                DISTINCT rs.memory_grant_feedback_loop_id
            ) > 5 
            THEN 'Stabilize memory grants'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Investigate performance variance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE rs.last_execution_time >= 
          DATEADD(DAY, -7, GETUTCDATE())
    AND p.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:MemoryGrantFeedback'
    ) = 1
    GROUP BY 
        q.query_id,
        qt.query_sql_text,
        p.query_plan
    HAVING MAX(rs.max_query_max_used_memory) > 
           MIN(rs.min_query_max_used_memory) * 2
    OR COUNT(DISTINCT rs.memory_grant_feedback_loop_id) > 5
    OR MAX(rs.max_duration) > MIN(rs.min_duration) * 2
    ORDER BY 
        CASE 
            WHEN MAX(rs.max_query_max_used_memory) > 
                 MIN(rs.min_query_max_used_memory) * 2 THEN 1
            WHEN COUNT(
                DISTINCT rs.memory_grant_feedback_loop_id
            ) > 5 THEN 2
            ELSE 3
        END,
        FeedbackLoopCount DESC;
END;
```

### Interleaved Execution Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeInterleavedExecution
AS
BEGIN
    -- Analyze interleaved execution patterns
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        AVG(rs.avg_cpu_time) as AvgCPUTimeMs,
        AVG(rs.avg_logical_io_reads) as AvgLogicalReads,
        AVG(rs.avg_query_max_used_memory) as AvgMemoryKB,
        COUNT(
            DISTINCT CAST(p.query_plan as nvarchar(max))
        ) as PlanVersionCount,
        STRING_AGG(
            CAST(rs.runtime_stats_id as varchar) + 
            ':' + 
            CAST(rs.avg_duration as varchar),
            ', '
        ) as ExecutionHistory,
        CASE 
            WHEN COUNT(
                DISTINCT CAST(p.query_plan as nvarchar(max))
            ) > 3 
            THEN 'Multiple Plan Versions'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Variable Performance'
            WHEN AVG(rs.avg_query_max_used_memory) > 102400 
            THEN 'High Memory Usage'
            ELSE 'Normal'
        END as ExecutionStatus,
        CASE 
            WHEN COUNT(
                DISTINCT CAST(p.query_plan as nvarchar(max))
            ) > 3 
            THEN 'Review cardinality estimates'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Analyze execution variance'
            WHEN AVG(rs.avg_query_max_used_memory) > 102400 
            THEN 'Optimize memory usage'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE rs.last_execution_time >= 
          DATEADD(DAY, -7, GETUTCDATE())
    AND p.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:InterleavedExecution'
    ) = 1
    GROUP BY 
        q.query_id,
        qt.query_sql_text,
        p.query_plan
    HAVING COUNT(
        DISTINCT CAST(p.query_plan as nvarchar(max))
    ) > 3
    OR MAX(rs.max_duration) > MIN(rs.min_duration) * 2
    OR AVG(rs.avg_query_max_used_memory) > 102400
    ORDER BY 
        CASE 
            WHEN COUNT(
                DISTINCT CAST(p.query_plan as nvarchar(max))
            ) > 3 THEN 1
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 THEN 2
            ELSE 3
        END,
        PlanVersionCount DESC;
END;
```

This Intelligent Query Processing analysis framework provides comprehensive tools for:
1. Monitoring batch mode processing effectiveness
2. Analyzing memory grant feedback patterns
3. Tracking interleaved execution performance
4. Optimizing adaptive query processing

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
