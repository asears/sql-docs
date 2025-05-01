# SQL Server FILESTREAM Analysis Framework

## FILESTREAM Monitoring Framework

### FILESTREAM I/O Analysis
```sql
CREATE TABLE dbo.FileStreamMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    TableName sysname,
    FileGroupName nvarchar(128),
    TotalSizeGB decimal(18,2),
    UsedSizeGB decimal(18,2),
    ReadsBytesSec bigint,
    WritesBytesSec bigint,
    ReadLatencyMs decimal(18,2),
    WriteLatencyMs decimal(18,2),
    GarbageBytes bigint,
    LastGarbageCollection datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorFileStream
    @HighLatencyThresholdMs decimal(18,2) = 100.0,
    @HighUtilizationThreshold decimal(5,2) = 85.0
AS
BEGIN
    -- Capture FILESTREAM metrics
    INSERT INTO dbo.FileStreamMetrics
    SELECT 
        DB_NAME(db.database_id) as DatabaseName,
        OBJECT_NAME(t.object_id) as TableName,
        fg.name as FileGroupName,
        fsf.size_in_bytes / 1073741824.0 as TotalSizeGB,
        (fsf.size_in_bytes - 
         fsf.available_bytes) / 1073741824.0 as UsedSizeGB,
        fs.read_bytes_per_sec,
        fs.write_bytes_per_sec,
        fs.read_latency_ms,
        fs.write_latency_ms,
        fsf.garbage_bytes,
        fsf.last_garbage_collection,
        GETUTCDATE()
    FROM sys.dm_filestream_file_system_stats fs
    JOIN sys.database_files f 
        ON fs.database_id = f.file_id
    JOIN sys.filegroups fg 
        ON f.data_space_id = fg.data_space_id
    JOIN sys.tables t 
        ON fg.data_space_id = t.filestream_data_space_id
    JOIN sys.databases db 
        ON fs.database_id = db.database_id
    CROSS APPLY sys.dm_filestream_file_io_stats fsf;

    -- Analyze FILESTREAM patterns
    WITH FileStreamStats AS (
        SELECT 
            DatabaseName,
            TableName,
            FileGroupName,
            TotalSizeGB,
            UsedSizeGB,
            ReadsBytesSec / 1048576.0 as ReadsMBSec,
            WritesBytesSec / 1048576.0 as WritesMBSec,
            ReadLatencyMs,
            WriteLatencyMs,
            GarbageBytes / 1073741824.0 as GarbageGB,
            LastGarbageCollection,
            UsedSizeGB * 100.0 / TotalSizeGB as UtilizationPercent,
            DATEDIFF(
                HOUR,
                LastGarbageCollection,
                GETUTCDATE()
            ) as HoursSinceGC
        FROM dbo.FileStreamMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        TableName,
        FileGroupName,
        TotalSizeGB,
        UsedSizeGB,
        ReadsMBSec,
        WritesMBSec,
        ReadLatencyMs,
        WriteLatencyMs,
        GarbageGB,
        LastGarbageCollection,
        UtilizationPercent,
        CASE 
            WHEN ReadLatencyMs > @HighLatencyThresholdMs OR 
                 WriteLatencyMs > @HighLatencyThresholdMs 
            THEN 'High Latency'
            WHEN UtilizationPercent > @HighUtilizationThreshold 
            THEN 'High Utilization'
            WHEN GarbageGB > 10 
            THEN 'High Garbage'
            ELSE 'Normal'
        END as StreamStatus,
        CASE 
            WHEN ReadLatencyMs > @HighLatencyThresholdMs OR 
                 WriteLatencyMs > @HighLatencyThresholdMs 
            THEN 'Review:
                  1. Storage performance
                  2. Network bandwidth
                  3. Concurrent access patterns'
            WHEN UtilizationPercent > @HighUtilizationThreshold 
            THEN 'Consider adding FILESTREAM containers'
            WHEN GarbageGB > 10 
            THEN 'Schedule garbage collection'
            ELSE 'No action needed'
        END as Recommendation
    FROM FileStreamStats
    WHERE ReadLatencyMs > @HighLatencyThresholdMs
    OR WriteLatencyMs > @HighLatencyThresholdMs
    OR UtilizationPercent > @HighUtilizationThreshold
    OR GarbageGB > 10
    ORDER BY 
        CASE 
            WHEN ReadLatencyMs > @HighLatencyThresholdMs OR 
                 WriteLatencyMs > @HighLatencyThresholdMs THEN 1
            WHEN UtilizationPercent > @HighUtilizationThreshold THEN 2
            ELSE 3
        END,
        COALESCE(ReadLatencyMs, WriteLatencyMs) DESC;
END;
```

### FileTable Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeFileTables
AS
BEGIN
    -- Analyze FileTable usage patterns
    SELECT 
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        fg.name as FileGroupName,
        COUNT(*) as FileCount,
        SUM(DATALENGTH(file_stream)) / 1073741824.0 as TotalSizeGB,
        AVG(DATALENGTH(file_stream)) / 1048576.0 as AvgFileSizeMB,
        MAX(DATALENGTH(file_stream)) / 1048576.0 as MaxFileSizeMB,
        COUNT(
            CASE 
                WHEN is_directory = 1 
                THEN 1 
            END
        ) as DirectoryCount,
        MAX(last_write_time) as LastWrite,
        MIN(creation_time) as OldestFile,
        COUNT(
            CASE 
                WHEN DATEDIFF(
                    DAY,
                    last_write_time,
                    GETDATE()
                ) > 90 THEN 1 
            END
        ) as OldFilesCount,
        CASE 
            WHEN COUNT(*) > 100000 
            THEN 'High File Count'
            WHEN SUM(
                DATALENGTH(file_stream)
            ) / 1073741824.0 > 100 
            THEN 'Large Total Size'
            WHEN MAX(
                DATALENGTH(file_stream)
            ) / 1048576.0 > 1024 
            THEN 'Large Individual Files'
            ELSE 'Normal'
        END as TableStatus,
        CASE 
            WHEN COUNT(*) > 100000 
            THEN 'Consider partitioning or archiving'
            WHEN SUM(
                DATALENGTH(file_stream)
            ) / 1073741824.0 > 100 
            THEN 'Review storage allocation'
            WHEN MAX(
                DATALENGTH(file_stream)
            ) / 1048576.0 > 1024 
            THEN 'Analyze large file usage'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.tables t
    JOIN sys.filegroups fg 
        ON t.filestream_data_space_id = fg.data_space_id
    JOIN sys.filetable_system_defined_objects ft 
        ON t.object_id = ft.object_id
    WHERE t.is_filetable = 1
    GROUP BY 
        t.object_id,
        t.name,
        fg.name
    HAVING COUNT(*) > 10000
    OR SUM(DATALENGTH(file_stream)) / 1073741824.0 > 100
    OR MAX(DATALENGTH(file_stream)) / 1048576.0 > 1024
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 100000 THEN 1
            WHEN SUM(
                DATALENGTH(file_stream)
            ) / 1073741824.0 > 100 THEN 2
            ELSE 3
        END,
        FileCount DESC;
END;
```

### FILESTREAM Access Pattern Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeFileStreamAccess
AS
BEGIN
    -- Analyze FILESTREAM access patterns
    SELECT 
        s.session_id,
        s.login_name,
        s.host_name,
        DB_NAME(r.database_id) as DatabaseName,
        OBJECT_NAME(fs.object_id) as TableName,
        fs.handle_count,
        fs.read_bytes_total / 1048576.0 as ReadMB,
        fs.write_bytes_total / 1048576.0 as WriteMB,
        fs.read_latency_ms / 
            NULLIF(fs.read_count, 0) as AvgReadLatencyMs,
        fs.write_latency_ms / 
            NULLIF(fs.write_count, 0) as AvgWriteLatencyMs,
        r.wait_type,
        r.wait_time,
        qt.text as LastQuery,
        CASE 
            WHEN fs.handle_count > 100 
            THEN 'High Handle Count'
            WHEN fs.read_latency_ms / 
                 NULLIF(fs.read_count, 0) > 100 
            THEN 'High Read Latency'
            WHEN fs.write_latency_ms / 
                 NULLIF(fs.write_count, 0) > 100 
            THEN 'High Write Latency'
            ELSE 'Normal'
        END as AccessPattern,
        CASE 
            WHEN fs.handle_count > 100 
            THEN 'Review handle management'
            WHEN fs.read_latency_ms / 
                 NULLIF(fs.read_count, 0) > 100 
            THEN 'Investigate read performance'
            WHEN fs.write_latency_ms / 
                 NULLIF(fs.write_count, 0) > 100 
            THEN 'Check write performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_filestream_file_io_handles fs
    JOIN sys.dm_exec_sessions s 
        ON fs.requester_id = s.session_id
    LEFT JOIN sys.dm_exec_requests r 
        ON s.session_id = r.session_id
    OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) qt
    WHERE fs.handle_count > 0
    AND (
        fs.handle_count > 100
        OR fs.read_latency_ms / NULLIF(fs.read_count, 0) > 100
        OR fs.write_latency_ms / NULLIF(fs.write_count, 0) > 100
    )
    ORDER BY 
        CASE 
            WHEN fs.handle_count > 100 THEN 1
            WHEN fs.read_latency_ms / 
                 NULLIF(fs.read_count, 0) > 100 THEN 2
            ELSE 3
        END,
        fs.handle_count DESC;
END;
```

This FILESTREAM analysis framework provides comprehensive tools for:
1. Monitoring FILESTREAM I/O performance and space usage
2. Analyzing FileTable patterns and growth
3. Tracking access patterns and handle usage
4. Optimizing FILESTREAM performance

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
