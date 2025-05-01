# SQL Server Scalar UDF Analysis Framework

## UDF Performance Monitoring

### Inlining Analysis
```sql
CREATE TABLE dbo.UDFMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    FunctionName sysname,
    IsInlineable bit,
    InlineReason nvarchar(max),
    ExecutionCount bigint,
    AvgDurationMs decimal(18,2),
    CPUTimeMs decimal(18,2),
    LogicalReads bigint,
    InvokedByQueries int,
    LastExecutionTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorUDFPerformance
    @HighDurationThresholdMs decimal(18,2) = 100.0,
    @HighUsageThreshold int = 1000
AS
BEGIN
    -- Capture UDF metrics
    INSERT INTO dbo.UDFMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(o.object_id) as SchemaName,
        o.name as FunctionName,
        CASE 
            WHEN o.is_inlineable = 1 THEN 1
            ELSE 0
        END as IsInlineable,
        m.inline_type_desc as InlineReason,
        SUM(qs.execution_count) as ExecutionCount,
        AVG(qs.total_elapsed_time * 1.0 / 
            NULLIF(qs.execution_count, 0)) / 1000.0 as AvgDurationMs,
        SUM(qs.total_worker_time) / 1000.0 as CPUTimeMs,
        SUM(qs.total_logical_reads) as LogicalReads,
        COUNT(DISTINCT qt.text) as InvokedByQueries,
        MAX(qs.last_execution_time) as LastExecutionTime,
        GETUTCDATE()
    FROM sys.objects o
    JOIN sys.sql_modules m ON o.object_id = m.object_id
    LEFT JOIN sys.dm_exec_query_stats qs ON 
        OBJECT_NAME(o.object_id) = 
        OBJECT_NAME(
            OBJECT_ID(
                SUBSTRING(
                    qt.text,
                    (qs.statement_start_offset/2) + 1,
                    ((
                        CASE 
                            qs.statement_end_offset
                            WHEN -1 
                            THEN DATALENGTH(qt.text)
                            ELSE qs.statement_end_offset
                        END - qs.statement_start_offset
                    )/2) + 1
                )
            )
        )
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    WHERE o.type IN ('FN', 'FS', 'FT')
    GROUP BY 
        DB_NAME(),
        OBJECT_SCHEMA_NAME(o.object_id),
        o.name,
        o.is_inlineable,
        m.inline_type_desc;

    -- Analyze UDF patterns
    WITH UDFMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            FunctionName,
            IsInlineable,
            InlineReason,
            ExecutionCount,
            AvgDurationMs,
            CPUTimeMs,
            LogicalReads,
            InvokedByQueries,
            LastExecutionTime,
            LAG(AvgDurationMs) OVER (
                PARTITION BY DatabaseName, SchemaName, FunctionName 
                ORDER BY CollectionTime
            ) as PreviousAvgDuration
        FROM dbo.UDFMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -24, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        FunctionName,
        IsInlineable,
        InlineReason,
        ExecutionCount,
        AvgDurationMs,
        CPUTimeMs,
        LogicalReads,
        InvokedByQueries,
        CASE 
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold 
                 AND AvgDurationMs > @HighDurationThresholdMs 
            THEN 'Critical Performance'
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold 
            THEN 'High Usage Non-Inlineable'
            WHEN AvgDurationMs > @HighDurationThresholdMs 
            THEN 'High Duration'
            WHEN AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Performance Regression'
            ELSE 'Normal'
        END as UDFStatus,
        CASE 
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold 
                 AND AvgDurationMs > @HighDurationThresholdMs 
            THEN 'Refactor for inlining'
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold 
            THEN 'Review inlining blockers'
            WHEN AvgDurationMs > @HighDurationThresholdMs 
            THEN 'Optimize function logic'
            WHEN AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Investigate regression'
            ELSE 'No action needed'
        END as Recommendation
    FROM UDFMetrics
    WHERE NOT IsInlineable 
          AND ExecutionCount > @HighUsageThreshold
    OR AvgDurationMs > @HighDurationThresholdMs
    OR AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold 
                 AND AvgDurationMs > @HighDurationThresholdMs THEN 1
            WHEN NOT IsInlineable 
                 AND ExecutionCount > @HighUsageThreshold THEN 2
            ELSE 3
        END,
        ExecutionCount DESC;
END;
```

### Inlining Impact Analysis 
```sql
CREATE PROCEDURE dbo.AnalyzeInliningImpact
AS
BEGIN
    -- Analyze inlining performance impact
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        MIN(rs.min_duration) as MinDurationMs,
        MAX(rs.max_duration) as MaxDurationMs,
        COUNT(
            CASE 
                WHEN p.query_plan.exist(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    //p:ScalarOperator[@FunctionName]'
                ) = 1 THEN 1 
            END
        ) as UDFCount,
        STRING_AGG(
            CAST(
                p.query_plan.value(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    (/p:ShowPlanXML/p:BatchSequence/p:Batch/p:Statements/
                     p:StmtSimple/p:QueryPlan/p:RelOp//
                     p:ScalarOperator[@FunctionName]/@FunctionName)[1]',
                    'nvarchar(max)'
                ) as varchar(max)
            ),
            ', '
        ) as InlinedFunctions,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND MAX(rs.max_duration) > 
                     MIN(rs.min_duration) * 2 
            THEN 'High Usage Variance'
            WHEN COUNT(*) > 1000 
            THEN 'High Usage'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Performance Variance'
            ELSE 'Normal'
        END as InliningPattern,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND MAX(rs.max_duration) > 
                     MIN(rs.min_duration) * 2 
            THEN 'Review inlining effectiveness'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor performance'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Analyze performance variance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE p.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:ScalarOperator[@FunctionName]'
    ) = 1
    AND rs.last_execution_time >= 
        DATEADD(DAY, -7, GETUTCDATE())
    GROUP BY 
        q.query_id,
        qt.query_sql_text,
        p.query_plan
    HAVING COUNT(*) > 1000
    OR MAX(rs.max_duration) > MIN(rs.min_duration) * 2
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 1000 
                 AND MAX(rs.max_duration) > 
                     MIN(rs.min_duration) * 2 THEN 1
            WHEN COUNT(*) > 1000 THEN 2
            ELSE 3
        END,
        ExecutionCount DESC;
END;
```

### Inlining Optimization Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeInliningOptimizations
AS
BEGIN
    -- Analyze potential inlining optimizations
    SELECT 
        OBJECT_SCHEMA_NAME(o.object_id) as SchemaName,
        o.name as FunctionName,
        m.definition as FunctionDefinition,
        CASE 
            WHEN o.is_inlineable = 0 
            THEN m.inline_type_desc 
            ELSE 'Inlineable'
        END as InlineableStatus,
        (
            SELECT COUNT(*)
            FROM sys.sql_expression_dependencies d
            WHERE d.referenced_id = o.object_id
        ) as ReferencedByCount,
        (
            SELECT COUNT(DISTINCT p.query_id)
            FROM sys.query_store_query q
            JOIN sys.query_store_plan p 
                ON q.query_id = p.query_id
            WHERE p.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:ScalarOperator[@FunctionName[contains(., "' + 
                o.name + '")]]'
            ) = 1
        ) as QueryStoreReferences,
        CASE 
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%Data Access%' 
            THEN 'Contains Data Access'
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%CLR%' 
            THEN 'CLR Function'
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%Complex%' 
            THEN 'Complex Logic'
            WHEN o.is_inlineable = 0 
            THEN 'Other Blocker'
            ELSE 'Already Inlineable'
        END as OptimizationCategory,
        CASE 
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%Data Access%' 
            THEN 'Move data access to calling query'
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%CLR%' 
            THEN 'Consider T-SQL implementation'
            WHEN o.is_inlineable = 0 
                 AND m.inline_type_desc LIKE '%Complex%' 
            THEN 'Simplify function logic'
            WHEN o.is_inlineable = 0 
            THEN 'Review inlining requirements'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.objects o
    JOIN sys.sql_modules m ON o.object_id = m.object_id
    WHERE o.type IN ('FN', 'FS', 'FT')
    AND (
        o.is_inlineable = 0
        OR EXISTS (
            SELECT 1
            FROM sys.query_store_query q
            JOIN sys.query_store_plan p 
                ON q.query_id = p.query_id
            WHERE p.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:ScalarOperator[@FunctionName[contains(., "' + 
                o.name + '")]]'
            ) = 1
        )
    )
    ORDER BY 
        CASE o.is_inlineable
            WHEN 0 THEN 1
            ELSE 2
        END,
        ReferencedByCount DESC;
END;
```

This Scalar UDF Analysis framework provides comprehensive tools for:
1. Monitoring UDF performance and inlining eligibility
2. Analyzing the impact of function inlining on query performance
3. Identifying optimization opportunities for non-inlineable functions
4. Tracking UDF usage patterns and performance trends

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
