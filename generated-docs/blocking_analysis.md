# SQL Server Blocking and Deadlock Analysis Framework

## Blocking Chain Analysis

### Real-Time Block Monitoring
```sql
CREATE TABLE dbo.BlockingHistory
(
    BlockingId bigint IDENTITY(1,1) PRIMARY KEY,
    BlockingSessionId int,
    BlockedSessionId int,
    BlockingResource nvarchar(100),
    WaitType nvarchar(100),
    WaitDuration bigint,  -- milliseconds
    BlockingSQL nvarchar(max),
    BlockedSQL nvarchar(max),
    BlockingApplicationName nvarchar(128),
    BlockedApplicationName nvarchar(128),
    DetectionTime datetime2,
    ResolutionTime datetime2,
    ResolutionType varchar(20)  -- Timeout, Killed, Completed
);

CREATE PROCEDURE dbo.CollectBlockingChains
    @MinDurationSeconds int = 10
AS
BEGIN
    WITH BlockingTree AS (
        -- Find head blockers (no blockers themselves)
        SELECT 
            r.session_id as blocking_session_id,
            r.blocking_session_id as blocked_by_id,
            CAST(r.session_id as varchar(1000)) as chain,
            0 as chain_level,
            r.wait_time as wait_duration,
            r.wait_type,
            OBJECT_NAME(p.object_id) as blocking_resource,
            s.program_name as application_name,
            t.text as sql_text
        FROM sys.dm_exec_requests r
        JOIN sys.dm_exec_sessions s 
            ON r.session_id = s.session_id
        LEFT JOIN sys.partitions p 
            ON r.wait_resource LIKE 'OBJECT: ' + 
               CAST(p.hobt_id as varchar(100)) + '%'
        OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
        WHERE r.blocking_session_id = 0
        AND EXISTS (
            SELECT * 
            FROM sys.dm_exec_requests r2
            WHERE r2.blocking_session_id = r.session_id
        )

        UNION ALL

        -- Get blocked sessions
        SELECT 
            r.session_id,
            r.blocking_session_id,
            CAST(bt.chain + ' -> ' + 
                 CAST(r.session_id as varchar(10)) as varchar(1000)),
            bt.chain_level + 1,
            r.wait_time,
            r.wait_type,
            OBJECT_NAME(p.object_id),
            s.program_name,
            t.text
        FROM sys.dm_exec_requests r
        JOIN BlockingTree bt 
            ON r.blocking_session_id = bt.blocking_session_id
        JOIN sys.dm_exec_sessions s 
            ON r.session_id = s.session_id
        LEFT JOIN sys.partitions p 
            ON r.wait_resource LIKE 'OBJECT: ' + 
               CAST(p.hobt_id as varchar(100)) + '%'
        OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
        WHERE r.blocking_session_id > 0
    )
    INSERT INTO dbo.BlockingHistory
    (
        BlockingSessionId, BlockedSessionId, 
        BlockingResource, WaitType,
        WaitDuration, BlockingSQL, BlockedSQL,
        BlockingApplicationName, BlockedApplicationName,
        DetectionTime
    )
    SELECT 
        blocking_session_id,
        blocked_by_id,
        blocking_resource,
        wait_type,
        wait_duration,
        MAX(CASE WHEN chain_level = 0 THEN sql_text END),
        MAX(CASE WHEN chain_level > 0 THEN sql_text END),
        MAX(CASE WHEN chain_level = 0 THEN application_name END),
        MAX(CASE WHEN chain_level > 0 THEN application_name END),
        GETUTCDATE()
    FROM BlockingTree
    WHERE wait_duration > @MinDurationSeconds * 1000
    GROUP BY 
        blocking_session_id,
        blocked_by_id,
        blocking_resource,
        wait_type,
        wait_duration;
END;
```

### Deadlock Analysis
```sql
CREATE TABLE dbo.DeadlockEvents
(
    DeadlockId bigint IDENTITY(1,1) PRIMARY KEY,
    DeadlockTime datetime2,
    VictimSessionId int,
    VictimSQL nvarchar(max),
    VictimApplicationName nvarchar(128),
    BlockerSessionId int,
    BlockerSQL nvarchar(max),
    BlockerApplicationName nvarchar(128),
    DeadlockGraph xml,
    ResolutionType varchar(20),
    PreventiveMeasure nvarchar(max)
);

CREATE PROCEDURE dbo.AnalyzeDeadlocks
AS
BEGIN
    -- Collect deadlock graph from system_health
    INSERT INTO dbo.DeadlockEvents
    (
        DeadlockTime, VictimSessionId, VictimSQL,
        VictimApplicationName, BlockerSessionId,
        BlockerSQL, BlockerApplicationName,
        DeadlockGraph, ResolutionType
    )
    SELECT
        DATEADD(mi, 
            DATEDIFF(mi, GETUTCDATE(), GETDATE()), 
            event_data.value('(event/@timestamp)[1]', 'datetime2')) as DeadlockTime,
        event_data.value('(//deadlock/process-list/process/@spid)[1]', 'int') as VictimSessionId,
        event_data.value('(//deadlock/process-list/process/inputbuf)[1]', 'nvarchar(max)') as VictimSQL,
        event_data.value('(//deadlock/process-list/process/@clientapp)[1]', 'nvarchar(128)') as VictimApplicationName,
        event_data.value('(//deadlock/process-list/process/@spid)[2]', 'int') as BlockerSessionId,
        event_data.value('(//deadlock/process-list/process/inputbuf)[2]', 'nvarchar(max)') as BlockerSQL,
        event_data.value('(//deadlock/process-list/process/@clientapp)[2]', 'nvarchar(128)') as BlockerApplicationName,
        event_data as DeadlockGraph,
        'Victim Chosen' as ResolutionType
    FROM 
    (
        SELECT 
            CAST(target_data as xml) as TargetData
        FROM sys.dm_xe_session_targets t
        JOIN sys.dm_xe_sessions s 
            ON t.event_session_id = s.session_id
        WHERE s.name = 'system_health'
        AND t.target_name = 'ring_buffer'
    ) as Data
    CROSS APPLY TargetData.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') as XEventData(event_data)
    WHERE NOT EXISTS (
        SELECT 1 
        FROM dbo.DeadlockEvents de
        WHERE de.DeadlockGraph = event_data
    );

    -- Analyze patterns and suggest preventive measures
    UPDATE de
    SET PreventiveMeasure = 
        CASE 
            WHEN VictimSQL LIKE '%UPDATE%' 
                 AND BlockerSQL LIKE '%UPDATE%'
                 AND VictimSQL LIKE '%' + 
                     SUBSTRING(BlockerSQL, 
                        CHARINDEX('UPDATE', BlockerSQL), 
                        CHARINDEX(' ', BlockerSQL, 
                            CHARINDEX('UPDATE', BlockerSQL)) - 
                        CHARINDEX('UPDATE', BlockerSQL)
                     ) + '%'
            THEN 'Consider adding UPDLOCK hint or implementing optimistic concurrency'
            
            WHEN VictimSQL LIKE '%SELECT%' 
                 AND BlockerSQL LIKE '%UPDATE%'
            THEN 'Consider using READPAST hint or implementing RCSI'
            
            WHEN DeadlockGraph.exist('//deadlock/resource-list/pagelock')=1
            THEN 'Consider reviewing index design and reducing page contention'
            
            WHEN DeadlockGraph.exist('//deadlock/resource-list/keylock')=1
            THEN 'Consider implementing row versioning or adjusting transaction isolation level'
            
            ELSE 'Manual analysis required'
        END
    FROM dbo.DeadlockEvents de
    WHERE PreventiveMeasure IS NULL;
END;
```

### Lock Escalation Prevention
```sql
CREATE PROCEDURE dbo.MonitorLockEscalation
AS
BEGIN
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(object_id) as TableName,
        resource_type,
        request_mode,
        request_status,
        request_session_id,
        COUNT(*) as LockCount
    FROM sys.dm_tran_locks
    WHERE resource_type IN ('PAGE', 'KEY', 'RID', 'OBJECT')
    GROUP BY 
        database_id, object_id, resource_type,
        request_mode, request_status, request_session_id
    HAVING COUNT(*) > 5000;  -- Threshold for potential escalation

    -- Monitor lock wait times
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(object_id) as TableName,
        request_mode,
        wait_time_ms,
        blocking_session_id
    FROM sys.dm_tran_locks l
    JOIN sys.dm_os_waiting_tasks w 
        ON l.lock_owner_address = w.resource_address
    WHERE wait_time_ms > 1000;  -- 1 second threshold
END;

CREATE PROCEDURE dbo.PreventLockEscalation
    @SchemaName sysname,
    @TableName sysname
AS
BEGIN
    -- Disable lock escalation
    DECLARE @SQL nvarchar(max) = '
    ALTER TABLE ' + QUOTENAME(@SchemaName) + '.' + 
        QUOTENAME(@TableName) + '
    SET (LOCK_ESCALATION = DISABLE)';
    
    EXEC sp_executesql @SQL;

    -- Create monitoring trigger
    SET @SQL = '
    CREATE TRIGGER tr_' + @TableName + '_LockMonitor
    ON ' + QUOTENAME(@SchemaName) + '.' + 
        QUOTENAME(@TableName) + '
    AFTER UPDATE, DELETE
    AS
    BEGIN
        SET NOCOUNT ON;
        
        DECLARE @LockCount int;
        
        SELECT @LockCount = COUNT(*)
        FROM sys.dm_tran_locks
        WHERE resource_type IN (''PAGE'', ''KEY'', ''RID'')
        AND request_session_id = @@SPID;
        
        IF @LockCount > 5000
        BEGIN
            ROLLBACK TRANSACTION;
            RAISERROR(''Lock escalation threshold exceeded'', 16, 1);
            RETURN;
        END;
    END;';
    
    EXEC sp_executesql @SQL;
END;
```

### Extended Events Analysis
```sql
-- Create Extended Events session for blocking analysis
CREATE EVENT SESSION [BlockingAnalysis] ON SERVER 
ADD EVENT sqlserver.blocked_process_report(
    ACTION (
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
),
ADD EVENT sqlserver.lock_deadlock_chain(
    ACTION (
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
),
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(
    SET filename=N'C:\Traces\BlockingAnalysis.xel',
        max_file_size=(100),
        max_rollover_files=10
)
WITH (
    MAX_MEMORY=4096 KB,
    EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY=30 SECONDS,
    MAX_EVENT_SIZE=0 KB,
    MEMORY_PARTITION_MODE=NONE,
    TRACK_CAUSALITY=ON,
    STARTUP_STATE=ON
);
GO

-- Create analysis procedure
CREATE PROCEDURE dbo.AnalyzeBlockingPatterns
    @StartTime datetime2,
    @EndTime datetime2
AS
BEGIN
    -- Read XE file
    WITH BlockingData AS (
        SELECT 
            event_data.value('(event/@timestamp)[1]', 'datetime2') as EventTime,
            event_data.value('(event/action[@name="client_app_name"]/value)[1]', 'nvarchar(128)') as ApplicationName,
            event_data.value('(event/action[@name="database_name"]/value)[1]', 'nvarchar(128)') as DatabaseName,
            event_data.value('(event/action[@name="sql_text"]/value)[1]', 'nvarchar(max)') as SQLText,
            event_data.value('(event/data[@name="duration"]/value)[1]', 'bigint') as Duration,
            event_data.value('(event/data[@name="lock_mode"]/text)[1]', 'nvarchar(50)') as LockMode,
            event_data.value('(event/data[@name="object_id"]/value)[1]', 'int') as ObjectId
        FROM sys.fn_xe_file_target_read_file(
            'C:\Traces\BlockingAnalysis*.xel', 
            NULL, NULL, NULL)
        CROSS APPLY (SELECT CAST(event_data as xml)) as event_data_xml(event_data)
        WHERE event_data.value('(event/@timestamp)[1]', 'datetime2') 
            BETWEEN @StartTime AND @EndTime
    )
    SELECT 
        DatabaseName,
        OBJECT_NAME(ObjectId, DB_ID(DatabaseName)) as ObjectName,
        LockMode,
        COUNT(*) as BlockingCount,
        AVG(Duration) as AvgDuration,
        MAX(Duration) as MaxDuration,
        STRING_AGG(CAST(SQLText as nvarchar(max)), CHAR(13)) as SQLPatterns
    FROM BlockingData
    GROUP BY 
        DatabaseName,
        ObjectId,
        LockMode
    ORDER BY BlockingCount DESC;
END;
```

### Performance Impact Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeBlockingImpact
    @StartTime datetime2,
    @EndTime datetime2
AS
BEGIN
    -- Calculate blocking impact metrics
    WITH BlockingMetrics AS (
        SELECT 
            BlockingSessionId,
            COUNT(DISTINCT BlockedSessionId) as BlockedSessions,
            AVG(WaitDuration) as AvgWaitTime,
            MAX(WaitDuration) as MaxWaitTime,
            SUM(WaitDuration) as TotalWaitTime,
            STRING_AGG(BlockingResource, ', ') as BlockedResources
        FROM dbo.BlockingHistory
        WHERE DetectionTime BETWEEN @StartTime AND @EndTime
        GROUP BY BlockingSessionId
    )
    SELECT 
        bm.*,
        CASE 
            WHEN MaxWaitTime > 30000 THEN 'Critical'  -- Over 30 seconds
            WHEN MaxWaitTime > 5000 THEN 'Warning'   -- Over 5 seconds
            ELSE 'Normal'
        END as Severity,
        CASE 
            WHEN BlockedSessions > 10 THEN 'Chain Blocking'
            WHEN TotalWaitTime > 60000 THEN 'Long Duration'
            ELSE 'Standard'
        END as BlockingPattern
    FROM BlockingMetrics bm
    ORDER BY TotalWaitTime DESC;

    -- Analyze temporal patterns
    SELECT 
        DATEPART(HOUR, DetectionTime) as HourOfDay,
        DATEPART(WEEKDAY, DetectionTime) as DayOfWeek,
        COUNT(*) as BlockingCount,
        AVG(WaitDuration) as AvgWaitDuration,
        MAX(WaitDuration) as MaxWaitDuration
    FROM dbo.BlockingHistory
    WHERE DetectionTime BETWEEN @StartTime AND @EndTime
    GROUP BY 
        DATEPART(HOUR, DetectionTime),
        DATEPART(WEEKDAY, DetectionTime)
    ORDER BY BlockingCount DESC;

    -- Resource contention analysis
    SELECT 
        BlockingResource,
        COUNT(*) as ContentionCount,
        AVG(WaitDuration) as AvgContentionTime,
        MAX(WaitDuration) as MaxContentionTime,
        STRING_AGG(CAST(BlockingSQL as nvarchar(max)), CHAR(13)) as SQLPatterns
    FROM dbo.BlockingHistory
    WHERE DetectionTime BETWEEN @StartTime AND @EndTime
    GROUP BY BlockingResource
    HAVING COUNT(*) > 5
    ORDER BY ContentionCount DESC;
END;
```

This comprehensive blocking analysis framework provides detailed tools for monitoring, analyzing, and preventing blocking and deadlocks in SQL Server. The framework includes:

1. Real-time block monitoring with chain analysis
2. Deadlock detection and pattern analysis
3. Lock escalation prevention mechanisms
4. Extended Events integration for detailed analysis
5. Impact analysis and reporting capabilities

Would you like me to add more specific components for any of these areas or focus on another aspect of performance troubleshooting?
