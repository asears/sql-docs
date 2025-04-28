# SQL Server Network Performance Analysis Framework

## Network Connection Analysis

### Connection Tracking
```sql
CREATE TABLE dbo.NetworkConnectionHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    SessionId int,
    ClientNetAddress varchar(48),
    ClientTcpPort int,
    AuthScheme nvarchar(40),
    ProtocolType nvarchar(40),
    EncryptionOption nvarchar(40),
    NetPacketSize int,
    ClientInterfaceLibrary nvarchar(32),
    ConnectionDurationMs bigint,
    LastRequestStartTime datetime2,
    LastRequestEndTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.TrackNetworkConnections
    @HighLatencyThresholdMs int = 100
AS
BEGIN
    -- Capture current connections
    INSERT INTO dbo.NetworkConnectionHistory
    SELECT 
        c.session_id,
        c.client_net_address,
        c.client_tcp_port,
        c.auth_scheme,
        c.protocol_type,
        c.encrypt_option,
        c.net_packet_size,
        c.client_interface_name,
        c.connect_time_ms,
        c.last_request_start_time,
        c.last_request_end_time,
        GETUTCDATE()
    FROM sys.dm_exec_connections c
    WHERE c.session_id > 50;  -- Exclude system sessions

    -- Analyze connection patterns
    WITH ConnectionMetrics AS (
        SELECT 
            ClientNetAddress,
            COUNT(*) as ConnectionCount,
            AVG(ConnectionDurationMs) as AvgDurationMs,
            MAX(ConnectionDurationMs) as MaxDurationMs,
            AVG(NetPacketSize) as AvgPacketSize,
            STRING_AGG(CAST(ClientInterfaceLibrary as varchar(100)), ', ') 
                as ClientLibraries
        FROM dbo.NetworkConnectionHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY ClientNetAddress
    )
    SELECT 
        ClientNetAddress,
        ConnectionCount,
        AvgDurationMs / 1000.0 as AvgDurationSec,
        MaxDurationMs / 1000.0 as MaxDurationSec,
        AvgPacketSize,
        ClientLibraries,
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs THEN 'High Latency'
            WHEN ConnectionCount > 100 THEN 'High Connection Count'
            ELSE 'Normal'
        END as ConnectionStatus,
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
            THEN 'Review network latency and routing'
            WHEN ConnectionCount > 100 
            THEN 'Consider connection pooling optimization'
            ELSE 'No action needed'
        END as Recommendation
    FROM ConnectionMetrics
    ORDER BY 
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs THEN 1
            WHEN ConnectionCount > 100 THEN 2
            ELSE 3
        END,
        ConnectionCount DESC;
END;
```

### Network Wait Analysis
```sql
CREATE TABLE dbo.NetworkWaitHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    WaitType nvarchar(60),
    WaitingTasksCount bigint,
    WaitTimeMs bigint,
    MaxWaitTimeMs bigint,
    SignalWaitTimeMs bigint,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeNetworkWaits
    @HighWaitThresholdMs int = 1000
AS
BEGIN
    -- Capture network wait statistics
    INSERT INTO dbo.NetworkWaitHistory
    SELECT 
        wait_type,
        waiting_tasks_count,
        wait_time_ms,
        max_wait_time_ms,
        signal_wait_time_ms,
        GETUTCDATE()
    FROM sys.dm_os_wait_stats
    WHERE wait_type LIKE 'NETWORK%'
    OR wait_type LIKE 'ASYNC_NETWORK%';

    -- Analyze wait patterns
    WITH WaitMetrics AS (
        SELECT 
            WaitType,
            WaitingTasksCount,
            WaitTimeMs,
            MaxWaitTimeMs,
            SignalWaitTimeMs,
            LAG(WaitTimeMs) OVER (
                PARTITION BY WaitType 
                ORDER BY CollectionTime
            ) as PreviousWaitTime
        FROM dbo.NetworkWaitHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        WaitType,
        WaitingTasksCount,
        WaitTimeMs / 1000.0 as WaitTimeSec,
        MaxWaitTimeMs / 1000.0 as MaxWaitTimeSec,
        SignalWaitTimeMs * 100.0 / 
            NULLIF(WaitTimeMs, 0) as SignalWaitPercent,
        CASE 
            WHEN WaitTimeMs > @HighWaitThresholdMs THEN 'Critical'
            WHEN WaitTimeMs > @HighWaitThresholdMs / 2 THEN 'Warning'
            ELSE 'Normal'
        END as WaitStatus,
        CASE 
            WHEN WaitTimeMs > @HighWaitThresholdMs 
            THEN 'Consider:
                  1. Network configuration review
                  2. Client connection settings
                  3. Packet size optimization'
            WHEN WaitTimeMs > @HighWaitThresholdMs / 2 
            THEN 'Monitor for increase'
            ELSE 'No action needed'
        END as Recommendation
    FROM WaitMetrics
    WHERE WaitTimeMs > 0
    ORDER BY WaitTimeMs DESC;
END;
```

### Network Throughput Analysis
```sql
CREATE TABLE dbo.NetworkThroughputHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    ReadThroughputMBSec decimal(18,2),
    WriteThroughputMBSec decimal(18,2),
    TotalConnections int,
    AvgResponseTimeMs decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorNetworkThroughput
    @SampleDurationSeconds int = 60
AS
BEGIN
    -- Create temporary tables for snapshots
    CREATE TABLE #InitialSnapshot (
        session_id int,
        reads bigint,
        writes bigint,
        logical_reads bigint
    );

    -- Capture initial state
    INSERT INTO #InitialSnapshot
    SELECT 
        session_id,
        reads,
        writes,
        logical_reads
    FROM sys.dm_exec_sessions
    WHERE session_id > 50;

    -- Wait for sample duration
    WAITFOR DELAY @SampleDurationSeconds;

    -- Calculate throughput
    WITH CurrentMetrics AS (
        SELECT 
            s.session_id,
            s.reads - ISNULL(i.reads, 0) as reads_delta,
            s.writes - ISNULL(i.writes, 0) as writes_delta,
            s.logical_reads - ISNULL(i.logical_reads, 0) as logical_reads_delta
        FROM sys.dm_exec_sessions s
        LEFT JOIN #InitialSnapshot i 
            ON s.session_id = i.session_id
        WHERE s.session_id > 50
    )
    INSERT INTO dbo.NetworkThroughputHistory
    SELECT 
        DB_NAME(r.database_id),
        SUM(cm.reads_delta) * 8.0 / 
            (1024 * @SampleDurationSeconds) as ReadThroughputMBSec,
        SUM(cm.writes_delta) * 8.0 / 
            (1024 * @SampleDurationSeconds) as WriteThroughputMBSec,
        COUNT(DISTINCT cm.session_id) as TotalConnections,
        AVG(DATEDIFF(MILLISECOND, 
            r.start_time, 
            GETDATE())) as AvgResponseTimeMs,
        GETUTCDATE()
    FROM CurrentMetrics cm
    JOIN sys.dm_exec_requests r 
        ON cm.session_id = r.session_id
    GROUP BY DB_NAME(r.database_id);

    -- Analyze throughput patterns
    SELECT 
        DatabaseName,
        ReadThroughputMBSec,
        WriteThroughputMBSec,
        TotalConnections,
        AvgResponseTimeMs,
        CASE 
            WHEN AvgResponseTimeMs > 1000 THEN 'High Latency'
            WHEN ReadThroughputMBSec + WriteThroughputMBSec > 100 
            THEN 'High Throughput'
            ELSE 'Normal'
        END as ThroughputStatus,
        CASE 
            WHEN AvgResponseTimeMs > 1000 
            THEN 'Review network latency and query patterns'
            WHEN ReadThroughputMBSec + WriteThroughputMBSec > 100 
            THEN 'Monitor network capacity'
            ELSE 'No action needed'
        END as Recommendation
    FROM dbo.NetworkThroughputHistory
    WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
    ORDER BY ReadThroughputMBSec + WriteThroughputMBSec DESC;
END;
```

### Connection Pool Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeConnectionPools
AS
BEGIN
    -- Analyze connection pooling efficiency
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        COUNT(*) as ActiveConnections,
        SUM(CASE 
            WHEN connection_id IS NOT NULL 
            THEN 1 ELSE 0 
        END) as PooledConnections,
        SUM(CASE 
            WHEN connection_id IS NULL 
            THEN 1 ELSE 0 
        END) as NonPooledConnections,
        AVG(CAST(
            connect_time_ms as decimal(18,2)
        )) as AvgConnectTimeMs,
        MAX(connect_time_ms) as MaxConnectTimeMs,
        MIN(connect_time_ms) as MinConnectTimeMs,
        CASE 
            WHEN COUNT(*) > 1000 THEN 'High Connection Count'
            WHEN AVG(connect_time_ms) > 100 THEN 'High Connect Time'
            ELSE 'Normal'
        END as PoolStatus
    FROM sys.dm_exec_connections c
    JOIN sys.dm_exec_sessions s 
        ON c.session_id = s.session_id
    WHERE s.session_id > 50
    GROUP BY database_id
    ORDER BY ActiveConnections DESC;

    -- Analyze connection reuse
    SELECT 
        ec.client_net_address,
        ec.auth_scheme,
        ec.protocol_type,
        COUNT(*) as ConnectionCount,
        AVG(DATEDIFF(
            MILLISECOND, 
            ec.connect_time, 
            GETDATE()
        )) as AvgConnectionAgeMs,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 0 
            THEN 1 ELSE 0 
        END) as UnspecifiedIsolationCount,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 1 
            THEN 1 ELSE 0 
        END) as ReadUncommittedCount,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 2 
            THEN 1 ELSE 0 
        END) as ReadCommittedCount,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 3 
            THEN 1 ELSE 0 
        END) as RepeatableReadCount,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 4 
            THEN 1 ELSE 0 
        END) as SerializableCount,
        SUM(CASE 
            WHEN es.transaction_isolation_level = 5 
            THEN 1 ELSE 0 
        END) as SnapshotCount
    FROM sys.dm_exec_connections ec
    JOIN sys.dm_exec_sessions es 
        ON ec.session_id = es.session_id
    WHERE es.session_id > 50
    GROUP BY 
        ec.client_net_address,
        ec.auth_scheme,
        ec.protocol_type
    HAVING COUNT(*) > 10
    ORDER BY ConnectionCount DESC;
END;
```

This network analysis framework provides comprehensive tools for:
1. Connection tracking and pattern analysis
2. Network wait statistics monitoring
3. Throughput measurement and optimization
4. Connection pool efficiency analysis

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
