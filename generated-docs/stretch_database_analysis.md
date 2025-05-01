# SQL Server Stretch Database Analysis Framework

## Stretch Database Performance Monitoring

### Remote Data Analysis
```sql
CREATE TABLE dbo.StretchMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    TableName sysname,
    LocalRowCount bigint,
    RemoteRowCount bigint,
    DataMovementLatencyMs decimal(18,2),
    DataMovementBytes bigint,
    QueryCount int,
    RemoteQueryLatencyMs decimal(18,2),
    RemoteIOBytes bigint,
    ErrorCount int,
    LastSyncTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorStretchPerformance
    @HighLatencyThresholdMs decimal(18,2) = 1000.0,
    @DataMovementThresholdMB decimal(18,2) = 100.0
AS
BEGIN
    -- Capture Stretch Database metrics
    INSERT INTO dbo.StretchMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        t.name as TableName,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = t.object_id
            AND ps.index_id <= 1
        ) as LocalRowCount,
        srt.remote_table_rows as RemoteRowCount,
        sdo.data_movement_latency_ms as DataMovementLatencyMs,
        sdo.data_movement_bytes_transferred as DataMovementBytes,
        sdo.remote_query_count as QueryCount,
        sdo.remote_query_latency_ms as RemoteQueryLatencyMs,
        sdo.remote_query_bytes_transferred as RemoteIOBytes,
        sdo.error_count as ErrorCount,
        sdo.last_sync_time as LastSyncTime,
        GETUTCDATE()
    FROM sys.tables t
    JOIN sys.remote_tables srt 
        ON t.object_id = srt.object_id
    JOIN sys.dm_db_stretch_database_operations sdo 
        ON t.object_id = sdo.object_id;

    -- Analyze Stretch Database patterns
    WITH StretchMetrics AS (
        SELECT 
            DatabaseName,
            TableName,
            LocalRowCount,
            RemoteRowCount,
            DataMovementLatencyMs,
            DataMovementBytes / 1048576.0 as DataMovementMB,
            QueryCount,
            RemoteQueryLatencyMs,
            RemoteIOBytes / 1048576.0 as RemoteIOMB,
            ErrorCount,
            LastSyncTime,
            DATEDIFF(
                MINUTE, 
                LastSyncTime, 
                GETUTCDATE()
            ) as MinutesSinceSync,
            LAG(RemoteRowCount) OVER (
                PARTITION BY DatabaseName, TableName 
                ORDER BY CollectionTime
            ) as PreviousRemoteRows
        FROM dbo.StretchMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        TableName,
        LocalRowCount,
        RemoteRowCount,
        DataMovementLatencyMs,
        DataMovementMB,
        QueryCount,
        RemoteQueryLatencyMs,
        RemoteIOMB,
        ErrorCount,
        MinutesSinceSync,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Sync Errors'
            WHEN DataMovementLatencyMs > @HighLatencyThresholdMs 
            THEN 'High Movement Latency'
            WHEN RemoteQueryLatencyMs > @HighLatencyThresholdMs 
            THEN 'High Query Latency'
            WHEN DataMovementMB > @DataMovementThresholdMB 
            THEN 'High Data Movement'
            ELSE 'Normal'
        END as StretchStatus,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Investigate sync errors'
            WHEN DataMovementLatencyMs > @HighLatencyThresholdMs 
            THEN 'Check network connectivity'
            WHEN RemoteQueryLatencyMs > @HighLatencyThresholdMs 
            THEN 'Review remote query patterns'
            WHEN DataMovementMB > @DataMovementThresholdMB 
            THEN 'Monitor data movement'
            ELSE 'No action needed'
        END as Recommendation
    FROM StretchMetrics
    WHERE ErrorCount > 0
    OR DataMovementLatencyMs > @HighLatencyThresholdMs
    OR RemoteQueryLatencyMs > @HighLatencyThresholdMs
    OR DataMovementMB > @DataMovementThresholdMB
    ORDER BY 
        CASE 
            WHEN ErrorCount > 0 THEN 1
            WHEN DataMovementLatencyMs > @HighLatencyThresholdMs THEN 2
            WHEN RemoteQueryLatencyMs > @HighLatencyThresholdMs THEN 3
            ELSE 4
        END,
        DataMovementLatencyMs DESC;
END;
```

### Remote Query Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeStretchQueries
AS
BEGIN
    -- Analyze remote query patterns
    SELECT 
        DB_NAME(qt.dbid) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qs.execution_count,
        qs.total_elapsed_time * 1.0 / 
            qs.execution_count as AvgExecutionTimeMs,
        qs.total_worker_time * 1.0 / 
            qs.execution_count as AvgCPUTimeMs,
        qs.total_logical_reads * 1.0 / 
            qs.execution_count as AvgLogicalReads,
        qs.total_physical_reads * 1.0 / 
            qs.execution_count as AvgPhysicalReads,
        qp.query_plan,
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 
            THEN 'Long Running'
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 
            THEN 'High I/O'
            WHEN qs.execution_count > 1000 
            THEN 'Frequently Used'
            ELSE 'Normal'
        END as QueryPattern,
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 
            THEN 'Review remote data access'
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 
            THEN 'Optimize query patterns'
            WHEN qs.execution_count > 1000 
            THEN 'Monitor performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE EXISTS (
        SELECT 1
        FROM sys.tables t
        JOIN sys.remote_tables srt 
            ON t.object_id = srt.object_id
        WHERE qt.text LIKE '%' + t.name + '%'
    )
    AND (
        qs.total_elapsed_time * 1.0 / qs.execution_count > 1000
        OR qs.total_logical_reads * 1.0 / qs.execution_count > 10000
        OR qs.execution_count > 1000
    )
    ORDER BY 
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 THEN 1
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 THEN 2
            ELSE 3
        END,
        qs.total_elapsed_time DESC;
END;
```

### Data Movement Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeDataMovement
AS
BEGIN
    -- Analyze data movement operations
    SELECT 
        t.name as TableName,
        srt.remote_table_rows as RemoteRows,
        sdo.data_movement_latency_ms as MovementLatencyMs,
        sdo.data_movement_bytes_transferred / 1048576.0 as MovementMB,
        sdo.remote_query_count,
        sdo.remote_query_latency_ms as QueryLatencyMs,
        sdo.remote_query_bytes_transferred / 1048576.0 as QueryDataMB,
        sdo.error_count,
        sdo.last_sync_time,
        DATEDIFF(
            MINUTE, 
            sdo.last_sync_time, 
            GETUTCDATE()
        ) as MinutesSinceSync,
        CASE 
            WHEN sdo.error_count > 0 
            THEN 'Movement Errors'
            WHEN sdo.data_movement_latency_ms > 1000 
            THEN 'High Movement Latency'
            WHEN DATEDIFF(
                MINUTE, 
                sdo.last_sync_time, 
                GETUTCDATE()
            ) > 60 
            THEN 'Stale Sync'
            ELSE 'Normal'
        END as MovementStatus,
        CASE 
            WHEN sdo.error_count > 0 
            THEN 'Review movement errors'
            WHEN sdo.data_movement_latency_ms > 1000 
            THEN 'Check network performance'
            WHEN DATEDIFF(
                MINUTE, 
                sdo.last_sync_time, 
                GETUTCDATE()
            ) > 60 
            THEN 'Verify sync schedule'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.tables t
    JOIN sys.remote_tables srt 
        ON t.object_id = srt.object_id
    JOIN sys.dm_db_stretch_database_operations sdo 
        ON t.object_id = sdo.object_id
    WHERE sdo.error_count > 0
    OR sdo.data_movement_latency_ms > 1000
    OR DATEDIFF(
        MINUTE, 
        sdo.last_sync_time, 
        GETUTCDATE()
    ) > 60
    ORDER BY 
        CASE 
            WHEN sdo.error_count > 0 THEN 1
            WHEN sdo.data_movement_latency_ms > 1000 THEN 2
            ELSE 3
        END,
        MovementLatencyMs DESC;
END;
```

This Stretch Database analysis framework provides comprehensive tools for:
1. Monitoring remote data access and movement performance
2. Analyzing remote query patterns and efficiency
3. Tracking data movement operations and sync status
4. Optimizing hybrid database operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
