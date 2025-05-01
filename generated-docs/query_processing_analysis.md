# SQL Server Query Processing Analysis Framework

## Cardinality Estimation Monitoring

### Row Estimation Analysis
```sql
CREATE TABLE dbo.CardinalityMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseId int,
    ObjectId int,
    StatisticsId int,
    StatisticsName sysname,
    ColumnName sysname,
    LastUpdated datetime2,
    RowCount bigint,
    ModificationCount bigint,
    SamplingRate decimal(5,2),
    EstimationAccuracy decimal(5,2),
    StaleStatsThreshold int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorCardinalityEstimation
    @AccuracyThreshold decimal(5,2) = 80.0,
    @ModificationThreshold decimal(5,2) = 20.0
AS
BEGIN
    -- Capture cardinality estimation metrics
    INSERT INTO dbo.CardinalityMetrics
    SELECT 
        CAST(pa.value AS int) as DatabaseId,
        s.object_id as ObjectId,
        s.stats_id as StatisticsId,
        s.name as StatisticsName,
        c.name as ColumnName,
        STATS_DATE(s.object_id, s.stats_id) as LastUpdated,
        CAST(sp.rows AS bigint) as RowCount,
        CAST(sp.modification_counter AS bigint) as ModificationCount,
        CAST(sp.rows_sampled * 100.0 / 
             NULLIF(sp.rows, 0) as decimal(5,2)) as SamplingRate,
        CAST(
            CASE 
                WHEN sp.rows = 0 THEN 100
                ELSE (1 - ABS(
                    sp.rows - sp.rows_sampled
                ) * 1.0 / sp.rows) * 100 
            END as decimal(5,2)
        ) as EstimationAccuracy,
        CAST(
            CASE 
                WHEN sp.rows < 500 THEN 500
                WHEN sp.rows < 10000 THEN sp.rows * 0.20
                ELSE 2000
            END as int
        ) as StaleStatsThreshold,
        GETUTCDATE()
    FROM sys.stats s
    CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
    JOIN sys.columns c 
        ON s.object_id = c.object_id 
        AND s.stats_column_id = c.column_id
    CROSS APPLY sys.dm_exec_plan_attributes(
        @@SPID
    ) pa
    WHERE pa.attribute = 'dbid';

    -- Analyze estimation patterns
    WITH CardinalityStats AS (
        SELECT 
            DatabaseId,
            ObjectId,
            StatisticsName,
            ColumnName,
            LastUpdated,
            RowCount,
            ModificationCount,
            SamplingRate,
            EstimationAccuracy,
            CAST(
                ModificationCount * 100.0 / 
                NULLIF(RowCount, 0) as decimal(5,2)
            ) as ModificationPercent,
            DATEDIFF(
                HOUR, 
                LastUpdated, 
                GETUTCDATE()
            ) as HoursSinceUpdate
        FROM dbo.CardinalityMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DB_NAME(DatabaseId) as DatabaseName,
        OBJECT_NAME(ObjectId) as TableName,
        StatisticsName,
        ColumnName,
        LastUpdated,
        RowCount,
        ModificationCount,
        SamplingRate,
        EstimationAccuracy,
        ModificationPercent,
        HoursSinceUpdate,
        CASE 
            WHEN EstimationAccuracy < @AccuracyThreshold 
                 AND ModificationPercent > @ModificationThreshold 
            THEN 'Critical Stats'
            WHEN EstimationAccuracy < @AccuracyThreshold 
            THEN 'Low Accuracy'
            WHEN ModificationPercent > @ModificationThreshold 
            THEN 'High Modifications'
            WHEN SamplingRate < 50 
            THEN 'Low Sampling'
            ELSE 'Normal'
        END as StatsStatus,
        CASE 
            WHEN EstimationAccuracy < @AccuracyThreshold 
                 AND ModificationPercent > @ModificationThreshold 
            THEN 'Update statistics with fullscan'
            WHEN EstimationAccuracy < @AccuracyThreshold 
            THEN 'Increase sampling rate'
            WHEN ModificationPercent > @ModificationThreshold 
            THEN 'Schedule stats update'
            WHEN SamplingRate < 50 
            THEN 'Review sampling strategy'
            ELSE 'No action needed'
        END as Recommendation
    FROM CardinalityStats
    WHERE EstimationAccuracy < @AccuracyThreshold
    OR ModificationPercent > @ModificationThreshold
    OR SamplingRate < 50
    ORDER BY 
        CASE 
            WHEN EstimationAccuracy < @AccuracyThreshold 
                 AND ModificationPercent > @ModificationThreshold THEN 1
            WHEN EstimationAccuracy < @AccuracyThreshold THEN 2
            WHEN ModificationPercent > @ModificationThreshold THEN 3
            ELSE 4
        END,
        ModificationCount DESC;
END;
```

### Plan Quality Analysis
```sql
CREATE PROCEDURE dbo.AnalyzePlanQuality
AS
BEGIN
    -- Analyze execution plan quality patterns
    SELECT 
        p.query_id,
        p.plan_id,
        qt.query_sql_text,
        p.engine_version,
        p.compatibility_level,
        p.query_plan_hash,
        rs.count_executions,
        rs.avg_duration / 1000.0 as AvgDurationMs,
        rs.avg_cpu_time / 1000.0 as AvgCPUTimeMs,
        rs.avg_logical_io_reads as AvgLogicalReads,
        rs.avg_query_max_used_memory as AvgMemoryKB,
        CONVERT(xml, p.query_plan) as QueryPlan,
        CASE 
            WHEN p.engine_version != (
                SELECT SERVERPROPERTY('ProductVersion')
            ) THEN 'Version Mismatch'
            WHEN COUNT(*) OVER (
                PARTITION BY p.query_id
            ) > 1 THEN 'Multiple Plans'
            WHEN rs.avg_duration > 1000000  -- 1 second
                 AND rs.count_executions > 100 
            THEN 'Poor Performance'
            ELSE 'Normal'
        END as PlanStatus,
        CASE 
            WHEN p.engine_version != (
                SELECT SERVERPROPERTY('ProductVersion')
            ) THEN 'Recompile with current version'
            WHEN COUNT(*) OVER (
                PARTITION BY p.query_id
            ) > 1 THEN 'Review plan choice'
            WHEN rs.avg_duration > 1000000 
                 AND rs.count_executions > 100 
            THEN 'Optimize query performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_plan p
    JOIN sys.query_store_query q 
        ON p.query_id = q.query_id
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE rs.last_execution_time >= 
          DATEADD(HOUR, -1, GETUTCDATE())
    AND (
        p.engine_version != (
            SELECT SERVERPROPERTY('ProductVersion')
        )
        OR EXISTS (
            SELECT 1
            FROM sys.query_store_plan p2
            WHERE p2.query_id = p.query_id
            AND p2.plan_id != p.plan_id
        )
        OR rs.avg_duration > 1000000 
        AND rs.count_executions > 100
    )
    ORDER BY 
        CASE 
            WHEN p.engine_version != (
                SELECT SERVERPROPERTY('ProductVersion')
            ) THEN 1
            WHEN COUNT(*) OVER (
                PARTITION BY p.query_id
            ) > 1 THEN 2
            ELSE 3
        END,
        rs.avg_duration DESC;
END;
```

### Parameter Sensitivity Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeParameterSensitivity
AS
BEGIN
    -- Analyze parameter sniffing patterns
    SELECT 
        q.query_id,
        qt.query_sql_text,
        COUNT(DISTINCT p.plan_id) as PlanCount,
        MIN(rs.avg_duration) / 1000.0 as MinDurationMs,
        MAX(rs.avg_duration) / 1000.0 as MaxDurationMs,
        AVG(rs.avg_duration) / 1000.0 as AvgDurationMs,
        STDEV(rs.avg_duration) / 1000.0 as StdDevDurationMs,
        SUM(rs.count_executions) as TotalExecutions,
        STRING_AGG(
            'Plan ' + CAST(p.plan_id as varchar(20)) + 
            ' (' + 
            CAST(rs.avg_duration / 1000.0 as varchar(20)) + 
            'ms)',
            ', '
        ) as PlanDurations,
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 
            THEN 'Highly Sensitive'
            WHEN MAX(rs.avg_duration) > 
                 MIN(rs.avg_duration) * 2 
            THEN 'Variable Performance'
            WHEN COUNT(DISTINCT p.plan_id) > 1 
            THEN 'Plan Choice Variance'
            ELSE 'Stable'
        END as SensitivityStatus,
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 
            THEN 'Consider OPTION(RECOMPILE)'
            WHEN MAX(rs.avg_duration) > 
                 MIN(rs.avg_duration) * 2 
            THEN 'Review parameter distributions'
            WHEN COUNT(DISTINCT p.plan_id) > 1 
            THEN 'Monitor plan stability'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE qt.query_sql_text LIKE N'%@%'
    AND rs.last_execution_time >= 
        DATEADD(DAY, -7, GETUTCDATE())
    GROUP BY 
        q.query_id,
        qt.query_sql_text
    HAVING COUNT(DISTINCT p.plan_id) > 1
    OR MAX(rs.avg_duration) > MIN(rs.avg_duration) * 2
    ORDER BY 
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 THEN 1
            WHEN MAX(rs.avg_duration) > 
                 MIN(rs.avg_duration) * 2 THEN 2
            ELSE 3
        END,
        MaxDurationMs DESC;
END;
```

This Query Processing analysis framework provides comprehensive tools for:
1. Monitoring cardinality estimation accuracy and statistics health
2. Analyzing execution plan quality and version compatibility
3. Tracking parameter sensitivity and plan choice variance
4. Optimizing query processing performance

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
