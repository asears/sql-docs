# SQL Server Ledger Analysis Framework

## Ledger Performance Monitoring

### Ledger Table Analysis
```sql
CREATE TABLE dbo.LedgerMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    LedgerType nvarchar(60),
    RowCount bigint,
    DigestSize bigint,
    TransactionCount int,
    VerificationCount int,
    LastDigestTime datetime2,
    LastVerificationTime datetime2,
    AvgVerificationMs decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorLedgerTables
    @HighVerificationThresholdMs decimal(18,2) = 1000.0,
    @DigestAgeThresholdHours int = 24
AS
BEGIN
    -- Capture ledger metrics
    INSERT INTO dbo.LedgerMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        t.ledger_type_desc as LedgerType,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = t.object_id
            AND ps.index_id <= 1
        ) as RowCount,
        (
            SELECT SUM(ps.used_page_count) * 8 * 1024
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = t.ledger_view_id
        ) as DigestSize,
        COUNT(DISTINCT tr.transaction_id) as TransactionCount,
        COUNT(DISTINCT lv.verification_time) as VerificationCount,
        MAX(tr.commit_time) as LastDigestTime,
        MAX(lv.verification_time) as LastVerificationTime,
        AVG(
            DATEDIFF(
                MILLISECOND,
                lv.start_time,
                lv.verification_time
            )
        ) as AvgVerificationMs,
        GETUTCDATE()
    FROM sys.tables t
    JOIN sys.ledger_table_history th 
        ON t.object_id = th.table_id
    LEFT JOIN sys.ledger_transactions tr 
        ON th.transaction_id = tr.transaction_id
    LEFT JOIN sys.ledger_verifications lv 
        ON t.object_id = lv.table_id
    WHERE t.ledger_type IS NOT NULL
    GROUP BY 
        t.object_id,
        t.name,
        t.ledger_type_desc,
        t.ledger_view_id;

    -- Analyze ledger patterns
    WITH LedgerMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            TableName,
            LedgerType,
            RowCount,
            DigestSize / 1048576.0 as DigestSizeMB,
            TransactionCount,
            VerificationCount,
            LastDigestTime,
            LastVerificationTime,
            AvgVerificationMs,
            DATEDIFF(
                HOUR,
                LastDigestTime,
                GETUTCDATE()
            ) as DigestAgeHours
        FROM dbo.LedgerMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -24, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        LedgerType,
        RowCount,
        DigestSizeMB,
        TransactionCount,
        VerificationCount,
        LastDigestTime,
        LastVerificationTime,
        AvgVerificationMs,
        DigestAgeHours,
        CASE 
            WHEN AvgVerificationMs > @HighVerificationThresholdMs 
                 AND DigestAgeHours > @DigestAgeThresholdHours 
            THEN 'Critical Performance'
            WHEN AvgVerificationMs > @HighVerificationThresholdMs 
            THEN 'Slow Verification'
            WHEN DigestAgeHours > @DigestAgeThresholdHours 
            THEN 'Stale Digest'
            WHEN VerificationCount = 0 
            THEN 'No Verifications'
            ELSE 'Normal'
        END as LedgerStatus,
        CASE 
            WHEN AvgVerificationMs > @HighVerificationThresholdMs 
                 AND DigestAgeHours > @DigestAgeThresholdHours 
            THEN 'Review:
                  1. Verification process
                  2. Digest frequency
                  3. Table size'
            WHEN AvgVerificationMs > @HighVerificationThresholdMs 
            THEN 'Optimize verification'
            WHEN DigestAgeHours > @DigestAgeThresholdHours 
            THEN 'Update digest'
            WHEN VerificationCount = 0 
            THEN 'Schedule verification'
            ELSE 'No action needed'
        END as Recommendation
    FROM LedgerMetrics
    WHERE AvgVerificationMs > @HighVerificationThresholdMs
    OR DigestAgeHours > @DigestAgeThresholdHours
    OR VerificationCount = 0
    ORDER BY 
        CASE 
            WHEN AvgVerificationMs > @HighVerificationThresholdMs 
                 AND DigestAgeHours > @DigestAgeThresholdHours THEN 1
            WHEN AvgVerificationMs > @HighVerificationThresholdMs THEN 2
            ELSE 3
        END,
        AvgVerificationMs DESC;
END;
```

### Digest Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeDigestPerformance
AS
BEGIN
    -- Analyze digest generation patterns
    SELECT 
        t.name as TableName,
        tr.transaction_id,
        tr.commit_time,
        tr.digest_creation_time,
        tr.sequence_number,
        DATEDIFF(
            MILLISECOND,
            tr.commit_time,
            tr.digest_creation_time
        ) as DigestLatencyMs,
        tr.principal_name as DigestCreator,
        CASE 
            WHEN DATEDIFF(
                MILLISECOND,
                tr.commit_time,
                tr.digest_creation_time
            ) > 1000 
            THEN 'High Latency'
            WHEN tr.append_only_duplicate = 1 
            THEN 'Duplicate Transaction'
            ELSE 'Normal'
        END as DigestStatus,
        CASE 
            WHEN DATEDIFF(
                MILLISECOND,
                tr.commit_time,
                tr.digest_creation_time
            ) > 1000 
            THEN 'Review digest performance'
            WHEN tr.append_only_duplicate = 1 
            THEN 'Investigate duplicates'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.tables t
    JOIN sys.ledger_table_history th 
        ON t.object_id = th.table_id
    JOIN sys.ledger_transactions tr 
        ON th.transaction_id = tr.transaction_id
    WHERE DATEDIFF(
        MILLISECOND,
        tr.commit_time,
        tr.digest_creation_time
    ) > 1000
    OR tr.append_only_duplicate = 1
    ORDER BY 
        CASE 
            WHEN DATEDIFF(
                MILLISECOND,
                tr.commit_time,
                tr.digest_creation_time
            ) > 1000 THEN 1
            ELSE 2
        END,
        DigestLatencyMs DESC;
END;
```

### Verification History Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeVerificationHistory
AS
BEGIN
    -- Analyze verification history patterns
    SELECT 
        t.name as TableName,
        COUNT(*) as VerificationCount,
        AVG(
            DATEDIFF(
                MILLISECOND,
                lv.start_time,
                lv.verification_time
            )
        ) as AvgVerificationMs,
        MAX(
            DATEDIFF(
                MILLISECOND,
                lv.start_time,
                lv.verification_time
            )
        ) as MaxVerificationMs,
        MIN(lv.start_time) as FirstVerification,
        MAX(lv.verification_time) as LastVerification,
        COUNT(
            CASE 
                WHEN lv.verification_result = 0 
                THEN 1 
            END
        ) as FailedVerifications,
        STRING_AGG(
            CASE 
                WHEN lv.verification_result = 0 
                THEN CONVERT(varchar(20), lv.verification_time, 120)
            END,
            ', '
        ) as FailureTimes,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN lv.verification_result = 0 
                    THEN 1 
                END
            ) > 0 
            THEN 'Verification Failures'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND,
                    lv.start_time,
                    lv.verification_time
                )
            ) > 1000 
            THEN 'High Verification Time'
            WHEN DATEDIFF(
                HOUR,
                MAX(lv.verification_time),
                GETUTCDATE()
            ) > 24 
            THEN 'Outdated Verification'
            ELSE 'Normal'
        END as VerificationStatus,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN lv.verification_result = 0 
                    THEN 1 
                END
            ) > 0 
            THEN 'Investigate failures'
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND,
                    lv.start_time,
                    lv.verification_time
                )
            ) > 1000 
            THEN 'Optimize verification'
            WHEN DATEDIFF(
                HOUR,
                MAX(lv.verification_time),
                GETUTCDATE()
            ) > 24 
            THEN 'Schedule verification'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.tables t
    JOIN sys.ledger_verifications lv 
        ON t.object_id = lv.table_id
    GROUP BY 
        t.name
    HAVING COUNT(
        CASE 
            WHEN lv.verification_result = 0 
            THEN 1 
        END
    ) > 0
    OR AVG(
        DATEDIFF(
            MILLISECOND,
            lv.start_time,
            lv.verification_time
        )
    ) > 1000
    OR DATEDIFF(
        HOUR,
        MAX(lv.verification_time),
        GETUTCDATE()
    ) > 24
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN lv.verification_result = 0 
                    THEN 1 
                END
            ) > 0 THEN 1
            WHEN AVG(
                DATEDIFF(
                    MILLISECOND,
                    lv.start_time,
                    lv.verification_time
                )
            ) > 1000 THEN 2
            ELSE 3
        END,
        FailedVerifications DESC;
END;
```

This Ledger Analysis framework provides comprehensive tools for:
1. Monitoring ledger table performance and digest generation
2. Analyzing verification processes and performance
3. Tracking ledger operations and anomalies
4. Optimizing blockchain-enabled tables

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
