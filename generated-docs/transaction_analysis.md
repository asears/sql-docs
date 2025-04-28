# SQL Server Transaction Analysis Framework

## Transaction Monitoring Framework

### Active Transaction Analysis
```sql
CREATE TABLE dbo.TransactionHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    SessionId int,
    TransactionId bigint,
    TransactionName nvarchar(128),
    TransactionType nvarchar(60),
    IsolationLevel nvarchar(60),
    Status nvarchar(60),
    DurationMs bigint,
    LogUsageKB bigint,
    TemporalTableUsage bit,
    OpenResultSets int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorActiveTransactions
    @LongRunningThresholdMs int = 10000
AS
BEGIN
    -- Capture current transactions
    INSERT INTO dbo.TransactionHistory
    SELECT 
        s.session_id,
        t.transaction_id,
        t.name,
        CASE t.transaction_type
            WHEN 1 THEN 'Read/Write'
            WHEN 2 THEN 'Read-Only'
            WHEN 3 THEN 'System'
            WHEN 4 THEN 'Distributed'
            ELSE 'Unknown'
        END as TransactionType,
        CASE s.transaction_isolation_level
            WHEN 0 THEN 'Unspecified'
            WHEN 1 THEN 'Read Uncommitted'
            WHEN 2 THEN 'Read Committed'
            WHEN 3 THEN 'Repeatable Read'
            WHEN 4 THEN 'Serializable'
            WHEN 5 THEN 'Snapshot'
            ELSE 'Unknown'
        END as IsolationLevel,
        t.transaction_state,
        DATEDIFF(MILLISECOND, t.transaction_begin_time, GETDATE()),
        t.database_transaction_log_bytes_used / 1024,
        CAST(CASE WHEN t.database_transaction_log_bytes_used > 0 
             AND EXISTS (
                 SELECT 1 
                 FROM sys.temporal_tables tt
                 JOIN sys.dm_tran_database_transactions dt
                     ON t.transaction_id = dt.transaction_id
                 WHERE tt.temporal_type = 2
             ) THEN 1 ELSE 0 END as bit),
        (SELECT COUNT(*) 
         FROM sys.dm_exec_cursors(s.session_id)),
        GETUTCDATE()
    FROM sys.dm_tran_active_transactions t
    JOIN sys.dm_tran_session_transactions st 
        ON t.transaction_id = st.transaction_id
    JOIN sys.dm_exec_sessions s 
        ON st.session_id = s.session_id
    WHERE s.session_id > 50;  -- Exclude system sessions

    -- Analyze transaction patterns
    WITH TransactionMetrics AS (
        SELECT 
            TransactionType,
            IsolationLevel,
            COUNT(*) as TransactionCount,
            AVG(DurationMs) as AvgDurationMs,
            MAX(DurationMs) as MaxDurationMs,
            SUM(LogUsageKB) as TotalLogUsageKB,
            SUM(CASE WHEN TemporalTableUsage = 1 THEN 1 ELSE 0 END) 
                as TemporalTableTxCount
        FROM dbo.TransactionHistory
        WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        GROUP BY TransactionType, IsolationLevel
    )
    SELECT 
        TransactionType,
        IsolationLevel,
        TransactionCount,
        AvgDurationMs / 1000.0 as AvgDurationSec,
        MaxDurationMs / 1000.0 as MaxDurationSec,
        TotalLogUsageKB / 1024.0 as TotalLogUsageMB,
        TemporalTableTxCount,
        CASE 
            WHEN AvgDurationMs > @LongRunningThresholdMs THEN 'Long Running'
            WHEN TotalLogUsageKB > 102400 THEN 'High Log Usage'
            ELSE 'Normal'
        END as TransactionStatus,
        CASE 
            WHEN AvgDurationMs > @LongRunningThresholdMs 
            THEN 'Review transaction duration and blocking'
            WHEN TotalLogUsageKB > 102400 
            THEN 'Monitor log space and consider checkpoints'
            ELSE 'No action needed'
        END as Recommendation
    FROM TransactionMetrics
    ORDER BY TransactionCount DESC;
END;
```

### Transaction Lock Analysis
```sql
CREATE TABLE dbo.TransactionLockHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    TransactionId bigint,
    ResourceType nvarchar(60),
    ResourceDescription nvarchar(max),
    LockMode nvarchar(60),
    LockDurationMs bigint,
    WaitResource nvarchar(256),
    BlockingSessionId int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeTransactionLocks
    @LockTimeoutThresholdMs int = 5000
AS
BEGIN
    -- Capture current locks
    INSERT INTO dbo.TransactionLockHistory
    SELECT 
        tst.transaction_id,
        tl.resource_type,
        tl.resource_description,
        tl.request_mode,
        r.wait_time,
        r.wait_resource,
        r.blocking_session_id,
        GETUTCDATE()
    FROM sys.dm_tran_locks tl
    JOIN sys.dm_tran_session_transactions tst 
        ON tl.request_session_id = tst.session_id
    LEFT JOIN sys.dm_exec_requests r 
        ON tst.session_id = r.session_id
    WHERE tst.session_id > 50;

    -- Analyze lock patterns
    WITH LockMetrics AS (
        SELECT 
            ResourceType,
            LockMode,
            COUNT(*) as LockCount,
            COUNT(DISTINCT TransactionId) as TransactionCount,
            AVG(LockDurationMs) as AvgDurationMs,
            MAX(LockDurationMs) as MaxDurationMs,
            COUNT(DISTINCT BlockingSessionId) as BlockingSessionCount
        FROM dbo.TransactionLockHistory
        WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        GROUP BY ResourceType, LockMode
    )
    SELECT 
        ResourceType,
        LockMode,
        LockCount,
        TransactionCount,
        AvgDurationMs / 1000.0 as AvgDurationSec,
        MaxDurationMs / 1000.0 as MaxDurationSec,
        BlockingSessionCount,
        CASE 
            WHEN BlockingSessionCount > 0 THEN 'Blocking Detected'
            WHEN AvgDurationMs > @LockTimeoutThresholdMs 
            THEN 'Long Duration'
            ELSE 'Normal'
        END as LockStatus,
        CASE 
            WHEN BlockingSessionCount > 0 
            THEN 'Review blocking chains and deadlocks'
            WHEN AvgDurationMs > @LockTimeoutThresholdMs 
            THEN 'Investigate lock duration'
            ELSE 'No action needed'
        END as Recommendation
    FROM LockMetrics
    ORDER BY 
        BlockingSessionCount DESC,
        LockCount DESC;
END;
```

### Transaction Log Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeTransactionLog
AS
BEGIN
    -- Analyze log usage
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        (active_log_size_mb + log_truncation_holdup_mb) as TotalLogUsageMB,
        active_transaction_count,
        total_log_size_mb,
        log_space_in_bytes_since_last_backup / 1048576.0 
            as LogGrowthSinceBackupMB,
        log_truncation_holdup_reason,
        CASE 
            WHEN log_space_in_bytes_since_last_backup > 
                 5368709120 -- 5GB
            THEN 'High Log Growth'
            WHEN active_transaction_count > 100 
            THEN 'High Transaction Count'
            ELSE 'Normal'
        END as LogStatus,
        CASE 
            WHEN log_space_in_bytes_since_last_backup > 
                 5368709120 
            THEN 'Consider log backup'
            WHEN active_transaction_count > 100 
            THEN 'Review active transactions'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_db_log_stats(DB_ID());

    -- Analyze VLF count
    DECLARE @VLFInfo TABLE (
        RecoveryUnitId int,
        FileId int,
        FileSize bigint,
        StartOffset bigint,
        FSeqNo int,
        [Status] int,
        Parity int,
        CreateLSN numeric(25,0)
    );

    INSERT INTO @VLFInfo
    EXEC sp_executesql N'DBCC LOGINFO WITH NO_INFOMSGS';

    SELECT 
        COUNT(*) as VLFCount,
        SUM(CASE WHEN [Status] = 2 THEN 1 ELSE 0 END) as ActiveVLFs,
        MIN(FileSize) / 1024.0 as MinVLFSizeKB,
        MAX(FileSize) / 1024.0 as MaxVLFSizeKB,
        AVG(FileSize) / 1024.0 as AvgVLFSizeKB,
        CASE 
            WHEN COUNT(*) > 1000 THEN 'High VLF Count'
            WHEN COUNT(*) > 500 THEN 'Moderate VLF Count'
            ELSE 'Normal'
        END as VLFStatus,
        CASE 
            WHEN COUNT(*) > 1000 
            THEN 'Consider log file rebuild'
            WHEN COUNT(*) > 500 
            THEN 'Monitor VLF growth'
            ELSE 'No action needed'
        END as Recommendation
    FROM @VLFInfo;
END;
```

### Transaction Rollback Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeTransactionRollbacks
AS
BEGIN
    -- Analyze active rollbacks
    SELECT 
        r.session_id,
        r.command,
        r.percent_complete,
        r.estimated_completion_time / 1000.0 as EstimatedSecondsRemaining,
        r.transaction_id,
        t.name as TransactionName,
        t.transaction_begin_time,
        DATEDIFF(SECOND, t.transaction_begin_time, GETDATE()) 
            as TransactionDurationSec,
        t.transaction_type,
        t.transaction_state,
        DB_NAME(r.database_id) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qt.text as CommandText,
        p.query_plan,
        CASE 
            WHEN r.estimated_completion_time > 300000 -- 5 minutes
            THEN 'Long Rollback'
            ELSE 'Normal'
        END as RollbackStatus,
        CASE 
            WHEN r.estimated_completion_time > 300000 
            THEN 'Monitor rollback progress'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_requests r
    JOIN sys.dm_tran_active_transactions t 
        ON r.transaction_id = t.transaction_id
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
    WHERE r.command LIKE '%ROLLBACK%'
    ORDER BY r.estimated_completion_time DESC;

    -- Analyze rollback history
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        transaction_id,
        operation,
        operation_desc,
        total_elapsed_time_ms / 1000.0 as TotalElapsedTimeSec,
        logical_reads,
        physical_reads,
        row_count,
        error_number,
        error_message
    FROM sys.dm_tran_database_transactions
    WHERE database_transaction_state = 3  -- Rollback
    ORDER BY total_elapsed_time_ms DESC;
END;
```

This transaction analysis framework provides comprehensive tools for:
1. Active transaction monitoring with duration and resource usage tracking
2. Lock analysis with blocking detection
3. Transaction log monitoring including VLF analysis
4. Rollback monitoring and impact assessment

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
