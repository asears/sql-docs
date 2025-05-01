# SQL Server Always Encrypted Analysis Framework

## Encryption Performance Monitoring

### Encrypted Column Analysis
```sql
CREATE TABLE dbo.EncryptionMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    ColumnName sysname,
    EncryptionType nvarchar(60),
    KeyName sysname,
    QueryCount int,
    AvgDurationMs decimal(18,2),
    CPUTimeMs decimal(18,2),
    EncryptedBytes bigint,
    LastAccessTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorEncryptedColumns
    @HighLatencyThresholdMs decimal(18,2) = 100.0,
    @HighUsageThreshold int = 1000
AS
BEGIN
    -- Capture encryption metrics
    INSERT INTO dbo.EncryptionMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        c.name as ColumnName,
        ec.encryption_type_desc as EncryptionType,
        k.name as KeyName,
        COUNT(DISTINCT qs.query_id) as QueryCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        SUM(rs.avg_cpu_time) as CPUTimeMs,
        SUM(ps.used_page_count) * 8 * 1024 as EncryptedBytes,
        MAX(rs.last_execution_time) as LastAccessTime,
        GETUTCDATE()
    FROM sys.tables t
    JOIN sys.columns c ON t.object_id = c.object_id
    JOIN sys.column_encryption_keys k 
        ON c.column_encryption_key_id = k.column_encryption_key_id
    JOIN sys.column_encryption_properties ec 
        ON c.object_id = ec.object_id 
        AND c.column_id = ec.column_id
    JOIN sys.dm_db_partition_stats ps 
        ON t.object_id = ps.object_id
    LEFT JOIN sys.query_store_query qs
        ON CHARINDEX(
            c.name,
            OBJECT_NAME(
                OBJECT_ID(
                    qs.query_sql_text
                )
            )
        ) > 0
    LEFT JOIN sys.query_store_runtime_stats rs 
        ON qs.query_id = rs.plan_id
    GROUP BY 
        t.object_id,
        t.name,
        c.name,
        ec.encryption_type_desc,
        k.name;

    -- Analyze encryption patterns
    WITH EncryptionMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            TableName,
            ColumnName,
            EncryptionType,
            KeyName,
            QueryCount,
            AvgDurationMs,
            CPUTimeMs,
            EncryptedBytes / 1048576.0 as EncryptedMB,
            LastAccessTime,
            LAG(AvgDurationMs) OVER (
                PARTITION BY DatabaseName, SchemaName, 
                             TableName, ColumnName 
                ORDER BY CollectionTime
            ) as PreviousAvgDuration
        FROM dbo.EncryptionMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -24, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        ColumnName,
        EncryptionType,
        KeyName,
        QueryCount,
        AvgDurationMs,
        CPUTimeMs,
        EncryptedMB,
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
                 AND QueryCount > @HighUsageThreshold 
            THEN 'Critical Performance'
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
            THEN 'High Latency'
            WHEN QueryCount > @HighUsageThreshold 
            THEN 'High Usage'
            WHEN AvgDurationMs > 
                 COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Performance Regression'
            ELSE 'Normal'
        END as EncryptionStatus,
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
                 AND QueryCount > @HighUsageThreshold 
            THEN 'Review:
                  1. Encryption type
                  2. Index strategy
                  3. Query patterns'
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
            THEN 'Optimize query performance'
            WHEN QueryCount > @HighUsageThreshold 
            THEN 'Monitor resource usage'
            WHEN AvgDurationMs > 
                 COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Investigate regression'
            ELSE 'No action needed'
        END as Recommendation
    FROM EncryptionMetrics
    WHERE AvgDurationMs > @HighLatencyThresholdMs
    OR QueryCount > @HighUsageThreshold
    OR AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN AvgDurationMs > @HighLatencyThresholdMs 
                 AND QueryCount > @HighUsageThreshold THEN 1
            WHEN AvgDurationMs > @HighLatencyThresholdMs THEN 2
            ELSE 3
        END,
        AvgDurationMs DESC;
END;
```

### Key Rotation Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeKeyRotation
AS
BEGIN
    -- Analyze key rotation patterns
    SELECT 
        k.name as KeyName,
        k.key_store_provider_name as Provider,
        k.create_date as CreationDate,
        k.modify_date as LastModified,
        COUNT(DISTINCT c.object_id) as EncryptedObjects,
        COUNT(DISTINCT c.column_id) as EncryptedColumns,
        SUM(ps.used_page_count) * 8.0 / 1024 as EncryptedDataMB,
        DATEDIFF(
            DAY,
            k.create_date,
            GETUTCDATE()
        ) as KeyAgeDays,
        CASE 
            WHEN DATEDIFF(
                DAY,
                k.create_date,
                GETUTCDATE()
            ) > 365 
            THEN 'Key Rotation Due'
            WHEN COUNT(DISTINCT c.object_id) = 0 
            THEN 'Unused Key'
            WHEN k.key_store_provider_name = 'MSSQL_CERTIFICATE_STORE' 
            THEN 'Consider Azure Key Vault'
            ELSE 'Normal'
        END as KeyStatus,
        CASE 
            WHEN DATEDIFF(
                DAY,
                k.create_date,
                GETUTCDATE()
            ) > 365 
            THEN 'Schedule key rotation'
            WHEN COUNT(DISTINCT c.object_id) = 0 
            THEN 'Review key necessity'
            WHEN k.key_store_provider_name = 'MSSQL_CERTIFICATE_STORE' 
            THEN 'Evaluate Azure Key Vault'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.column_encryption_keys k
    LEFT JOIN sys.columns c 
        ON c.column_encryption_key_id = k.column_encryption_key_id
    LEFT JOIN sys.dm_db_partition_stats ps 
        ON c.object_id = ps.object_id
    GROUP BY 
        k.name,
        k.key_store_provider_name,
        k.create_date,
        k.modify_date
    HAVING DATEDIFF(
        DAY,
        k.create_date,
        GETUTCDATE()
    ) > 365
    OR COUNT(DISTINCT c.object_id) = 0
    OR k.key_store_provider_name = 'MSSQL_CERTIFICATE_STORE'
    ORDER BY 
        CASE 
            WHEN DATEDIFF(
                DAY,
                k.create_date,
                GETUTCDATE()
            ) > 365 THEN 1
            WHEN COUNT(DISTINCT c.object_id) = 0 THEN 2
            ELSE 3
        END,
        KeyAgeDays DESC;
END;
```

### Encryption Impact Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeEncryptionImpact
AS
BEGIN
    -- Analyze encryption performance impact
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        AVG(rs.avg_cpu_time) as AvgCPUTimeMs,
        AVG(rs.avg_logical_io_reads) as AvgLogicalReads,
        COUNT(DISTINCT c.name) as EncryptedColumnsAccessed,
        STRING_AGG(
            c.name + ' (' + 
            ec.encryption_type_desc + ')',
            ', '
        ) as EncryptedColumns,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'High Impact'
            WHEN COUNT(*) > 1000 
            THEN 'Frequently Used'
            WHEN AVG(rs.avg_duration) > 1000 
            THEN 'Long Running'
            ELSE 'Normal'
        END as QueryPattern,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'Optimize encryption usage'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor performance'
            WHEN AVG(rs.avg_duration) > 1000 
            THEN 'Review query patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    JOIN sys.columns c 
        ON c.column_encryption_key_id IS NOT NULL
    JOIN sys.column_encryption_properties ec 
        ON c.object_id = ec.object_id 
        AND c.column_id = ec.column_id
    WHERE qt.query_sql_text LIKE '%' + c.name + '%'
    GROUP BY 
        q.query_id,
        qt.query_sql_text,
        p.query_plan
    HAVING COUNT(*) > 1000
    OR AVG(rs.avg_duration) > 1000
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 THEN 1
            WHEN COUNT(*) > 1000 THEN 2
            ELSE 3
        END,
        AvgDurationMs DESC;
END;
```

This Always Encrypted Analysis framework provides comprehensive tools for:
1. Monitoring encrypted column performance and usage patterns
2. Analyzing encryption key rotation and management
3. Tracking encryption impact on query performance
4. Optimizing encrypted data operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
