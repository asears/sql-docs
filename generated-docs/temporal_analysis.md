# SQL Server Temporal Tables Analysis Framework

## Temporal Query Performance Monitoring

### Temporal Table Analysis
```sql
CREATE TABLE dbo.TemporalMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    HistoryTableName sysname,
    CurrentRowCount bigint,
    HistoryRowCount bigint,
    RetentionPeriodDays int,
    TemporalQueriesCount int,
    AvgDurationMs decimal(18,2),
    DataSizeMB decimal(18,2),
    HistorySizeMB decimal(18,2),
    LastCleanupTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorTemporalTables
    @HighHistoryRatioThreshold decimal(5,2) = 500.0,
    @LongQueryThresholdMs decimal(18,2) = 1000.0
AS
BEGIN
    -- Capture temporal table metrics
    INSERT INTO dbo.TemporalMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        OBJECT_NAME(h.history_table_id) as HistoryTableName,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = t.object_id
            AND ps.index_id <= 1
        ) as CurrentRowCount,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = h.history_table_id
            AND ps.index_id <= 1
        ) as HistoryRowCount,
        h.retention_period_unit_desc as RetentionPeriodDays,
        COUNT(DISTINCT qs.query_id) as TemporalQueriesCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        SUM(
            CASE 
                WHEN ps.object_id = t.object_id 
                THEN ps.used_page_count 
                ELSE 0 
            END
        ) * 8.0 / 1024 as DataSizeMB,
        SUM(
            CASE 
                WHEN ps.object_id = h.history_table_id 
                THEN ps.used_page_count 
                ELSE 0 
            END
        ) * 8.0 / 1024 as HistorySizeMB,
        MAX(h.cleaned_up_to) as LastCleanupTime,
        GETUTCDATE()
    FROM sys.tables t
    JOIN sys.dm_db_partition_stats ps 
        ON t.object_id = ps.object_id 
        OR ps.object_id IN (
            SELECT history_table_id 
            FROM sys.internal_tables 
            WHERE parent_id = t.object_id
        )
    JOIN sys.internal_tables h 
        ON t.object_id = h.parent_object_id
    LEFT JOIN sys.query_store_query qs 
        ON CHARINDEX(
            t.name, 
            OBJECT_NAME(
                OBJECT_ID(
                    qs.query_sql_text
                )
            )
        ) > 0
    LEFT JOIN sys.query_store_runtime_stats rs 
        ON qs.query_id = rs.plan_id
    WHERE t.temporal_type = 2  -- System-versioned temporal table
    GROUP BY 
        t.object_id,
        t.name,
        h.history_table_id,
        h.retention_period_unit_desc,
        h.cleaned_up_to;

    -- Analyze temporal patterns
    WITH TemporalMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            TableName,
            HistoryTableName,
            CurrentRowCount,
            HistoryRowCount,
            RetentionPeriodDays,
            TemporalQueriesCount,
            AvgDurationMs,
            DataSizeMB,
            HistorySizeMB,
            LastCleanupTime,
            CAST(
                HistoryRowCount * 100.0 / 
                NULLIF(CurrentRowCount, 0) as decimal(5,2)
            ) as HistoryRatio
        FROM dbo.TemporalMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -24, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        HistoryTableName,
        CurrentRowCount,
        HistoryRowCount,
        RetentionPeriodDays,
        TemporalQueriesCount,
        AvgDurationMs,
        DataSizeMB,
        HistorySizeMB,
        LastCleanupTime,
        HistoryRatio,
        CASE 
            WHEN HistoryRatio > @HighHistoryRatioThreshold 
                 AND AvgDurationMs > @LongQueryThresholdMs 
            THEN 'Critical Performance'
            WHEN HistoryRatio > @HighHistoryRatioThreshold 
            THEN 'High History Ratio'
            WHEN AvgDurationMs > @LongQueryThresholdMs 
            THEN 'Slow Queries'
            WHEN DATEDIFF(
                DAY,
                LastCleanupTime,
                GETUTCDATE()
            ) > RetentionPeriodDays 
            THEN 'Cleanup Needed'
            ELSE 'Normal'
        END as TableStatus,
        CASE 
            WHEN HistoryRatio > @HighHistoryRatioThreshold 
                 AND AvgDurationMs > @LongQueryThresholdMs 
            THEN 'Review:
                  1. Retention policy
                  2. Index strategy
                  3. Query patterns'
            WHEN HistoryRatio > @HighHistoryRatioThreshold 
            THEN 'Optimize retention period'
            WHEN AvgDurationMs > @LongQueryThresholdMs 
            THEN 'Review temporal queries'
            WHEN DATEDIFF(
                DAY,
                LastCleanupTime,
                GETUTCDATE()
            ) > RetentionPeriodDays 
            THEN 'Schedule cleanup'
            ELSE 'No action needed'
        END as Recommendation
    FROM TemporalMetrics
    WHERE HistoryRatio > @HighHistoryRatioThreshold
    OR AvgDurationMs > @LongQueryThresholdMs
    OR DATEDIFF(
        DAY,
        LastCleanupTime,
        GETUTCDATE()
    ) > RetentionPeriodDays
    ORDER BY 
        CASE 
            WHEN HistoryRatio > @HighHistoryRatioThreshold 
                 AND AvgDurationMs > @LongQueryThresholdMs THEN 1
            WHEN HistoryRatio > @HighHistoryRatioThreshold THEN 2
            ELSE 3
        END,
        HistoryRatio DESC;
END;
```

### Temporal Query Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeTemporalQueries
AS
BEGIN
    -- Analyze temporal query patterns
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        AVG(rs.avg_cpu_time) as AvgCPUTimeMs,
        AVG(rs.avg_logical_io_reads) as AvgLogicalReads,
        AVG(
            DATEDIFF(
                DAY,
                CAST(
                    SUBSTRING(
                        qt.query_sql_text,
                        CHARINDEX('FOR SYSTEM_TIME AS OF', qt.query_sql_text) + 22,
                        19
                    ) as datetime
                ),
                GETDATE()
            )
        ) as AvgHistoricalDays,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'High Impact'
            WHEN AVG(rs.avg_duration) > 1000 
            THEN 'Long Running'
            WHEN COUNT(*) > 1000 
            THEN 'Frequently Used'
            ELSE 'Normal'
        END as QueryPattern,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'Optimize temporal access'
            WHEN AVG(rs.avg_duration) > 1000 
            THEN 'Review query performance'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor usage patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE qt.query_sql_text LIKE '%FOR SYSTEM_TIME%'
    GROUP BY 
        q.query_id,
        qt.query_sql_text,
        p.query_plan
    HAVING COUNT(*) > 1000
    OR AVG(rs.avg_duration) > 1000
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 THEN 1
            WHEN AVG(rs.avg_duration) > 1000 THEN 2
            ELSE 3
        END,
        AvgDurationMs DESC;
END;
```

### Retention Management Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeRetentionManagement
AS
BEGIN
    -- Analyze retention policy effectiveness
    SELECT 
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        OBJECT_NAME(h.history_table_id) as HistoryTableName,
        h.retention_period as RetentionPeriod,
        h.retention_period_unit_desc as RetentionUnit,
        (
            SELECT MIN(SYSTEM_START_TIME)
            FROM sys.internal_tables_info i
            WHERE i.parent_id = t.object_id
        ) as OldestHistoryRecord,
        (
            SELECT COUNT(*)
            FROM sys.internal_tables_info i
            WHERE i.parent_id = t.object_id
            AND i.SYSTEM_START_TIME < DATEADD(
                DAY,
                -h.retention_period,
                GETUTCDATE()
            )
        ) as ExpiredRecordCount,
        (
            SELECT SUM(ps.used_page_count) * 8.0 / 1024
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = h.history_table_id
            AND ps.index_id <= 1
        ) as HistorySizeMB,
        h.cleaned_up_to as LastCleanupTime,
        CASE 
            WHEN EXISTS (
                SELECT 1
                FROM sys.internal_tables_info i
                WHERE i.parent_id = t.object_id
                AND i.SYSTEM_START_TIME < DATEADD(
                    DAY,
                    -h.retention_period,
                    GETUTCDATE()
                )
            ) THEN 'Cleanup Needed'
            WHEN DATEDIFF(
                DAY,
                h.cleaned_up_to,
                GETUTCDATE()
            ) > 7 
            THEN 'Maintenance Due'
            ELSE 'Normal'
        END as RetentionStatus,
        CASE 
            WHEN EXISTS (
                SELECT 1
                FROM sys.internal_tables_info i
                WHERE i.parent_id = t.object_id
                AND i.SYSTEM_START_TIME < DATEADD(
                    DAY,
                    -h.retention_period,
                    GETUTCDATE()
                )
            ) THEN 'Run cleanup process'
            WHEN DATEDIFF(
                DAY,
                h.cleaned_up_to,
                GETUTCDATE()
            ) > 7 
            THEN 'Schedule maintenance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.tables t
    JOIN sys.internal_tables h 
        ON t.object_id = h.parent_object_id
    WHERE t.temporal_type = 2  -- System-versioned temporal table
    AND (
        EXISTS (
            SELECT 1
            FROM sys.internal_tables_info i
            WHERE i.parent_id = t.object_id
            AND i.SYSTEM_START_TIME < DATEADD(
                DAY,
                -h.retention_period,
                GETUTCDATE()
            )
        )
        OR DATEDIFF(
            DAY,
            h.cleaned_up_to,
            GETUTCDATE()
        ) > 7
    )
    ORDER BY 
        CASE 
            WHEN EXISTS (
                SELECT 1
                FROM sys.internal_tables_info i
                WHERE i.parent_id = t.object_id
                AND i.SYSTEM_START_TIME < DATEADD(
                    DAY,
                    -h.retention_period,
                    GETUTCDATE()
                )
            ) THEN 1
            ELSE 2
        END,
        ExpiredRecordCount DESC;
END;
```

This Temporal Tables Analysis framework provides comprehensive tools for:
1. Monitoring temporal table performance and growth patterns
2. Analyzing temporal query execution and optimization
3. Managing retention policies and cleanup operations
4. Optimizing system-versioned temporal tables

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
