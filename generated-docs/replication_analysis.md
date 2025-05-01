# SQL Server Replication Analysis Framework

## Replication Monitoring Framework

### Distribution Agent Analysis
```sql
CREATE TABLE dbo.ReplicationAgentMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    PublicationDB sysname,
    PublicationName sysname,
    SubscriberDB sysname,
    SubscriberHost nvarchar(256),
    AgentName nvarchar(100),
    Status int,
    StatusMessage nvarchar(max),
    LastSyncTime datetime2,
    DeliveryLatency int,  -- seconds
    UndeliveredCommands int,
    DeliveryRate decimal(10,2),  -- commands/sec
    ErrorCount int,
    WarningCount int,
    ReplicationMode nvarchar(20),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorReplicationAgents
    @LatencyThresholdSeconds int = 300,
    @BacklogThreshold int = 10000
AS
BEGIN
    -- Capture replication metrics
    INSERT INTO dbo.ReplicationAgentMetrics
    SELECT 
        p.publisher_db as PublicationDB,
        p.publication as PublicationName,
        s.subscriber_db as SubscriberDB,
        s.subscriber_host as SubscriberHost,
        da.name as AgentName,
        da.status,
        da.comments as StatusMessage,
        da.time as LastSyncTime,
        DATEDIFF(SECOND, 
            da.last_distsync, 
            GETDATE()
        ) as DeliveryLatency,
        uncmd.UndeliveredCommands,
        CASE 
            WHEN DATEDIFF(SECOND, 
                da.last_distsync, 
                da.time) = 0 THEN 0
            ELSE da.delivered_commands * 1.0 / 
                DATEDIFF(SECOND, 
                    da.last_distsync, 
                    da.time)
        END as DeliveryRate,
        da.error_id as ErrorCount,
        da.alert as WarningCount,
        CASE da.subscription_type
            WHEN 0 THEN 'Push'
            WHEN 1 THEN 'Pull'
            WHEN 2 THEN 'Anonymous'
            ELSE 'Unknown'
        END as ReplicationMode,
        GETUTCDATE()
    FROM distribution.dbo.MSdistribution_agents da
    JOIN distribution.dbo.MSpublications p 
        ON da.publisher_id = p.publisher_id
    JOIN distribution.dbo.MSsubscriptions s 
        ON da.id = s.agent_id
    CROSS APPLY (
        SELECT COUNT(*) as UndeliveredCommands
        FROM distribution.dbo.MSrepl_commands c
        WHERE c.article_id IN (
            SELECT article_id
            FROM distribution.dbo.MSarticles
            WHERE publication_id = p.publication_id
        )
    ) as uncmd;

    -- Analyze replication health
    WITH ReplicationHealth AS (
        SELECT 
            PublicationDB,
            PublicationName,
            SubscriberDB,
            SubscriberHost,
            AgentName,
            DeliveryLatency,
            UndeliveredCommands,
            DeliveryRate,
            ErrorCount,
            WarningCount,
            ReplicationMode,
            LAG(UndeliveredCommands) OVER (
                PARTITION BY PublicationDB, 
                             PublicationName,
                             SubscriberDB 
                ORDER BY CollectionTime
            ) as PreviousBacklog
        FROM dbo.ReplicationAgentMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        PublicationDB,
        PublicationName,
        SubscriberDB,
        SubscriberHost,
        AgentName,
        DeliveryLatency,
        UndeliveredCommands,
        DeliveryRate,
        ErrorCount,
        WarningCount,
        CASE 
            WHEN ErrorCount > 0 THEN 'Error'
            WHEN DeliveryLatency > @LatencyThresholdSeconds 
            THEN 'High Latency'
            WHEN UndeliveredCommands > @BacklogThreshold 
            THEN 'High Backlog'
            WHEN UndeliveredCommands > PreviousBacklog 
            THEN 'Growing Backlog'
            ELSE 'Healthy'
        END as ReplicationStatus,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Review error log and agent status'
            WHEN DeliveryLatency > @LatencyThresholdSeconds 
            THEN 'Check network connectivity and agent performance'
            WHEN UndeliveredCommands > @BacklogThreshold 
            THEN 'Consider increasing agent resources'
            WHEN UndeliveredCommands > PreviousBacklog 
            THEN 'Monitor backlog growth rate'
            ELSE 'No action needed'
        END as Recommendation
    FROM ReplicationHealth
    WHERE ErrorCount > 0
    OR DeliveryLatency > @LatencyThresholdSeconds
    OR UndeliveredCommands > @BacklogThreshold
    OR UndeliveredCommands > PreviousBacklog
    ORDER BY 
        CASE 
            WHEN ErrorCount > 0 THEN 1
            WHEN DeliveryLatency > @LatencyThresholdSeconds THEN 2
            WHEN UndeliveredCommands > @BacklogThreshold THEN 3
            ELSE 4
        END,
        DeliveryLatency DESC;
END;
```

### Article Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeArticlePerformance
AS
BEGIN
    -- Analyze article-level metrics
    SELECT 
        p.publisher_db,
        p.publication,
        a.article,
        a.destination_object,
        s.subscriber_db,
        COUNT(*) as CommandCount,
        AVG(c.xact_seqno) as AvgTransactionSequence,
        SUM(DATALENGTH(c.command)) / 1048576.0 as TotalCommandSizeMB,
        AVG(DATALENGTH(c.command)) as AvgCommandSize,
        MAX(DATALENGTH(c.command)) as MaxCommandSize,
        COUNT(DISTINCT c.xact_seqno) as TransactionCount,
        CASE 
            WHEN AVG(DATALENGTH(c.command)) > 1048576 
            THEN 'Large Commands'
            WHEN COUNT(*) > 10000 
            THEN 'High Volume'
            ELSE 'Normal'
        END as ArticlePattern,
        CASE 
            WHEN AVG(DATALENGTH(c.command)) > 1048576 
            THEN 'Consider batch size adjustment'
            WHEN COUNT(*) > 10000 
            THEN 'Review indexing strategy'
            ELSE 'No action needed'
        END as Recommendation
    FROM distribution.dbo.MSrepl_commands c
    JOIN distribution.dbo.MSarticles a 
        ON c.article_id = a.article_id
    JOIN distribution.dbo.MSpublications p 
        ON a.publication_id = p.publication_id
    JOIN distribution.dbo.MSsubscriptions s 
        ON a.article_id = s.article_id
    WHERE c.entry_time >= DATEADD(HOUR, -1, GETUTCDATE())
    GROUP BY 
        p.publisher_db,
        p.publication,
        a.article,
        a.destination_object,
        s.subscriber_db
    HAVING COUNT(*) > 1000
    OR AVG(DATALENGTH(c.command)) > 102400
    ORDER BY COUNT(*) DESC;
END;
```

### Replication Conflict Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeReplicationConflicts
AS
BEGIN
    -- Analyze conflict patterns
    SELECT 
        p.publisher_db,
        p.publication,
        a.article,
        c.conflict_table,
        COUNT(*) as ConflictCount,
        MIN(c.create_time) as FirstConflict,
        MAX(c.create_time) as LastConflict,
        STRING_AGG(c.conflict_type, ', ') as ConflictTypes,
        COUNT(DISTINCT c.origin_datasource) as ConflictingSources,
        CASE 
            WHEN COUNT(*) > 100 THEN 'High Conflict Rate'
            WHEN COUNT(DISTINCT c.origin_datasource) > 2 
            THEN 'Multiple Sources'
            ELSE 'Normal'
        END as ConflictPattern,
        CASE 
            WHEN COUNT(*) > 100 
            THEN 'Review conflict resolution policy'
            WHEN COUNT(DISTINCT c.origin_datasource) > 2 
            THEN 'Analyze source patterns'
            ELSE 'Monitor for trends'
        END as Recommendation
    FROM distribution.dbo.MSmerge_conflicts c
    JOIN distribution.dbo.MSarticles a 
        ON c.article_id = a.article_id
    JOIN distribution.dbo.MSpublications p 
        ON a.publication_id = p.publication_id
    WHERE c.create_time >= DATEADD(DAY, -7, GETUTCDATE())
    GROUP BY 
        p.publisher_db,
        p.publication,
        a.article,
        c.conflict_table
    HAVING COUNT(*) > 10
    ORDER BY ConflictCount DESC;
END;
```

### Replication Token Tracking
```sql
CREATE PROCEDURE dbo.TrackReplicationTokens
    @LatencyThresholdSeconds int = 300
AS
BEGIN
    -- Analyze token latency
    SELECT 
        t.publisher_database_id,
        p.publisher_db,
        p.publication,
        s.subscriber_db,
        t.publication_id,
        t.tracer_id,
        t.insert_time,
        MIN(t.start_time) as TokenStart,
        MAX(t.end_time) as TokenEnd,
        DATEDIFF(
            SECOND, 
            MIN(t.start_time), 
            MAX(t.end_time)
        ) as TokenLatencySeconds,
        COUNT(*) as HopCount,
        CASE 
            WHEN DATEDIFF(
                SECOND, 
                MIN(t.start_time), 
                MAX(t.end_time)
            ) > @LatencyThresholdSeconds 
            THEN 'High Latency'
            WHEN COUNT(*) > 10 
            THEN 'Many Hops'
            ELSE 'Normal'
        END as TokenStatus,
        CASE 
            WHEN DATEDIFF(
                SECOND, 
                MIN(t.start_time), 
                MAX(t.end_time)
            ) > @LatencyThresholdSeconds 
            THEN 'Investigate replication bottlenecks'
            WHEN COUNT(*) > 10 
            THEN 'Review topology'
            ELSE 'No action needed'
        END as Recommendation
    FROM distribution.dbo.MStracer_tokens t
    JOIN distribution.dbo.MSpublications p 
        ON t.publication_id = p.publication_id
    JOIN distribution.dbo.MSsubscriptions s 
        ON p.publication_id = s.publication_id
    WHERE t.insert_time >= DATEADD(HOUR, -1, GETUTCDATE())
    GROUP BY 
        t.publisher_database_id,
        p.publisher_db,
        p.publication,
        s.subscriber_db,
        t.publication_id,
        t.tracer_id,
        t.insert_time
    HAVING DATEDIFF(
        SECOND, 
        MIN(t.start_time), 
        MAX(t.end_time)
    ) > @LatencyThresholdSeconds
    OR COUNT(*) > 10
    ORDER BY TokenLatencySeconds DESC;
END;
```

This replication analysis framework provides comprehensive tools for:
1. Monitoring distribution agent health and performance
2. Analyzing article-level replication metrics
3. Tracking and resolving replication conflicts
4. Measuring replication latency with tracer tokens

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
