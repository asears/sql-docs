# SQL Server Performance Optimization Guide

## Performance Monitoring and Troubleshooting

### Query Performance
1. Query Store
   - Automatic query performance monitoring
   - Regression detection
   - Forced plan management
   - Historical performance data analysis

2. Dynamic Management Views (DMVs)
   - sys.dm_exec_query_stats
   - sys.dm_exec_procedure_stats
   - sys.dm_exec_sessions
   - sys.dm_os_wait_stats

### Resource Optimization

#### Memory Configuration
- Max server memory setting
- Buffer pool optimization
- Plan cache management
- Memory-optimized tables
- Columnstore memory tuning

#### I/O Optimization
1. File Configuration
   - Separate data and log files
   - Multiple tempdb files
   - File group strategies
   - Proper RAID configuration

2. Storage Performance
   - Use enterprise-grade SSDs
   - Enable instant file initialization
   - Optimize NTFS allocation unit size
   - Monitor disk latency and queue length

#### Network Configuration
- SQL Server network packet size
- Network protocol optimization
- Compress network traffic
- Monitor network latency

### Large Database Optimization

#### Partitioning Strategies
1. Table Partitioning
   - Sliding window implementation
   - Archive old data automatically
   - Partition-aligned indexes
   - Partition elimination

2. File Group Management
```sql
-- Create new filegroup
ALTER DATABASE YourDB ADD FILEGROUP [FG_Archive_2024]
GO

-- Add file to filegroup
ALTER DATABASE YourDB ADD FILE 
(NAME = N'Archive2024', FILENAME = N'path\archive2024.ndf')
TO FILEGROUP [FG_Archive_2024]
```

#### Online Operations
1. Online Index Rebuilds
```sql
ALTER INDEX ALL ON LargeTable
REBUILD WITH (ONLINE = ON)
```

2. Moving Tables Between Filegroups
```sql
CREATE UNIQUE CLUSTERED INDEX [CI_TableName] 
ON [TableName] (KeyColumn)
WITH (DROP_EXISTING = ON) 
ON [NewFileGroup]
```

### Performance Best Practices

#### Index Optimization
1. Missing Index Detection
2. Index Usage Statistics
3. Index Maintenance Schedule
4. Filtered Indexes for Specific Queries

#### Statistics Management
- Auto Update Statistics
- Sampling Rates
- Histogram Updates
- Filtered Statistics

#### Resource Governor
- Workload Group Configuration
- Resource Pool Management
- IO Resource Management
- External Resource Pools

### Very Large Database (VLDB) Management

#### Scaling Strategies
1. Horizontal Partitioning
   - Distribute data across filegroups
   - Implement partition schemes
   - Manage sliding windows

2. Archival Solutions
   - Table partitioning
   - Stretch Database
   - Data compression
   - Columnstore indexes

#### Zero-Downtime Operations
1. Online Schema Changes
2. Parallel Operations
3. Minimal Logging
4. Batched Updates

### Performance Monitoring Tools

#### Built-in Tools
- Performance Monitor
- SQL Server Profiler
- Database Engine Tuning Advisor
- System Dynamic Management Views

#### Extended Events
- Lightweight event tracking
- Custom event sessions
- Performance tracking
- Wait statistics analysis

### Optimization Checklist

#### Daily Monitoring
- [ ] Check CPU utilization
- [ ] Monitor memory usage
- [ ] Review disk I/O patterns
- [ ] Analyze blocking and deadlocks
- [ ] Review long-running queries

#### Weekly Tasks
- [ ] Index maintenance
- [ ] Statistics updates
- [ ] Query plan analysis
- [ ] Resource utilization trends
- [ ] Backup performance review

#### Monthly Review
- [ ] Capacity planning
- [ ] Performance baseline comparison
- [ ] Storage growth analysis
- [ ] Query pattern changes
- [ ] Resource governor adjustments

## Performance Troubleshooting Guide

### Query Performance Investigation

#### Using SQL Server Profiler
1. Start Data Collection
   - Launch SQL Server Profiler
   - Create a new trace using the "Tuning" template
   - Add filters for specific databases/applications
   - Additional events to capture:
     * SP:StmtCompleted
     * SQL:BatchCompleted
     * Deadlock graph
     * Lock:Deadlock
     * Performance:Statistics

2. Analyzing Trace Results
```sql
-- Create trace table
SELECT * INTO #TraceResults
FROM ::fn_trace_gettable('C:\Traces\PerformanceTrace.trc', default)

-- Find most expensive queries
SELECT 
    TextData,
    CPU,
    Reads,
    Writes,
    Duration,
    COUNT(*) as ExecutionCount,
    AVG(Duration) as AvgDuration,
    MAX(Duration) as MaxDuration
FROM #TraceResults
GROUP BY TextData, CPU, Reads, Writes, Duration
ORDER BY Duration DESC
```

#### DMV Analysis
1. Query Store Investigation
```sql
-- Find regressed queries
SELECT 
    q.query_id,
    qt.query_sql_text,
    rs.count_executions,
    rs.avg_duration,
    rs.avg_cpu_time,
    rs.avg_logical_io_reads,
    p.compatibility_level
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt 
    ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p 
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs 
    ON p.plan_id = rs.plan_id
WHERE rs.avg_duration > 1000  -- 1 second
ORDER BY rs.avg_duration DESC

-- Analyze plan changes
SELECT 
    q.query_id,
    qt.query_sql_text,
    p.plan_id,
    p.last_execution_time,
    rs.avg_duration,
    cast(p.query_plan as xml) as plan_xml
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt 
    ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p 
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs 
    ON p.plan_id = rs.plan_id
WHERE q.query_id IN (
    SELECT TOP 10 query_id 
    FROM sys.query_store_runtime_stats
    ORDER BY avg_duration DESC
)
```

2. Wait Statistics Analysis
```sql
-- Clear wait stats for baseline
DBCC SQLPERF('sys.dm_os_wait_stats', CLEAR);

-- Analyze wait types
SELECT 
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms,
    CAST(100.0 * wait_time_ms / SUM(wait_time_ms) 
        OVER() as DECIMAL(5,2)) as wait_percent
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'BROKER_TASK_STOP',
    'SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT',
    'CLR_MANUAL_EVENT'
)
ORDER BY wait_time_ms DESC;

-- Analyze blocking chains
WITH BlockingHierarchy AS (
    SELECT 
        request_session_id as session_id,
        CAST(NULL as int) as blocking_session_id,
        CAST(request_session_id as varchar(4000)) as blocking_chain,
        0 as chain_level
    FROM sys.dm_exec_requests r
    WHERE blocking_session_id IS NULL
    AND request_session_id in (
        SELECT blocking_session_id 
        FROM sys.dm_exec_requests 
        WHERE blocking_session_id != 0
    )
    UNION ALL
    SELECT 
        r.request_session_id,
        r.blocking_session_id,
        CAST(bh.blocking_chain + ' -> ' + 
             CAST(r.request_session_id as varchar(4000)) as varchar(4000)),
        bh.chain_level + 1
    FROM sys.dm_exec_requests r
    INNER JOIN BlockingHierarchy bh 
    ON r.blocking_session_id = bh.session_id
    WHERE r.blocking_session_id != 0
)
SELECT 
    bh.blocking_chain,
    s.login_name,
    s.host_name,
    s.program_name,
    r.command,
    r.wait_type,
    r.wait_time,
    st.text as sql_text
FROM BlockingHierarchy bh
JOIN sys.dm_exec_sessions s 
    ON bh.session_id = s.session_id
LEFT JOIN sys.dm_exec_requests r 
    ON bh.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
ORDER BY chain_level;
```

#### Using Red Gate SQL Monitor

1. Performance Baseline Collection
   - Configure baseline periods
   - Set alert thresholds for:
     * CPU utilization > 80%
     * Memory pressure (Page Life Expectancy < 300)
     * Disk latency > 20ms
     * Blocking duration > 30 seconds
   - Enable custom metrics collection

2. Alert Response Procedures
   ```sql
   -- Create alert response logging
   CREATE TABLE dbo.AlertResponses (
       AlertId int IDENTITY(1,1),
       AlertType varchar(50),
       ServerName varchar(128),
       DatabaseName varchar(128),
       MetricValue decimal(18,2),
       ResponseAction varchar(max),
       ResponseTime datetime2
   )

   -- Log response actions
   INSERT INTO dbo.AlertResponses
   VALUES (
       'CPU Pressure',
       @@SERVERNAME,
       DB_NAME(),
       cpu_value,
       'Identified top CPU consumers using Query Store',
       GETUTCDATE()
   )
   ```

#### Using sp_Blitz Suite

1. Performance Analysis
```sql
-- Run comprehensive health check
EXEC sp_Blitz 
    @CheckServerInfo = 1,
    @CheckProcedureCache = 1,
    @OutputDatabaseName = 'DBA',
    @OutputSchemaName = 'dbo',
    @OutputTableName = 'BlitzResults';

-- Analyze specific performance issues
EXEC sp_BlitzCache 
    @SortOrder = 'CPU',
    @Top = 20,
    @ExportToExcel = 1;

-- Check index usage
EXEC sp_BlitzIndex 
    @DatabaseName = 'YourDB',
    @SchemaName = 'dbo',
    @TableName = 'LargeTable';
```

2. First Responder Kit Implementation
```sql
-- Create performance dashboard
CREATE TABLE dbo.PerformanceChecks (
    CheckId int IDENTITY(1,1),
    CheckType varchar(50),
    FindingsCount int,
    DetailsJSON nvarchar(max),
    CheckDate datetime2
)

-- Schedule regular checks
CREATE PROCEDURE dbo.RunPerformanceChecks
AS
BEGIN
    SET NOCOUNT ON;
    
    CREATE TABLE #BlitzResults (
        ID int,
        Priority int,
        FindingsGroup varchar(100),
        Finding varchar(200),
        Details nvarchar(max)
    );

    INSERT INTO #BlitzResults
    EXEC sp_Blitz;

    INSERT INTO dbo.PerformanceChecks
    SELECT 
        'Daily Health Check',
        COUNT(*),
        (SELECT * FROM #BlitzResults FOR JSON AUTO),
        GETUTCDATE()
    FROM #BlitzResults;
END;
```

#### Using Sysinternals Tools

1. Process Monitor Analysis
   - Filter for sqlservr.exe process
   - Monitor file system activity
   - Identify high I/O patterns
   - Track registry access
   - Command line setup:
   ```powershell
   # Download and extract Process Monitor
   Invoke-WebRequest -Uri "https://download.sysinternals.com/files/ProcessMonitor.zip" -OutFile "C:\Tools\procmon.zip"
   Expand-Archive -Path "C:\Tools\procmon.zip" -DestinationPath "C:\Tools\procmon"
   
   # Start monitoring SQL Server
   & "C:\Tools\procmon\procmon.exe" /AcceptEula /Quiet /Minimized /BackingFile "C:\Logs\sql_activity.pml"
   ```

2. Process Explorer Investigation
   - Monitor SQL Server memory usage
   - Track handle usage
   - Analyze thread activity
   - Implementation steps:
   ```powershell
   # Download Process Explorer
   Invoke-WebRequest -Uri "https://download.sysinternals.com/files/ProcessExplorer.zip" -OutFile "C:\Tools\procexp.zip"
   Expand-Archive -Path "C:\Tools\procexp.zip" -DestinationPath "C:\Tools\procexp"
   
   # Launch with specific SQL focus
   & "C:\Tools\procexp\procexp.exe" /AcceptEula /e /t "sqlservr"
   ```

### Mitigation Strategies

#### Memory Pressure Resolution

1. Buffer Pool Extension Optimization
```sql
-- Monitor BPE usage
SELECT 
    BPE_CURRENT_SIZE_MB,
    BPE_AVAILABLE_MB,
    FILE_NAME,
    STATE_DESCRIPTION
FROM sys.dm_os_buffer_pool_extension_configuration;

-- Adjust BPE size based on workload
ALTER SERVER CONFIGURATION
SET BUFFER POOL EXTENSION ON
(FILENAME = 'E:\BPE\bpe.bpe', SIZE = 102400 MB);

-- Monitor page life expectancy
CREATE TABLE dbo.PLEHistory (
    CaptureTime datetime2,
    PageLifeExpectancy int,
    MemoryGrantsPending int,
    TargetMemoryKB bigint,
    TotalMemoryKB bigint
);

-- Collect PLE metrics
INSERT INTO dbo.PLEHistory
SELECT 
    GETUTCDATE(),
    (SELECT cntr_value 
     FROM sys.dm_os_performance_counters 
     WHERE counter_name = 'Page life expectancy'),
    (SELECT cntr_value 
     FROM sys.dm_os_performance_counters 
     WHERE counter_name = 'Memory Grants Pending'),
    (SELECT physical_memory_in_use_kb 
     FROM sys.dm_os_process_memory),
    (SELECT total_physical_memory_kb 
     FROM sys.dm_os_sys_memory);
```

2. Memory Grant Feedback
```sql
-- Enable memory grant feedback
ALTER DATABASE SCOPED CONFIGURATION
SET MEMORY_GRANT_FEEDBACK = ON;

-- Monitor memory grants
SELECT 
    qt.query_sql_text,
    qp.query_plan,
    rs.avg_ideal_grant_kb,
    rs.avg_required_memory_kb,
    rs.avg_used_grant_kb,
    rs.max_used_grant_kb,
    rs.min_used_grant_kb,
    rs.count_executions
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan qp
    ON q.query_id = qp.query_id
JOIN sys.query_store_runtime_stats rs
    ON qp.plan_id = rs.plan_id
WHERE rs.avg_ideal_grant_kb > rs.avg_used_grant_kb * 2
ORDER BY rs.avg_ideal_grant_kb DESC;
```

#### I/O Optimization

1. IOPS Analysis
```sql
-- Monitor file I/O
SELECT 
    DB_NAME(mf.database_id) as database_name,
    mf.physical_name,
    divfs.num_of_reads,
    divfs.num_of_writes,
    divfs.io_stall_read_ms,
    divfs.io_stall_write_ms,
    CAST(100.0 * divfs.io_stall_read_ms /
        (divfs.io_stall_read_ms + divfs.io_stall_write_ms) 
        as DECIMAL(5,2)) as read_stall_pct,
    CAST(100.0 * divfs.io_stall_write_ms /
        (divfs.io_stall_read_ms + divfs.io_stall_write_ms) 
        as DECIMAL(5,2)) as write_stall_pct
FROM sys.dm_io_virtual_file_stats(NULL, NULL) divfs
JOIN sys.master_files mf
    ON divfs.database_id = mf.database_id
    AND divfs.file_id = mf.file_id
ORDER BY (divfs.io_stall_read_ms + divfs.io_stall_write_ms) DESC;

-- Create I/O baseline
CREATE TABLE dbo.IOBaseline (
    CaptureTime datetime2,
    DatabaseName sysname,
    FileName sysname,
    ReadLatencyMs decimal(10,2),
    WriteLatencyMs decimal(10,2),
    IOPSRead int,
    IOPSWrite int
);

-- Collect I/O metrics
INSERT INTO dbo.IOBaseline
SELECT 
    GETUTCDATE(),
    DB_NAME(mf.database_id),
    mf.name,
    CASE WHEN divfs.num_of_reads = 0 
         THEN 0 
         ELSE divfs.io_stall_read_ms * 1.0 / divfs.num_of_reads 
    END,
    CASE WHEN divfs.num_of_writes = 0 
         THEN 0 
         ELSE divfs.io_stall_write_ms * 1.0 / divfs.num_of_writes 
    END,
    divfs.num_of_reads,
    divfs.num_of_writes
FROM sys.dm_io_virtual_file_stats(NULL, NULL) divfs
JOIN sys.master_files mf
    ON divfs.database_id = mf.database_id
    AND divfs.file_id = mf.file_id;
```

2. Storage Configuration Optimization
```sql
-- Move tempdb files
ALTER DATABASE tempdb 
MODIFY FILE (NAME = 'tempdev', FILENAME = 'E:\MSSQL\Data\tempdb.mdf');

-- Add tempdb files
ALTER DATABASE tempdb 
ADD FILE (
    NAME = 'tempdev2',
    FILENAME = 'F:\MSSQL\Data\tempdb2.ndf',
    SIZE = 8192MB,
    FILEGROWTH = 1024MB
);

-- Monitor tempdb usage
SELECT 
    su.session_id,
    su.request_id,
    su.used_pages,
    t.text,
    p.query_plan
FROM tempdb.sys.allocation_units au
JOIN tempdb.sys.dm_db_session_space_usage su
    ON au.container_id = su.user_objects_alloc_page_count
JOIN sys.dm_exec_requests r
    ON su.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
ORDER BY su.used_pages DESC;
```

#### Network Configuration

1. Network Latency Analysis
```sql
-- Monitor network wait stats
SELECT 
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'NETWORK%'
ORDER BY wait_time_ms DESC;

-- Check network packet size
SELECT 
    client_net_address,
    client_tcp_port,
    net_packet_size,
    protocol_type
FROM sys.dm_exec_connections
WHERE session_id > 50;
```

2. Network Protocol Optimization
```sql
-- Create network monitoring
CREATE TABLE dbo.NetworkMetrics (
    CaptureTime datetime2,
    ConnectionCount int,
    AvgPacketSize int,
    NetworkWaitTimeMs bigint,
    NetworkIOWaitTimeMs bigint
);

-- Collect network metrics
INSERT INTO dbo.NetworkMetrics
SELECT 
    GETUTCDATE(),
    COUNT(*),
    AVG(net_packet_size),
    (SELECT wait_time_ms 
     FROM sys.dm_os_wait_stats 
     WHERE wait_type = 'NETWORK_IO'),
    (SELECT wait_time_ms 
     FROM sys.dm_os_wait_stats 
     WHERE wait_type = 'ASYNC_NETWORK_IO')
FROM sys.dm_exec_connections
WHERE session_id > 50;
```
