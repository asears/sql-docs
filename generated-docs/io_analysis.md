# SQL Server I/O Analysis Framework

## Storage Performance Analysis

### File I/O Monitoring
```sql
CREATE TABLE dbo.FileIOHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    FileName sysname,
    FileType varchar(10),
    ReadLatencyMs decimal(18,2),
    WriteLatencyMs decimal(18,2),
    IOPSRead int,
    IOPSWrite int,
    MBPerSecRead decimal(18,2),
    MBPerSecWrite decimal(18,2),
    AvgIOSize decimal(18,2),
    QueueDepth int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorFileIO
    @SampleDurationSeconds int = 60,
    @WarningLatencyMs int = 20,
    @CriticalLatencyMs int = 50
AS
BEGIN
    -- Capture initial snapshot
    CREATE TABLE #InitialSnapshot (
        DatabaseId int,
        FileId int,
        NumReads bigint,
        NumWrites bigint,
        BytesRead bigint,
        BytesWritten bigint,
        IoStallReadMs bigint,
        IoStallWriteMs bigint,
        QueueLength bigint
    );

    INSERT INTO #InitialSnapshot
    SELECT 
        database_id,
        file_id,
        num_of_reads,
        num_of_writes,
        num_of_bytes_read,
        num_of_bytes_written,
        io_stall_read_ms,
        io_stall_write_ms,
        io_queue_length
    FROM sys.dm_io_virtual_file_stats(NULL, NULL);

    -- Wait for sample duration
    WAITFOR DELAY @SampleDurationSeconds;

    -- Capture and analyze delta
    WITH CurrentSnapshot AS (
        SELECT 
            database_id,
            file_id,
            num_of_reads,
            num_of_writes,
            num_of_bytes_read,
            num_of_bytes_written,
            io_stall_read_ms,
            io_stall_write_ms,
            io_queue_length
        FROM sys.dm_io_virtual_file_stats(NULL, NULL)
    )
    INSERT INTO dbo.FileIOHistory
    SELECT 
        DB_NAME(cs.database_id),
        mf.name,
        mf.type_desc,
        CASE 
            WHEN (cs.num_of_reads - i.NumReads) = 0 THEN 0
            ELSE (cs.io_stall_read_ms - i.IoStallReadMs) * 1.0 / 
                 (cs.num_of_reads - i.NumReads)
        END,
        CASE 
            WHEN (cs.num_of_writes - i.NumWrites) = 0 THEN 0
            ELSE (cs.io_stall_write_ms - i.IoStallWriteMs) * 1.0 / 
                 (cs.num_of_writes - i.NumWrites)
        END,
        (cs.num_of_reads - i.NumReads) / @SampleDurationSeconds,
        (cs.num_of_writes - i.NumWrites) / @SampleDurationSeconds,
        (cs.num_of_bytes_read - i.BytesRead) / 1048576.0 / @SampleDurationSeconds,
        (cs.num_of_bytes_written - i.BytesWritten) / 1048576.0 / @SampleDurationSeconds,
        CASE 
            WHEN (cs.num_of_reads - i.NumReads + 
                  cs.num_of_writes - i.NumWrites) = 0 THEN 0
            ELSE ((cs.num_of_bytes_read - i.BytesRead + 
                   cs.num_of_bytes_written - i.BytesWritten) * 1.0) /
                 (cs.num_of_reads - i.NumReads + 
                  cs.num_of_writes - i.NumWrites)
        END,
        cs.io_queue_length,
        GETUTCDATE()
    FROM CurrentSnapshot cs
    JOIN #InitialSnapshot i 
        ON cs.database_id = i.DatabaseId 
        AND cs.file_id = i.FileId
    JOIN sys.master_files mf 
        ON cs.database_id = mf.database_id 
        AND cs.file_id = mf.file_id;

    -- Analyze I/O patterns
    WITH FileMetrics AS (
        SELECT 
            DatabaseName,
            FileName,
            FileType,
            ReadLatencyMs,
            WriteLatencyMs,
            IOPSRead + IOPSWrite as TotalIOPS,
            MBPerSecRead + MBPerSecWrite as TotalMBPerSec,
            QueueDepth,
            CASE 
                WHEN ReadLatencyMs > @CriticalLatencyMs OR 
                     WriteLatencyMs > @CriticalLatencyMs 
                THEN 'Critical'
                WHEN ReadLatencyMs > @WarningLatencyMs OR 
                     WriteLatencyMs > @WarningLatencyMs 
                THEN 'Warning'
                ELSE 'Normal'
            END as LatencyStatus
        FROM dbo.FileIOHistory
        WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        FileName,
        FileType,
        ReadLatencyMs,
        WriteLatencyMs,
        TotalIOPS,
        TotalMBPerSec,
        QueueDepth,
        LatencyStatus,
        CASE 
            WHEN LatencyStatus = 'Critical' 
            THEN 'Immediate attention required - Consider:
                  1. Moving files to faster storage
                  2. Reviewing index fragmentation
                  3. Analyzing query patterns'
            WHEN LatencyStatus = 'Warning' 
            THEN 'Monitor closely - Consider:
                  1. Scheduling maintenance
                  2. Reviewing file layout'
            ELSE 'No action needed'
        END as Recommendation
    FROM FileMetrics
    ORDER BY 
        CASE LatencyStatus
            WHEN 'Critical' THEN 1
            WHEN 'Warning' THEN 2
            ELSE 3
        END,
        TotalIOPS DESC;
END;
```

### Page Split Analysis
```sql
CREATE TABLE dbo.PageSplitHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseId int,
    ObjectId int,
    IndexId int,
    PageSplits int,
    PageDeallocations int,
    PageAllocations int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.TrackPageSplits
AS
BEGIN
    -- Capture current page split metrics
    INSERT INTO dbo.PageSplitHistory
    SELECT 
        ps.database_id,
        ps.object_id,
        ps.index_id,
        SUM(leaf_splits) as page_splits,
        SUM(page_deallocations) as page_deallocations,
        SUM(page_allocations) as page_allocations,
        GETUTCDATE()
    FROM (
        SELECT 
            database_id,
            object_id,
            index_id,
            leaf_allocation_count as page_allocations,
            leaf_page_merge_count as page_deallocations,
            leaf_split_count as leaf_splits
        FROM sys.dm_db_index_operational_stats(
            DB_ID(), NULL, NULL, NULL)
    ) ps
    GROUP BY 
        ps.database_id,
        ps.object_id,
        ps.index_id;

    -- Analyze split patterns
    WITH SplitMetrics AS (
        SELECT 
            DB_NAME(DatabaseId) as DatabaseName,
            OBJECT_NAME(ObjectId) as TableName,
            i.name as IndexName,
            PageSplits,
            PageDeallocations,
            PageAllocations,
            ROW_NUMBER() OVER (
                PARTITION BY DatabaseId, ObjectId, IndexId 
                ORDER BY CollectionTime DESC
            ) as rn
        FROM dbo.PageSplitHistory h
        JOIN sys.indexes i 
            ON h.ObjectId = i.object_id 
            AND h.IndexId = i.index_id
    )
    SELECT 
        DatabaseName,
        TableName,
        IndexName,
        PageSplits,
        PageDeallocations,
        PageAllocations,
        CASE 
            WHEN PageSplits > 1000 THEN 'High Split Rate'
            WHEN PageSplits > 100 THEN 'Moderate Split Rate'
            ELSE 'Normal Split Rate'
        END as SplitStatus,
        CASE 
            WHEN PageSplits > 1000 
            THEN 'Consider:
                  1. Rebuilding index with higher fill factor
                  2. Implementing page split monitoring
                  3. Reviewing insert patterns'
            WHEN PageSplits > 100 
            THEN 'Monitor split rate trend'
            ELSE 'No action needed'
        END as Recommendation
    FROM SplitMetrics
    WHERE rn = 1
    AND (PageSplits > 0 OR PageDeallocations > 0)
    ORDER BY PageSplits DESC;
END;
```

### Buffer Cache Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeBufferCache
AS
BEGIN
    -- Analyze buffer cache contents
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(p.object_id) as TableName,
        i.name as IndexName,
        COUNT(*) as BufferCount,
        COUNT(*) * 8 / 1024.0 as BufferMB,
        SUM(CAST(is_modified AS int)) as DirtyPages,
        SUM(CAST(is_modified AS int)) * 100.0 / 
            COUNT(*) as DirtyPercent,
        AVG(read_microsec) / 1000.0 as AvgReadLatencyMs
    FROM sys.dm_os_buffer_descriptors b
    JOIN sys.allocation_units a 
        ON b.allocation_unit_id = a.allocation_unit_id
    JOIN sys.partitions p 
        ON a.container_id = p.hobt_id
    LEFT JOIN sys.indexes i 
        ON p.object_id = i.object_id 
        AND p.index_id = i.index_id
    WHERE b.database_id = DB_ID()
    AND p.object_id > 100
    GROUP BY 
        database_id,
        p.object_id,
        i.name
    HAVING COUNT(*) > 1000
    ORDER BY BufferCount DESC;

    -- Analyze buffer cache efficiency
    SELECT 
        counter_name,
        cntr_value
    FROM sys.dm_os_performance_counters
    WHERE object_name LIKE '%Buffer Manager%'
    AND counter_name IN (
        'Buffer cache hit ratio',
        'Page life expectancy',
        'Free list stalls/sec',
        'Lazy writes/sec'
    );
END;
```

### I/O Wait Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeIOWaits
AS
BEGIN
    -- Analyze I/O-related wait stats
    WITH IOWaits AS (
        SELECT 
            wait_type,
            waiting_tasks_count,
            wait_time_ms,
            max_wait_time_ms,
            signal_wait_time_ms
        FROM sys.dm_os_wait_stats
        WHERE wait_type LIKE 'PAGEIOLATCH_%'
        OR wait_type LIKE 'IO_COMPLETION'
        OR wait_type LIKE 'WRITE_COMPLETION'
        OR wait_type LIKE 'ASYNC_IO_COMPLETION'
    )
    SELECT 
        wait_type,
        waiting_tasks_count as WaitCount,
        wait_time_ms * 1.0 / 
            waiting_tasks_count as AvgWaitTimeMs,
        max_wait_time_ms as MaxWaitTimeMs,
        signal_wait_time_ms * 100.0 / 
            wait_time_ms as SignalPercent,
        CASE 
            WHEN wait_time_ms * 1.0 / 
                 waiting_tasks_count > 100 
            THEN 'Critical'
            WHEN wait_time_ms * 1.0 / 
                 waiting_tasks_count > 30 
            THEN 'Warning'
            ELSE 'Normal'
        END as WaitStatus,
        CASE 
            WHEN wait_time_ms * 1.0 / 
                 waiting_tasks_count > 100 
            THEN 'Investigate storage subsystem and query patterns'
            WHEN wait_time_ms * 1.0 / 
                 waiting_tasks_count > 30 
            THEN 'Monitor wait times trend'
            ELSE 'No action needed'
        END as Recommendation
    FROM IOWaits
    WHERE waiting_tasks_count > 0
    ORDER BY wait_time_ms DESC;

    -- Analyze currently waiting tasks
    SELECT 
        r.session_id,
        r.wait_type,
        r.wait_time / 1000.0 as WaitTimeSec,
        r.blocking_session_id,
        s.program_name,
        DB_NAME(r.database_id) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qt.text as QueryText,
        p.query_plan
    FROM sys.dm_exec_requests r
    JOIN sys.dm_exec_sessions s 
        ON r.session_id = s.session_id
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
    WHERE r.wait_type LIKE 'PAGEIOLATCH_%'
    OR r.wait_type LIKE 'IO_COMPLETION'
    OR r.wait_type LIKE 'WRITE_COMPLETION'
    OR r.wait_type LIKE 'ASYNC_IO_COMPLETION'
    ORDER BY r.wait_time DESC;
END;
```

This I/O analysis framework provides comprehensive tools for:
1. Detailed file I/O monitoring and latency analysis
2. Page split tracking and optimization
3. Buffer cache content and efficiency analysis
4. I/O wait statistics and pattern detection

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
