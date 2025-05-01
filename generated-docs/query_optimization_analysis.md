# SQL Server Query Optimization Analysis Framework

## Query Optimization Monitoring

### Plan Guide Analysis
```sql
CREATE TABLE dbo.PlanGuideMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    PlanGuideName sysname,
    DatabaseName sysname,
    SchemaName sysname,
    ObjectName sysname,
    GuideType nvarchar(60),
    IsDisabled bit,
    HintText nvarchar(max),
    UsageCount int,
    LastUsedTime datetime2,
    AvgDurationMs decimal(18,2),
    PlanMatchFailures int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorPlanGuides
    @UnusedThresholdDays int = 30,
    @HighFailureCount int = 10
AS
BEGIN
    -- Capture plan guide metrics
    INSERT INTO dbo.PlanGuideMetrics
    SELECT 
        pg.name as PlanGuideName,
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(pg.scope_object_id) as SchemaName,
        OBJECT_NAME(pg.scope_object_id) as ObjectName,
        pg.scope_type_desc as GuideType,
        pg.is_disabled,
        pg.hints as HintText,
        (
            SELECT COUNT(*)
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%' + pg.name + '%'
        ) as UsageCount,
        (
            SELECT MAX(last_execution_time)
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%' + pg.name + '%'
        ) as LastUsedTime,
        (
            SELECT AVG(total_elapsed_time * 1.0 / execution_count)
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%' + pg.name + '%'
        ) / 1000.0 as AvgDurationMs,
        (
            SELECT COUNT(*)
            FROM sys.dm_exec_invalid_plans ip
            WHERE ip.plan_guide_name = pg.name
        ) as PlanMatchFailures,
        GETUTCDATE()
    FROM sys.plan_guides pg;

    -- Analyze plan guide patterns
    WITH GuideMetrics AS (
        SELECT 
            PlanGuideName,
            DatabaseName,
            SchemaName,
            ObjectName,
            GuideType,
            IsDisabled,
            HintText,
            UsageCount,
            LastUsedTime,
            AvgDurationMs,
            PlanMatchFailures,
            DATEDIFF(
                DAY, 
                LastUsedTime, 
                GETUTCDATE()
            ) as DaysSinceLastUse
        FROM dbo.PlanGuideMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        PlanGuideName,
        DatabaseName,
        SchemaName,
        ObjectName,
        GuideType,
        IsDisabled,
        HintText,
        UsageCount,
        LastUsedTime,
        AvgDurationMs,
        PlanMatchFailures,
        DaysSinceLastUse,
        CASE 
            WHEN PlanMatchFailures > @HighFailureCount 
            THEN 'High Failures'
            WHEN IsDisabled = 1 
            THEN 'Disabled'
            WHEN DaysSinceLastUse > @UnusedThresholdDays 
            THEN 'Unused'
            WHEN UsageCount = 0 
            THEN 'Never Used'
            ELSE 'Active'
        END as GuideStatus,
        CASE 
            WHEN PlanMatchFailures > @HighFailureCount 
            THEN 'Review guide definition'
            WHEN IsDisabled = 1 
            THEN 'Evaluate need for guide'
            WHEN DaysSinceLastUse > @UnusedThresholdDays 
            THEN 'Consider removing guide'
            WHEN UsageCount = 0 
            THEN 'Verify guide necessity'
            ELSE 'Monitor performance'
        END as Recommendation
    FROM GuideMetrics
    WHERE PlanMatchFailures > @HighFailureCount
    OR IsDisabled = 1
    OR DaysSinceLastUse > @UnusedThresholdDays
    OR UsageCount = 0
    ORDER BY 
        CASE 
            WHEN PlanMatchFailures > @HighFailureCount THEN 1
            WHEN IsDisabled = 1 THEN 2
            WHEN DaysSinceLastUse > @UnusedThresholdDays THEN 3
            ELSE 4
        END,
        PlanMatchFailures DESC;
END;
```

### Parallelism Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeParallelQueries
AS
BEGIN
    -- Analyze parallel query patterns
    SELECT 
        qs.query_hash,
        qt.query_sql_text,
        COUNT(*) as ExecutionCount,
        AVG(qs.total_elapsed_time * 1.0 / 
            qs.execution_count) as AvgDurationMs,
        AVG(qs.total_worker_time * 1.0 / 
            qs.execution_count) as AvgCPUTimeMs,
        MAX(qs.max_dop) as MaxDOP,
        AVG(qs.total_spills * 1.0 / 
            qs.execution_count) as AvgSpills,
        MAX(qs.last_dop) as LastDOP,
        SUM(qs.total_worker_time) / 
            SUM(qs.execution_count * qs.max_dop) 
                as AvgCPUPerThread,
        CASE 
            WHEN MAX(qs.max_dop) > 8 
                 AND AVG(qs.total_spills * 1.0 / 
                        qs.execution_count) > 100 
            THEN 'High DOP with Spills'
            WHEN MAX(qs.max_dop) > 8 
            THEN 'High DOP'
            WHEN AVG(qs.total_spills * 1.0 / 
                    qs.execution_count) > 100 
            THEN 'Excessive Spills'
            WHEN SUM(qs.total_worker_time) / 
                 SUM(qs.execution_count * qs.max_dop) < 
                 1000000  -- 1 second
            THEN 'Low Thread Utilization'
            ELSE 'Normal'
        END as ParallelStatus,
        CASE 
            WHEN MAX(qs.max_dop) > 8 
                 AND AVG(qs.total_spills * 1.0 / 
                        qs.execution_count) > 100 
            THEN 'Reduce MAXDOP setting'
            WHEN MAX(qs.max_dop) > 8 
            THEN 'Review DOP requirement'
            WHEN AVG(qs.total_spills * 1.0 / 
                    qs.execution_count) > 100 
            THEN 'Increase memory grant'
            WHEN SUM(qs.total_worker_time) / 
                 SUM(qs.execution_count * qs.max_dop) < 
                 1000000 
            THEN 'Consider serial execution'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    WHERE qs.max_dop > 1
    GROUP BY 
        qs.query_hash,
        qt.query_sql_text
    HAVING MAX(qs.max_dop) > 8
    OR AVG(qs.total_spills * 1.0 / qs.execution_count) > 100
    OR SUM(qs.total_worker_time) / 
       SUM(qs.execution_count * qs.max_dop) < 1000000
    ORDER BY 
        CASE 
            WHEN MAX(qs.max_dop) > 8 
                 AND AVG(qs.total_spills * 1.0 / 
                        qs.execution_count) > 100 THEN 1
            WHEN MAX(qs.max_dop) > 8 THEN 2
            ELSE 3
        END,
        AvgDurationMs DESC;
END;
```

### Optimization Hint Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeQueryHints
AS
BEGIN
    -- Analyze query hint usage patterns
    SELECT 
        qt.query_sql_text,
        COUNT(*) as ExecutionCount,
        AVG(qs.total_elapsed_time * 1.0 / 
            qs.execution_count) as AvgDurationMs,
        MIN(qs.min_elapsed_time) / 1000.0 as MinDurationMs,
        MAX(qs.max_elapsed_time) / 1000.0 as MaxDurationMs,
        AVG(qs.total_worker_time * 1.0 / 
            qs.execution_count) as AvgCPUTimeMs,
        SUM(qs.total_logical_reads) / 
            SUM(qs.execution_count) as AvgLogicalReads,
        CASE 
            WHEN qt.query_sql_text LIKE '%FORCE ORDER%' THEN 1 
            ELSE 0 
        END as HasForceOrder,
        CASE 
            WHEN qt.query_sql_text LIKE '%OPTIMIZE FOR%' THEN 1 
            ELSE 0 
        END as HasOptimizeFor,
        CASE 
            WHEN qt.query_sql_text LIKE '%RECOMPILE%' THEN 1 
            ELSE 0 
        END as HasRecompile,
        CASE 
            WHEN qt.query_sql_text LIKE '%MAXDOP%' THEN 1 
            ELSE 0 
        END as HasMaxDOP,
        CASE 
            WHEN MAX(qs.max_elapsed_time) > 
                 MIN(qs.min_elapsed_time) * 5 
            THEN 'High Variance'
            WHEN AVG(qs.total_elapsed_time * 1.0 / 
                    qs.execution_count) > 1000000  -- 1 second
            THEN 'Poor Performance'
            WHEN COUNT(*) > 1000 
                 AND qt.query_sql_text LIKE '%RECOMPILE%' 
            THEN 'Frequent Recompile'
            ELSE 'Normal'
        END as HintPattern,
        CASE 
            WHEN MAX(qs.max_elapsed_time) > 
                 MIN(qs.min_elapsed_time) * 5 
            THEN 'Review hint effectiveness'
            WHEN AVG(qs.total_elapsed_time * 1.0 / 
                    qs.execution_count) > 1000000 
            THEN 'Evaluate hint impact'
            WHEN COUNT(*) > 1000 
                 AND qt.query_sql_text LIKE '%RECOMPILE%' 
            THEN 'Consider removing RECOMPILE'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    WHERE qt.query_sql_text LIKE '%FORCE ORDER%'
    OR qt.query_sql_text LIKE '%OPTIMIZE FOR%'
    OR qt.query_sql_text LIKE '%RECOMPILE%'
    OR qt.query_sql_text LIKE '%MAXDOP%'
    GROUP BY 
        qt.query_sql_text
    HAVING MAX(qs.max_elapsed_time) > 
           MIN(qs.min_elapsed_time) * 5
    OR AVG(qs.total_elapsed_time * 1.0 / 
           qs.execution_count) > 1000000
    OR COUNT(*) > 1000 
       AND qt.query_sql_text LIKE '%RECOMPILE%'
    ORDER BY 
        CASE 
            WHEN MAX(qs.max_elapsed_time) > 
                 MIN(qs.min_elapsed_time) * 5 THEN 1
            WHEN AVG(qs.total_elapsed_time * 1.0 / 
                    qs.execution_count) > 1000000 THEN 2
            ELSE 3
        END,
        AvgDurationMs DESC;
END;
```

This Query Optimization analysis framework provides comprehensive tools for:
1. Monitoring plan guide effectiveness and usage patterns
2. Analyzing parallel query performance and resource utilization
3. Tracking query hint impact and optimization choices
4. Optimizing query execution strategies

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
