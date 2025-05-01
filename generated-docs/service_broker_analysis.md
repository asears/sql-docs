# SQL Server Service Broker Analysis Framework

## Service Broker Monitoring Framework

### Queue Performance Analysis
```sql
CREATE TABLE dbo.ServiceBrokerMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    QueueName sysname,
    ServiceName sysname,
    MessageTypeCount int,
    MessageCount bigint,
    ActivationState nvarchar(60),
    ProcedureExecutions bigint,
    LastActivationTime datetime2,
    AvgProcessingTimeMs decimal(18,2),
    ErrorCount int,
    RetryCount int,
    StorageUsedKB bigint,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorServiceBroker
    @HighQueueThreshold int = 1000,
    @HighLatencyThresholdMs decimal(18,2) = 5000.0
AS
BEGIN
    -- Capture Service Broker metrics
    INSERT INTO dbo.ServiceBrokerMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(q.object_id) as SchemaName,
        q.name as QueueName,
        s.name as ServiceName,
        COUNT(DISTINCT mt.name) as MessageTypeCount,
        (
            SELECT COUNT(*) 
            FROM sys.transmission_queue tq
            WHERE tq.service_contract_id = sc.service_contract_id
        ) as MessageCount,
        q.activation_state_desc as ActivationState,
        (
            SELECT execution_count 
            FROM sys.dm_exec_procedure_stats ps
            WHERE ps.object_id = q.activation_procedure_id
        ) as ProcedureExecutions,
        MAX(tq.enqueue_time) as LastActivationTime,
        AVG(
            DATEDIFF(
                MILLISECOND, 
                tq.enqueue_time, 
                tq.transmission_status_time
            )
        ) as AvgProcessingTimeMs,
        COUNT(
            CASE 
                WHEN tq.transmission_status < 0 
                THEN 1 
            END
        ) as ErrorCount,
        SUM(tq.retry_count) as RetryCount,
        (
            SELECT SUM(used_page_count) * 8 
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = q.object_id
        ) as StorageUsedKB,
        GETUTCDATE()
    FROM sys.service_queues q
    JOIN sys.services s 
        ON q.service_id = s.service_id
    JOIN sys.service_contracts sc 
        ON s.service_contract_id = sc.service_contract_id
    JOIN sys.service_message_types mt 
        ON sc.service_contract_id = mt.service_contract_id
    LEFT JOIN sys.transmission_queue tq 
        ON s.service_id = tq.service_id
    GROUP BY 
        q.object_id,
        q.name,
        s.name,
        q.activation_state_desc,
        q.activation_procedure_id,
        sc.service_contract_id;

    -- Analyze Service Broker patterns
    WITH BrokerMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            QueueName,
            ServiceName,
            MessageCount,
            ActivationState,
            ProcedureExecutions,
            AvgProcessingTimeMs,
            ErrorCount,
            RetryCount,
            StorageUsedKB / 1024.0 as StorageUsedMB,
            LAG(MessageCount) OVER (
                PARTITION BY DatabaseName, 
                             SchemaName, 
                             QueueName 
                ORDER BY CollectionTime
            ) as PreviousMessageCount
        FROM dbo.ServiceBrokerMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        QueueName,
        ServiceName,
        MessageCount,
        ActivationState,
        ProcedureExecutions,
        AvgProcessingTimeMs,
        ErrorCount,
        RetryCount,
        StorageUsedMB,
        CASE 
            WHEN MessageCount > @HighQueueThreshold 
                 AND AvgProcessingTimeMs > @HighLatencyThresholdMs 
            THEN 'Critical Performance'
            WHEN MessageCount > @HighQueueThreshold 
            THEN 'High Queue Depth'
            WHEN AvgProcessingTimeMs > @HighLatencyThresholdMs 
            THEN 'High Latency'
            WHEN ErrorCount > 0 
            THEN 'Errors Detected'
            ELSE 'Normal'
        END as QueueStatus,
        CASE 
            WHEN MessageCount > @HighQueueThreshold 
                 AND AvgProcessingTimeMs > @HighLatencyThresholdMs 
            THEN 'Urgent:
                  1. Increase activation threads
                  2. Review procedure performance
                  3. Check for blocking'
            WHEN MessageCount > @HighQueueThreshold 
            THEN 'Scale processing capacity'
            WHEN AvgProcessingTimeMs > @HighLatencyThresholdMs 
            THEN 'Optimize message processing'
            WHEN ErrorCount > 0 
            THEN 'Investigate error causes'
            ELSE 'No action needed'
        END as Recommendation
    FROM BrokerMetrics
    WHERE MessageCount > @HighQueueThreshold
    OR AvgProcessingTimeMs > @HighLatencyThresholdMs
    OR ErrorCount > 0
    OR MessageCount > COALESCE(PreviousMessageCount, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN MessageCount > @HighQueueThreshold 
                 AND AvgProcessingTimeMs > @HighLatencyThresholdMs THEN 1
            WHEN MessageCount > @HighQueueThreshold THEN 2
            WHEN AvgProcessingTimeMs > @HighLatencyThresholdMs THEN 3
            ELSE 4
        END,
        MessageCount DESC;
END;
```

### Conversation Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeServiceBrokerConversations
AS
BEGIN
    -- Analyze conversation patterns
    SELECT 
        DB_NAME() as DatabaseName,
        s.name as ServiceName,
        ce.state_desc as ConversationState,
        COUNT(*) as ConversationCount,
        AVG(
            DATEDIFF(
                MILLISECOND, 
                ce.lifetime_start, 
                ce.lifetime_end
            )
        ) as AvgDurationMs,
        MAX(
            DATEDIFF(
                MILLISECOND, 
                ce.lifetime_start, 
                ce.lifetime_end
            )
        ) as MaxDurationMs,
        COUNT(
            CASE 
                WHEN ce.lifetime_end IS NULL 
                THEN 1 
            END
        ) as OpenConversations,
        SUM(ce.message_sequence_number) as TotalMessages,
        COUNT(
            CASE 
                WHEN ce.security_timestamp IS NOT NULL 
                THEN 1 
            END
        ) as SecuredConversations,
        COUNT(
            CASE 
                WHEN ce.is_system 
                THEN 1 
            END
        ) as SystemConversations,
        CASE 
            WHEN COUNT(*) > 1000 
            THEN 'High Volume'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND, 
                    ce.lifetime_start, 
                    ce.lifetime_end
                )
            ) > 5000 
            THEN 'Long Duration'
            WHEN COUNT(
                CASE 
                    WHEN ce.lifetime_end IS NULL 
                    THEN 1 
                END
            ) > 100 
            THEN 'Many Open'
            ELSE 'Normal'
        END as ConversationPattern,
        CASE 
            WHEN COUNT(*) > 1000 
            THEN 'Review conversation cleanup'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND, 
                    ce.lifetime_start, 
                    ce.lifetime_end
                )
            ) > 5000 
            THEN 'Optimize conversation duration'
            WHEN COUNT(
                CASE 
                    WHEN ce.lifetime_end IS NULL 
                    THEN 1 
                END
            ) > 100 
            THEN 'Check for stuck conversations'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.conversation_endpoints ce
    JOIN sys.services s 
        ON ce.service_id = s.service_id
    GROUP BY 
        s.name,
        ce.state_desc
    HAVING COUNT(*) > 100
    OR AVG(
        DATEDIFF(
            MILLISECOND, 
            ce.lifetime_start, 
            ce.lifetime_end
        )
    ) > 5000
    OR COUNT(
        CASE 
            WHEN ce.lifetime_end IS NULL 
            THEN 1 
        END
    ) > 100
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 1000 THEN 1
            WHEN COUNT(
                CASE 
                    WHEN ce.lifetime_end IS NULL 
                    THEN 1 
                END
            ) > 100 THEN 2
            ELSE 3
        END,
        ConversationCount DESC;
END;
```

### Message Type Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeMessageTypes
AS
BEGIN
    -- Analyze message patterns by type
    SELECT 
        mt.name as MessageTypeName,
        mt.validation_desc as ValidationLevel,
        mt.schema_collection_id,
        COUNT(DISTINCT sc.service_contract_id) as ContractCount,
        COUNT(DISTINCT s.service_id) as ServiceCount,
        (
            SELECT COUNT(*) 
            FROM sys.transmission_queue tq
            JOIN sys.conversation_endpoints ce 
                ON tq.conversation_handle = ce.conversation_handle
            WHERE ce.message_type_id = mt.message_type_id
        ) as PendingMessages,
        AVG(
            DATEDIFF(
                MILLISECOND, 
                tq.enqueue_time, 
                tq.transmission_status_time
            )
        ) as AvgProcessingTimeMs,
        COUNT(
            CASE 
                WHEN tq.transmission_status < 0 
                THEN 1 
            END
        ) as ErrorCount,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN tq.transmission_status < 0 
                    THEN 1 
                END
            ) > 0 
            THEN 'Errors Present'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND, 
                    tq.enqueue_time, 
                    tq.transmission_status_time
                )
            ) > 5000 
            THEN 'High Latency'
            WHEN COUNT(*) > 1000 
            THEN 'High Volume'
            ELSE 'Normal'
        END as MessagePattern,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN tq.transmission_status < 0 
                    THEN 1 
                END
            ) > 0 
            THEN 'Investigate message validation'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND, 
                    tq.enqueue_time, 
                    tq.transmission_status_time
                )
            ) > 5000 
            THEN 'Review processing time'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor volume patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.service_message_types mt
    JOIN sys.service_contracts sc 
        ON mt.service_contract_id = sc.service_contract_id
    JOIN sys.services s 
        ON sc.service_contract_id = s.service_contract_id
    LEFT JOIN sys.transmission_queue tq 
        ON s.service_id = tq.service_id
    GROUP BY 
        mt.name,
        mt.validation_desc,
        mt.schema_collection_id,
        mt.message_type_id
    HAVING COUNT(
        CASE 
            WHEN tq.transmission_status < 0 
            THEN 1 
        END
    ) > 0
    OR AVG(
        DATEDIFF(
            MILLISECOND, 
            tq.enqueue_time, 
            tq.transmission_status_time
        )
    ) > 5000
    OR COUNT(*) > 1000
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN tq.transmission_status < 0 
                    THEN 1 
                END
            ) > 0 THEN 1
            WHEN COUNT(*) > 1000 THEN 2
            ELSE 3
        END,
        ErrorCount DESC;
END;
```

This Service Broker analysis framework provides comprehensive tools for:
1. Monitoring queue performance and processing efficiency
2. Analyzing conversation patterns and lifecycle
3. Tracking message type usage and validation
4. Optimizing Service Broker operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
