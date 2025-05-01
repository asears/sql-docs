# SQL Server Change Data Capture Analysis Framework

## CDC Monitoring Framework

### CDC Performance Analysis
```sql
CREATE TABLE dbo.CDCMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    CaptureInstanceName sysname,
    TotalChanges bigint,
    PendingChanges bigint,
    CaptureLatencyMs decimal(18,2),
    CleanupLatencyMs decimal(18,2),
    StorageUsedKB bigint,
    RetentionPeriodHours int,
    LastCaptureTime datetime2,
    LastCleanupTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorCDCPerformance
    @HighLatencyThresholdMs decimal(18,2) = 5000.0,
    @HighPendingThreshold int = 10000
AS
BEGIN
    -- Capture CDC metrics
    INSERT INTO dbo.CDCMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        ci.capture_instance as CaptureInstanceName,
        (
            SELECT COUNT(*) 
            FROM cdc.fn_cdc_get_all_changes_capture_instance(
                ci.capture_instance,
                sys.fn_cdc_get_min_lsn(ci.capture_instance),
                sys.fn_cdc_get_max_lsn(),
                'all'
            )
        ) as TotalChanges,
        (
            SELECT COUNT(*) 
            FROM cdc.fn_cdc_get_all_changes_capture_instance(
                ci.capture_instance,
                sys.fn_cdc_map_time_to_lsn('largest less than or equal', 
                    DATEADD(MINUTE, -5, GETDATE())),
                sys.fn_cdc_get_max_lsn(),
                'all'
            )
        ) as PendingChanges,
        DATEDIFF(
            MILLISECOND,
            ct.tran_begin_time,
            ct.tran_end_time
        ) as CaptureLatencyMs,
        DATEDIFF(
            MILLISECOND,
            cc.cleanup_start_time,
            cc.cleanup_end_time
        ) as CleanupLatencyMs,
        (
            SELECT SUM(used_page_count) * 8 
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = ci.object_id
        ) as StorageUsedKB,
        ci.retention as RetentionPeriodHours,
        ct.tran_end_time as LastCaptureTime,
        cc.cleanup_end_time as LastCleanupTime,
        GETUTCDATE()
    FROM sys.tables t
    JOIN cdc.change_tables ct 
        ON t.object_id = ct.source_object_id
    JOIN cdc.captured_columns cc 
        ON ct.object_id = cc.object_id
    JOIN cdc.capture_instances ci 
        ON t.object_id = ci.source_object_id;

    -- Analyze CDC patterns
    WITH CDCMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            TableName,
            CaptureInstanceName,
            TotalChanges,
            PendingChanges,
            CaptureLatencyMs,
            CleanupLatencyMs,
            StorageUsedKB / 1024.0 as StorageUsedMB,
            RetentionPeriodHours,
            LastCaptureTime,
            LastCleanupTime,
            LAG(PendingChanges) OVER (
                PARTITION BY DatabaseName, 
                             SchemaName, 
                             TableName 
                ORDER BY CollectionTime
            ) as PreviousPendingChanges
        FROM dbo.CDCMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        CaptureInstanceName,
        TotalChanges,
        PendingChanges,
        CaptureLatencyMs,
        CleanupLatencyMs,
        StorageUsedMB,
        RetentionPeriodHours,
        DATEDIFF(
            MINUTE, 
            LastCaptureTime, 
            GETDATE()
        ) as MinutesSinceLastCapture,
        CASE 
            WHEN CaptureLatencyMs > @HighLatencyThresholdMs 
            THEN 'High Latency'
            WHEN PendingChanges > @HighPendingThreshold 
            THEN 'High Pending Changes'
            WHEN PendingChanges > COALESCE(
                PreviousPendingChanges, 
                0
            ) * 1.5 
            THEN 'Growing Backlog'
            ELSE 'Normal'
        END as CDCStatus,
        CASE 
            WHEN CaptureLatencyMs > @HighLatencyThresholdMs 
            THEN 'Review:
                  1. CDC agent performance
                  2. Transaction log size
                  3. Network latency'
            WHEN PendingChanges > @HighPendingThreshold 
            THEN 'Investigate capture job performance'
            WHEN PendingChanges > COALESCE(
                PreviousPendingChanges, 
                0
            ) * 1.5 
            THEN 'Monitor change processing rate'
            ELSE 'No action needed'
        END as Recommendation
    FROM CDCMetrics
    WHERE CaptureLatencyMs > @HighLatencyThresholdMs
    OR PendingChanges > @HighPendingThreshold
    OR PendingChanges > COALESCE(PreviousPendingChanges, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN CaptureLatencyMs > @HighLatencyThresholdMs THEN 1
            WHEN PendingChanges > @HighPendingThreshold THEN 2
            ELSE 3
        END,
        CaptureLatencyMs DESC;
END;
```

### CDC Agent Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeCDCAgents
AS
BEGIN
    -- Analyze CDC agent performance
    SELECT 
        j.name as JobName,
        ja.start_execution_date,
        ja.stop_execution_date,
        DATEDIFF(
            SECOND,
            ja.start_execution_date,
            COALESCE(
                ja.stop_execution_date,
                GETDATE()
            )
        ) as DurationSeconds,
        ja.job_history_id,
        ja.run_status,
        ja.message,
        jh.step_id,
        jh.step_name,
        CASE js.last_run_outcome
            WHEN 0 THEN 'Failed'
            WHEN 1 THEN 'Succeeded'
            WHEN 2 THEN 'Retry'
            WHEN 3 THEN 'Canceled'
            WHEN 4 THEN 'In Progress'
            ELSE 'Unknown'
        END as LastRunStatus,
        js.last_run_duration as LastRunDurationSeconds,
        js.last_run_retries as RetryCount,
        CASE 
            WHEN ja.run_status = 0 
            THEN 'Failed'
            WHEN DATEDIFF(
                MINUTE,
                ja.start_execution_date,
                COALESCE(
                    ja.stop_execution_date,
                    GETDATE()
                )
            ) > 60 
            THEN 'Long Running'
            WHEN js.last_run_retries > 3 
            THEN 'Multiple Retries'
            ELSE 'Normal'
        END as AgentStatus,
        CASE 
            WHEN ja.run_status = 0 
            THEN 'Investigate agent failure'
            WHEN DATEDIFF(
                MINUTE,
                ja.start_execution_date,
                COALESCE(
                    ja.stop_execution_date,
                    GETDATE()
                )
            ) > 60 
            THEN 'Review agent performance'
            WHEN js.last_run_retries > 3 
            THEN 'Check for transient issues'
            ELSE 'No action needed'
        END as Recommendation
    FROM msdb.dbo.sysjobs j
    JOIN msdb.dbo.sysjobactivity ja 
        ON j.job_id = ja.job_id
    JOIN msdb.dbo.sysjobsteps js 
        ON j.job_id = js.job_id
    LEFT JOIN msdb.dbo.sysjobhistory jh 
        ON ja.job_history_id = jh.instance_id
    WHERE j.name LIKE 'cdc%'
    AND (
        ja.run_status = 0
        OR DATEDIFF(
            MINUTE,
            ja.start_execution_date,
            COALESCE(
                ja.stop_execution_date,
                GETDATE()
            )
        ) > 60
        OR js.last_run_retries > 3
    )
    ORDER BY 
        CASE 
            WHEN ja.run_status = 0 THEN 1
            WHEN js.last_run_retries > 3 THEN 2
            ELSE 3
        END,
        ja.start_execution_date DESC;
END;
```

### CDC Storage Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeCDCStorage
    @HighStorageThresholdMB decimal(10,2) = 1024.0,
    @HighGrowthRatePercent decimal(5,2) = 20.0
AS
BEGIN
    -- Analyze CDC storage patterns
    WITH StorageMetrics AS (
        SELECT 
            DB_NAME() as DatabaseName,
            OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
            t.name as TableName,
            ci.capture_instance as CaptureInstanceName,
            (
                SELECT SUM(used_page_count) * 8.0 / 1024 
                FROM sys.dm_db_partition_stats ps
                WHERE ps.object_id = ct.object_id
            ) as StorageUsedMB,
            (
                SELECT COUNT(*) 
                FROM sys.dm_cdc_log_scan_sessions
                WHERE start_time >= DATEADD(HOUR, -24, GETDATE())
                AND source_table = t.object_id
            ) as ScanCount24Hours,
            ci.retention as RetentionHours,
            MAX(ls.start_time) as LastScanTime,
            MAX(ls.duration) as MaxScanDuration,
            AVG(ls.duration) as AvgScanDuration,
            COUNT(DISTINCT ls.session_id) as ScanSessions
        FROM sys.tables t
        JOIN cdc.change_tables ct 
            ON t.object_id = ct.source_object_id
        JOIN cdc.capture_instances ci 
            ON t.object_id = ci.source_object_id
        LEFT JOIN sys.dm_cdc_log_scan_sessions ls 
            ON t.object_id = ls.source_table
        GROUP BY 
            t.object_id,
            t.name,
            ci.capture_instance,
            ct.object_id,
            ci.retention
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        CaptureInstanceName,
        StorageUsedMB,
        ScanCount24Hours,
        RetentionHours,
        LastScanTime,
        MaxScanDuration,
        AvgScanDuration,
        ScanSessions,
        CASE 
            WHEN StorageUsedMB > @HighStorageThresholdMB 
                 AND ScanCount24Hours > 1000 
            THEN 'High Usage'
            WHEN StorageUsedMB > @HighStorageThresholdMB 
            THEN 'High Storage'
            WHEN ScanCount24Hours > 1000 
            THEN 'High Scan Rate'
            ELSE 'Normal'
        END as StoragePattern,
        CASE 
            WHEN StorageUsedMB > @HighStorageThresholdMB 
                 AND ScanCount24Hours > 1000 
            THEN 'Consider:
                  1. Reducing retention period
                  2. Archiving older changes
                  3. Optimizing scan frequency'
            WHEN StorageUsedMB > @HighStorageThresholdMB 
            THEN 'Review retention settings'
            WHEN ScanCount24Hours > 1000 
            THEN 'Analyze scan patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM StorageMetrics
    WHERE StorageUsedMB > @HighStorageThresholdMB
    OR ScanCount24Hours > 1000
    ORDER BY 
        CASE 
            WHEN StorageUsedMB > @HighStorageThresholdMB 
                 AND ScanCount24Hours > 1000 THEN 1
            WHEN StorageUsedMB > @HighStorageThresholdMB THEN 2
            ELSE 3
        END,
        StorageUsedMB DESC;
END;
```

This CDC analysis framework provides comprehensive tools for:
1. Monitoring CDC performance and latency
2. Analyzing CDC agent health and execution patterns
3. Tracking CDC storage usage and growth
4. Optimizing CDC operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
