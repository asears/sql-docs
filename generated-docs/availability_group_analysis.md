# SQL Server Always On Availability Group Analysis Framework

## Availability Group Health Monitoring

### Replica Synchronization Analysis
```sql
CREATE TABLE dbo.AGSyncMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    GroupName sysname,
    ReplicaServer nvarchar(256),
    DatabaseName sysname,
    SyncState nvarchar(60),
    SyncHealth nvarchar(60),
    LogSendQueueSize bigint,
    LogSendRate decimal(18,2),
    RedoQueueSize bigint,
    RedoRate decimal(18,2),
    LastCommitTime datetime2,
    LastHardenedLSN numeric(25,0),
    LastRedoneTime datetime2,
    EstimatedDataLoss int,  -- seconds
    EstimatedRecoveryTime int,  -- seconds
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorAGSynchronization
    @QueueThresholdMB int = 1024,
    @SyncLatencyThresholdSec int = 10
AS
BEGIN
    -- Capture AG metrics
    INSERT INTO dbo.AGSyncMetrics
    SELECT 
        ag.name as GroupName,
        ar.replica_server_name as ReplicaServer,
        db.database_name as DatabaseName,
        drs.synchronization_state_desc as SyncState,
        drs.synchronization_health_desc as SyncHealth,
        drs.log_send_queue_size / 1024.0 as LogSendQueueSizeMB,
        drs.log_send_rate / 1024.0 as LogSendRateMB,
        drs.redo_queue_size / 1024.0 as RedoQueueSizeMB,
        drs.redo_rate / 1024.0 as RedoRateMB,
        drs.last_commit_time,
        drs.last_hardened_lsn,
        drs.last_redone_time,
        DATEDIFF(
            SECOND, 
            drs.last_commit_time, 
            GETDATE()
        ) as EstimatedDataLoss,
        CASE 
            WHEN drs.redo_rate = 0 THEN NULL
            ELSE (drs.redo_queue_size / 
                  NULLIF(drs.redo_rate, 0)
            )
        END as EstimatedRecoveryTime,
        GETUTCDATE()
    FROM sys.availability_groups ag
    JOIN sys.availability_replicas ar 
        ON ag.group_id = ar.group_id
    JOIN sys.dm_hadr_database_replica_states drs 
        ON ar.replica_id = drs.replica_id
    JOIN sys.databases db 
        ON drs.database_id = db.database_id;

    -- Analyze synchronization health
    WITH SyncHealth AS (
        SELECT 
            GroupName,
            ReplicaServer,
            DatabaseName,
            SyncState,
            SyncHealth,
            LogSendQueueSize,
            LogSendRate,
            RedoQueueSize,
            RedoRate,
            EstimatedDataLoss,
            EstimatedRecoveryTime,
            LAG(LogSendQueueSize) OVER (
                PARTITION BY GroupName, 
                             ReplicaServer, 
                             DatabaseName 
                ORDER BY CollectionTime
            ) as PreviousQueueSize
        FROM dbo.AGSyncMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        GroupName,
        ReplicaServer,
        DatabaseName,
        SyncState,
        SyncHealth,
        LogSendQueueSize,
        LogSendRate,
        RedoQueueSize,
        RedoRate,
        EstimatedDataLoss,
        EstimatedRecoveryTime,
        CASE 
            WHEN SyncHealth != 'HEALTHY' 
            THEN 'Critical'
            WHEN LogSendQueueSize > @QueueThresholdMB 
            THEN 'High Send Queue'
            WHEN RedoQueueSize > @QueueThresholdMB 
            THEN 'High Redo Queue'
            WHEN EstimatedDataLoss > @SyncLatencyThresholdSec 
            THEN 'High Latency'
            ELSE 'Healthy'
        END as ReplicaStatus,
        CASE 
            WHEN SyncHealth != 'HEALTHY' 
            THEN 'Investigate replica health issues'
            WHEN LogSendQueueSize > @QueueThresholdMB 
            THEN 'Check network bandwidth and latency'
            WHEN RedoQueueSize > @QueueThresholdMB 
            THEN 'Review secondary replica resources'
            WHEN EstimatedDataLoss > @SyncLatencyThresholdSec 
            THEN 'Monitor synchronization latency'
            ELSE 'No action needed'
        END as Recommendation
    FROM SyncHealth
    WHERE SyncHealth != 'HEALTHY'
    OR LogSendQueueSize > @QueueThresholdMB
    OR RedoQueueSize > @QueueThresholdMB
    OR EstimatedDataLoss > @SyncLatencyThresholdSec
    ORDER BY 
        CASE 
            WHEN SyncHealth != 'HEALTHY' THEN 1
            WHEN LogSendQueueSize > @QueueThresholdMB THEN 2
            WHEN RedoQueueSize > @QueueThresholdMB THEN 3
            ELSE 4
        END,
        EstimatedDataLoss DESC;
END;
```

### Failover Analysis
```sql
CREATE TABLE dbo.AGFailoverHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    GroupName sysname,
    PreviousPrimary nvarchar(256),
    NewPrimary nvarchar(256),
    FailoverType nvarchar(60),
    FailoverTime datetime2,
    TransitionDurationSec int,
    DataLossSec int,
    FailureReason nvarchar(max),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeFailovers
AS
BEGIN
    -- Analyze failover patterns
    WITH FailoverMetrics AS (
        SELECT 
            GroupName,
            PreviousPrimary,
            NewPrimary,
            FailoverType,
            FailoverTime,
            TransitionDurationSec,
            DataLossSec,
            ROW_NUMBER() OVER (
                PARTITION BY GroupName 
                ORDER BY FailoverTime DESC
            ) as rn,
            COUNT(*) OVER (
                PARTITION BY GroupName
            ) as FailoverCount,
            AVG(TransitionDurationSec) OVER (
                PARTITION BY GroupName
            ) as AvgTransitionDuration
        FROM dbo.AGFailoverHistory
        WHERE FailoverTime >= DATEADD(DAY, -30, GETUTCDATE())
    )
    SELECT 
        GroupName,
        PreviousPrimary,
        NewPrimary,
        FailoverType,
        FailoverTime,
        TransitionDurationSec,
        DataLossSec,
        FailoverCount,
        AvgTransitionDuration,
        CASE 
            WHEN FailoverCount > 5 
            THEN 'Frequent Failovers'
            WHEN DataLossSec > 0 
            THEN 'Data Loss Occurred'
            WHEN TransitionDurationSec > 60 
            THEN 'Slow Transition'
            ELSE 'Normal'
        END as FailoverPattern,
        CASE 
            WHEN FailoverCount > 5 
            THEN 'Investigate root cause of frequent failovers'
            WHEN DataLossSec > 0 
            THEN 'Review synchronization settings'
            WHEN TransitionDurationSec > 60 
            THEN 'Optimize failover process'
            ELSE 'No action needed'
        END as Recommendation
    FROM FailoverMetrics
    WHERE rn = 1
    AND (
        FailoverCount > 5
        OR DataLossSec > 0
        OR TransitionDurationSec > 60
    )
    ORDER BY 
        CASE 
            WHEN FailoverCount > 5 THEN 1
            WHEN DataLossSec > 0 THEN 2
            ELSE 3
        END,
        FailoverTime DESC;
END;
```

### Listener Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeListenerPerformance
AS
BEGIN
    -- Analyze listener connections
    SELECT 
        ag.name as GroupName,
        agl.dns_name as ListenerName,
        agl.port as ListenerPort,
        agl.ip_configuration_string_from_cluster as IPConfig,
        COUNT(c.session_id) as ActiveConnections,
        AVG(c.connect_time_ms) as AvgConnectTimeMs,
        MAX(c.connect_time_ms) as MaxConnectTimeMs,
        COUNT(DISTINCT c.client_net_address) as UniqueClients,
        SUM(CASE 
            WHEN c.connect_time_ms > 1000 
            THEN 1 ELSE 0 
        END) as SlowConnections,
        CASE 
            WHEN COUNT(CASE 
                WHEN c.connect_time_ms > 1000 
                THEN 1 
            END) > 10 
            THEN 'Connection Issues'
            WHEN COUNT(c.session_id) > 1000 
            THEN 'High Connection Count'
            ELSE 'Normal'
        END as ListenerStatus,
        CASE 
            WHEN COUNT(CASE 
                WHEN c.connect_time_ms > 1000 
                THEN 1 
            END) > 10 
            THEN 'Investigate network or DNS issues'
            WHEN COUNT(c.session_id) > 1000 
            THEN 'Review connection pooling settings'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.availability_groups ag
    JOIN sys.availability_group_listeners agl 
        ON ag.group_id = agl.group_id
    LEFT JOIN sys.dm_exec_connections c 
        ON agl.ip_address = c.local_net_address
    GROUP BY 
        ag.name,
        agl.dns_name,
        agl.port,
        agl.ip_configuration_string_from_cluster
    HAVING COUNT(CASE 
        WHEN c.connect_time_ms > 1000 
        THEN 1 
    END) > 10
    OR COUNT(c.session_id) > 1000
    ORDER BY 
        CASE 
            WHEN COUNT(CASE 
                WHEN c.connect_time_ms > 1000 
                THEN 1 
            END) > 10 THEN 1
            ELSE 2
        END,
        ActiveConnections DESC;
END;
```

### Resource Monitoring
```sql
CREATE PROCEDURE dbo.MonitorAGResources
AS
BEGIN
    -- Analyze resource usage across replicas
    SELECT 
        ag.name as GroupName,
        ar.replica_server_name as ReplicaServer,
        db.database_name as DatabaseName,
        dm.counter_name as MetricName,
        dm.cntr_value as CurrentValue,
        dm.cntr_type as CounterType,
        CASE 
            WHEN dm.counter_name LIKE '%Memory%'
            THEN dm.cntr_value / 1024.0 
            ELSE dm.cntr_value 
        END as FormattedValue,
        CASE 
            WHEN dm.counter_name LIKE '%Memory%' 
                 AND dm.cntr_value > 8589934592  -- 8GB
            THEN 'High Memory Usage'
            WHEN dm.counter_name LIKE '%CPU%' 
                 AND dm.cntr_value > 80 
            THEN 'High CPU Usage'
            WHEN dm.counter_name LIKE '%Disk%' 
                 AND dm.cntr_value > 100 
            THEN 'High Disk Activity'
            ELSE 'Normal'
        END as ResourceStatus,
        CASE 
            WHEN dm.counter_name LIKE '%Memory%' 
                 AND dm.cntr_value > 8589934592 
            THEN 'Review memory allocation'
            WHEN dm.counter_name LIKE '%CPU%' 
                 AND dm.cntr_value > 80 
            THEN 'Check CPU pressure'
            WHEN dm.counter_name LIKE '%Disk%' 
                 AND dm.cntr_value > 100 
            THEN 'Monitor disk performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.availability_groups ag
    JOIN sys.availability_replicas ar 
        ON ag.group_id = ar.group_id
    JOIN sys.dm_hadr_database_replica_states drs 
        ON ar.replica_id = drs.replica_id
    JOIN sys.databases db 
        ON drs.database_id = db.database_id
    CROSS APPLY (
        SELECT 
            counter_name,
            cntr_value,
            cntr_type
        FROM sys.dm_os_performance_counters
        WHERE object_name LIKE '%Database Replica%'
        AND instance_name = db.name
    ) dm
    WHERE dm.cntr_value > 0
    AND (
        (dm.counter_name LIKE '%Memory%' 
         AND dm.cntr_value > 8589934592)
        OR (dm.counter_name LIKE '%CPU%' 
            AND dm.cntr_value > 80)
        OR (dm.counter_name LIKE '%Disk%' 
            AND dm.cntr_value > 100)
    )
    ORDER BY 
        CASE dm.counter_name
            WHEN '%Memory%' THEN 1
            WHEN '%CPU%' THEN 2
            WHEN '%Disk%' THEN 3
            ELSE 4
        END,
        dm.cntr_value DESC;
END;
```

This Always On Availability Group analysis framework provides comprehensive tools for:
1. Monitoring replica synchronization health and performance
2. Analyzing failover patterns and impact
3. Tracking listener performance and connection issues
4. Monitoring resource usage across replicas

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
