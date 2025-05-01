# SQL Server Accelerated Database Recovery Analysis Framework

## ADR Performance Monitoring

### Version Store Analysis
```sql
CREATE TABLE dbo.VersionStoreMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    VersionStoreSizeMB decimal(18,2),
    VersionGenerationKBSec decimal(18,2),
    CleanupRateKBSec decimal(18,2),
    OldestVersionMinutes int,
    ActiveTransactions int,
    LongRunningTransactions int,
    AbortedTransactions int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorVersionStore
    @HighGrowthThresholdMBMin decimal(18,2) = 100.0,
    @LongTransactionMinutes int = 30
AS
BEGIN
    -- Capture version store metrics
    INSERT INTO dbo.VersionStoreMetrics
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        persistent_version_store_size_kb / 1024.0 as VersionStoreSizeMB,
        version_generation_rate_kb_per_sec as VersionGenerationKBSec,
        version_cleanup_rate_kb_per_sec as CleanupRateKBSec,
        DATEDIFF(
            MINUTE, 
            oldest_active_transaction_begin_time, 
            GETUTCDATE()
        ) as OldestVersionMinutes,
        active_transaction_count as ActiveTransactions,
        (
            SELECT COUNT(*)
            FROM sys.dm_tran_active_transactions t
            WHERE t.database_id = vs.database_id
            AND DATEDIFF(
                MINUTE, 
                t.transaction_begin_time, 
                GETUTCDATE()
            ) > @LongTransactionMinutes
        ) as LongRunningTransactions,
        (
            SELECT COUNT(*)
            FROM sys.dm_tran_active_transactions t
            WHERE t.database_id = vs.database_id
            AND t.transaction_state = 3  -- Aborted
        ) as AbortedTransactions,
        GETUTCDATE()
    FROM sys.dm_tran_persistent_version_store_stats vs;

    -- Analyze version store patterns
    WITH VersionMetrics AS (
        SELECT 
            DatabaseName,
            VersionStoreSizeMB,
            VersionGenerationKBSec,
            CleanupRateKBSec,
            OldestVersionMinutes,
            ActiveTransactions,
            LongRunningTransactions,
            AbortedTransactions,
            LAG(VersionStoreSizeMB) OVER (
                PARTITION BY DatabaseName 
                ORDER BY CollectionTime
            ) as PreviousStoreSizeMB
        FROM dbo.VersionStoreMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        VersionStoreSizeMB,
        VersionGenerationKBSec,
        CleanupRateKBSec,
        OldestVersionMinutes,
        ActiveTransactions,
        LongRunningTransactions,
        AbortedTransactions,
        CASE 
            WHEN (VersionStoreSizeMB - COALESCE(PreviousStoreSizeMB, 0)) / 
                 NULLIF(
                    DATEDIFF(
                        MINUTE, 
                        CollectionTime, 
                        GETUTCDATE()
                    ),
                    0
                 ) > @HighGrowthThresholdMBMin 
            THEN 'Rapid Growth'
            WHEN LongRunningTransactions > 0 
            THEN 'Long Transactions'
            WHEN VersionGenerationKBSec > CleanupRateKBSec * 2 
            THEN 'Cleanup Falling Behind'
            ELSE 'Normal'
        END as StoreStatus,
        CASE 
            WHEN (VersionStoreSizeMB - COALESCE(PreviousStoreSizeMB, 0)) / 
                 NULLIF(
                    DATEDIFF(
                        MINUTE, 
                        CollectionTime, 
                        GETUTCDATE()
                    ),
                    0
                 ) > @HighGrowthThresholdMBMin 
            THEN 'Investigate version store growth'
            WHEN LongRunningTransactions > 0 
            THEN 'Review long-running transactions'
            WHEN VersionGenerationKBSec > CleanupRateKBSec * 2 
            THEN 'Check cleanup process'
            ELSE 'No action needed'
        END as Recommendation
    FROM VersionMetrics
    WHERE (VersionStoreSizeMB - COALESCE(PreviousStoreSizeMB, 0)) / 
           NULLIF(
            DATEDIFF(
                MINUTE, 
                CollectionTime, 
                GETUTCDATE()
            ),
            0
           ) > @HighGrowthThresholdMBMin
    OR LongRunningTransactions > 0
    OR VersionGenerationKBSec > CleanupRateKBSec * 2
    ORDER BY 
        CASE 
            WHEN (VersionStoreSizeMB - COALESCE(PreviousStoreSizeMB, 0)) / 
                 NULLIF(
                    DATEDIFF(
                        MINUTE, 
                        CollectionTime, 
                        GETUTCDATE()
                    ),
                    0
                 ) > @HighGrowthThresholdMBMin THEN 1
            WHEN LongRunningTransactions > 0 THEN 2
            ELSE 3
        END,
        VersionStoreSizeMB DESC;
END;
```

### Recovery Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeRecoveryPerformance
AS
BEGIN
    -- Analyze ADR recovery patterns
    SELECT 
        d.name as DatabaseName,
        d.recovery_model_desc as RecoveryModel,
        d.log_reuse_wait_desc as LogReuseWait,
        COUNT(DISTINCT vt.transaction_sequence_num) as TransactionCount,
        AVG(
            DATEDIFF(
                MILLISECOND,
                vt.transaction_begin_time,
                vt.transaction_end_time
            )
        ) as AvgTransactionDurationMs,
        MAX(vt.page_count) as MaxPagesModified,
        SUM(vt.persistent_version_store_size_kb) / 1024.0 as TotalVersionStoreMB,
        COUNT(
            CASE 
                WHEN vt.transaction_type = 2 
                THEN 1 
            END
        ) as RollbackTransactions,
        MAX(
            DATEDIFF(
                SECOND,
                vt.transaction_begin_time,
                vt.transaction_end_time
            )
        ) as MaxTransactionDurationSec,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN vt.transaction_type = 2 
                    THEN 1 
                END
            ) > 10 
            THEN 'High Rollback Rate'
            WHEN MAX(
                DATEDIFF(
                    SECOND,
                    vt.transaction_begin_time,
                    vt.transaction_end_time
                )
            ) > 3600 
            THEN 'Long Transactions'
            WHEN SUM(
                vt.persistent_version_store_size_kb
            ) / 1024.0 > 1024 
            THEN 'Large Version Store'
            ELSE 'Normal'
        END as RecoveryPattern,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN vt.transaction_type = 2 
                    THEN 1 
                END
            ) > 10 
            THEN 'Review transaction patterns'
            WHEN MAX(
                DATEDIFF(
                    SECOND,
                    vt.transaction_begin_time,
                    vt.transaction_end_time
                )
            ) > 3600 
            THEN 'Optimize long transactions'
            WHEN SUM(
                vt.persistent_version_store_size_kb
            ) / 1024.0 > 1024 
            THEN 'Monitor version store size'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.databases d
    JOIN sys.dm_tran_version_store_transactions vt 
        ON d.database_id = vt.database_id
    WHERE d.is_accelerated_database_recovery_on = 1
    GROUP BY 
        d.name,
        d.recovery_model_desc,
        d.log_reuse_wait_desc
    HAVING COUNT(
        CASE 
            WHEN vt.transaction_type = 2 
            THEN 1 
        END
    ) > 10
    OR MAX(
        DATEDIFF(
            SECOND,
            vt.transaction_begin_time,
            vt.transaction_end_time
        )
    ) > 3600
    OR SUM(vt.persistent_version_store_size_kb) / 1024.0 > 1024
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN vt.transaction_type = 2 
                    THEN 1 
                END
            ) > 10 THEN 1
            WHEN MAX(
                DATEDIFF(
                    SECOND,
                    vt.transaction_begin_time,
                    vt.transaction_end_time
                )
            ) > 3600 THEN 2
            ELSE 3
        END,
        TransactionCount DESC;
END;
```

### Cleanup Process Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeCleanupProcess
AS
BEGIN
    -- Analyze version store cleanup patterns
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        persistent_version_store_size_kb / 1024.0 as VersionStoreSizeMB,
        version_cleanup_rate_kb_per_sec as CleanupRateKBSec,
        AVG(cleanup_rate_kb_per_sec) OVER (
            PARTITION BY database_id 
            ORDER BY cleanup_time_window 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as AvgCleanupRateKBSec,
        CASE 
            WHEN version_cleanup_rate_kb_per_sec < 
                 version_generation_rate_kb_per_sec 
            THEN 'Falling Behind'
            WHEN persistent_version_store_size_kb / 1024.0 > 1024 
                 AND version_cleanup_rate_kb_per_sec = 0 
            THEN 'Cleanup Stalled'
            WHEN version_cleanup_rate_kb_per_sec > 
                 version_generation_rate_kb_per_sec * 2 
            THEN 'Aggressive Cleanup'
            ELSE 'Normal'
        END as CleanupStatus,
        CASE 
            WHEN version_cleanup_rate_kb_per_sec < 
                 version_generation_rate_kb_per_sec 
            THEN 'Increase cleanup resources'
            WHEN persistent_version_store_size_kb / 1024.0 > 1024 
                 AND version_cleanup_rate_kb_per_sec = 0 
            THEN 'Investigate cleanup stall'
            WHEN version_cleanup_rate_kb_per_sec > 
                 version_generation_rate_kb_per_sec * 2 
            THEN 'Monitor cleanup impact'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_tran_persistent_version_store_stats
    WHERE version_cleanup_rate_kb_per_sec < version_generation_rate_kb_per_sec
    OR (
        persistent_version_store_size_kb / 1024.0 > 1024 
        AND version_cleanup_rate_kb_per_sec = 0
    )
    OR version_cleanup_rate_kb_per_sec > version_generation_rate_kb_per_sec * 2
    ORDER BY 
        CASE 
            WHEN version_cleanup_rate_kb_per_sec < 
                 version_generation_rate_kb_per_sec THEN 1
            WHEN persistent_version_store_size_kb / 1024.0 > 1024 
                 AND version_cleanup_rate_kb_per_sec = 0 THEN 2
            ELSE 3
        END,
        persistent_version_store_size_kb DESC;
END;
```

This Accelerated Database Recovery analysis framework provides comprehensive tools for:
1. Monitoring version store growth and cleanup efficiency
2. Analyzing recovery performance and transaction patterns
3. Tracking cleanup process effectiveness
4. Optimizing ADR operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
