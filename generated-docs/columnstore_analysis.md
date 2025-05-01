# SQL Server Columnstore Index Analysis Framework

## Columnstore Performance Monitoring

### Segment Health Analysis
```sql
CREATE TABLE dbo.ColumnstoreSegmentMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    SchemaName sysname,
    TableName sysname,
    IndexName sysname,
    PartitionNumber int,
    SegmentCount int,
    DeletedRows bigint,
    CompressedRows bigint,
    OpenRows bigint,
    AvgRowGroupSize int,
    CompressionRatio decimal(10,2),
    FragmentationPercent decimal(5,2),
    TrimmedRowGroups int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorColumnstoreSegments
    @CompressionThreshold decimal(5,2) = 80.0,
    @FragmentationThreshold decimal(5,2) = 20.0
AS
BEGIN
    -- Capture columnstore metrics
    INSERT INTO dbo.ColumnstoreSegmentMetrics
    SELECT 
        OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
        OBJECT_NAME(i.object_id) as TableName,
        i.name as IndexName,
        rg.partition_number,
        COUNT(*) as SegmentCount,
        SUM(rg.deleted_rows) as DeletedRows,
        SUM(rg.total_rows - rg.deleted_rows) as CompressedRows,
        (
            SELECT SUM(r.row_count)
            FROM sys.dm_db_column_store_row_group_physical_stats r
            WHERE r.object_id = i.object_id
            AND r.state = 1  -- OPEN
        ) as OpenRows,
        AVG(rg.total_rows) as AvgRowGroupSize,
        AVG(CAST(rg.size_in_bytes as decimal(10,2)) / 
            NULLIF(rg.total_rows * 
                  (
                      SELECT AVG(max_length)
                      FROM sys.columns c
                      WHERE c.object_id = i.object_id
                  ), 0)) as CompressionRatio,
        SUM(CASE 
            WHEN rg.total_rows < 500000 THEN 1
            ELSE 0 
        END) * 100.0 / COUNT(*) as FragmentationPercent,
        SUM(CASE 
            WHEN rg.total_rows < 100000 THEN 1
            ELSE 0 
        END) as TrimmedRowGroups,
        GETUTCDATE()
    FROM sys.indexes i
    JOIN sys.dm_db_column_store_row_group_physical_stats rg 
        ON i.object_id = rg.object_id
        AND i.index_id = rg.index_id
    WHERE i.type IN (5, 6)  -- Columnstore indexes
    GROUP BY 
        i.object_id,
        i.name,
        rg.partition_number;

    -- Analyze segment health
    WITH SegmentMetrics AS (
        SELECT 
            SchemaName,
            TableName,
            IndexName,
            PartitionNumber,
            SegmentCount,
            DeletedRows,
            CompressedRows,
            OpenRows,
            AvgRowGroupSize,
            CompressionRatio,
            FragmentationPercent,
            TrimmedRowGroups,
            ROW_NUMBER() OVER (
                PARTITION BY TableName, IndexName 
                ORDER BY CollectionTime DESC
            ) as rn
        FROM dbo.ColumnstoreSegmentMetrics
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        PartitionNumber,
        SegmentCount,
        DeletedRows,
        CompressedRows,
        OpenRows,
        AvgRowGroupSize,
        CompressionRatio,
        FragmentationPercent,
        TrimmedRowGroups,
        CASE 
            WHEN DeletedRows > CompressedRows * 0.2 
            THEN 'High Deleted Rows'
            WHEN FragmentationPercent > @FragmentationThreshold 
            THEN 'High Fragmentation'
            WHEN CompressionRatio < @CompressionThreshold 
            THEN 'Low Compression'
            ELSE 'Healthy'
        END as SegmentHealth,
        CASE 
            WHEN DeletedRows > CompressedRows * 0.2 
            THEN 'ALTER INDEX REORGANIZE'
            WHEN FragmentationPercent > @FragmentationThreshold 
            THEN 'Consider index rebuild'
            WHEN CompressionRatio < @CompressionThreshold 
            THEN 'Review archival strategy'
            ELSE 'No action needed'
        END as Recommendation
    FROM SegmentMetrics
    WHERE rn = 1
    AND (
        DeletedRows > CompressedRows * 0.2
        OR FragmentationPercent > @FragmentationThreshold
        OR CompressionRatio < @CompressionThreshold
    )
    ORDER BY 
        CASE 
            WHEN DeletedRows > CompressedRows * 0.2 THEN 1
            WHEN FragmentationPercent > @FragmentationThreshold THEN 2
            ELSE 3
        END,
        CompressedRows DESC;
END;
```

### Columnstore Query Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeColumnstoreQueries
AS
BEGIN
    -- Analyze query patterns
    SELECT TOP 50
        DB_NAME(qt.dbid) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qs.execution_count,
        qs.total_worker_time / 1000000.0 as TotalCPUSeconds,
        qs.total_elapsed_time / 1000000.0 as TotalDurationSeconds,
        qs.total_logical_reads / qs.execution_count as AvgLogicalReads,
        qs.total_physical_reads / qs.execution_count as AvgPhysicalReads,
        CAST(qp.query_plan as xml) as QueryPlan,
        CASE 
            WHEN qp.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:ColumnstoreScan'
            ) = 1 THEN 'Columnstore Scan'
            WHEN qp.query_plan.exist(
                'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                //p:ColumnstoreIndex'
            ) = 1 THEN 'Columnstore Index'
            ELSE 'Non-Columnstore'
        END as OperationType,
        CASE 
            WHEN qs.total_worker_time / qs.execution_count > 1000000 
            THEN 'High CPU'
            WHEN qs.total_logical_reads / qs.execution_count > 1000 
            THEN 'High IO'
            ELSE 'Normal'
        END as PerformanceStatus
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE qp.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:ColumnstoreIndex'
    ) = 1
    ORDER BY qs.total_worker_time DESC;
END;
```

### Archival Group Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeColumnstoreArchival
    @ArchivalThresholdDays int = 90,
    @CompressionRatioThreshold decimal(5,2) = 10.0
AS
BEGIN
    -- Analyze archival candidates
    WITH ArchivalMetrics AS (
        SELECT 
            OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
            OBJECT_NAME(i.object_id) as TableName,
            i.name as IndexName,
            rg.partition_number,
            rg.row_group_id,
            rg.state,
            rg.total_rows,
            rg.size_in_bytes,
            rg.created_time,
            DATEDIFF(DAY, rg.created_time, GETDATE()) as AgeDays,
            CAST(rg.size_in_bytes as decimal(10,2)) / 
                NULLIF(rg.total_rows * 
                      (
                          SELECT AVG(max_length)
                          FROM sys.columns c
                          WHERE c.object_id = i.object_id
                      ), 0) as CompressionRatio
        FROM sys.indexes i
        JOIN sys.dm_db_column_store_row_group_physical_stats rg 
            ON i.object_id = rg.object_id
            AND i.index_id = rg.index_id
        WHERE i.type IN (5, 6)  -- Columnstore indexes
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        partition_number,
        COUNT(*) as RowGroupCount,
        SUM(total_rows) as TotalRows,
        SUM(size_in_bytes) / 1048576.0 as SizeMB,
        MIN(created_time) as OldestRowGroup,
        MAX(created_time) as NewestRowGroup,
        AVG(CompressionRatio) as AvgCompressionRatio,
        CASE 
            WHEN MIN(AgeDays) > @ArchivalThresholdDays 
                 AND AVG(CompressionRatio) > @CompressionRatioThreshold 
            THEN 'Archival Candidate'
            WHEN MIN(AgeDays) > @ArchivalThresholdDays 
            THEN 'Age Based Archival'
            WHEN AVG(CompressionRatio) > @CompressionRatioThreshold 
            THEN 'Compression Based Archival'
            ELSE 'Retain'
        END as ArchivalStatus,
        CASE 
            WHEN MIN(AgeDays) > @ArchivalThresholdDays 
                 AND AVG(CompressionRatio) > @CompressionRatioThreshold 
            THEN 'Consider partition switching to archive table'
            WHEN MIN(AgeDays) > @ArchivalThresholdDays 
            THEN 'Review data retention policy'
            WHEN AVG(CompressionRatio) > @CompressionRatioThreshold 
            THEN 'Review compression settings'
            ELSE 'No action needed'
        END as Recommendation
    FROM ArchivalMetrics
    GROUP BY 
        SchemaName,
        TableName,
        IndexName,
        partition_number
    HAVING 
        MIN(AgeDays) > @ArchivalThresholdDays
        OR AVG(CompressionRatio) > @CompressionRatioThreshold
    ORDER BY SizeMB DESC;
END;
```

### Delta Store Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeDeltaStores
    @OpenRowGroupThreshold int = 500000,
    @DeltaRowThreshold int = 100000
AS
BEGIN
    -- Analyze delta store patterns
    WITH DeltaMetrics AS (
        SELECT 
            OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
            OBJECT_NAME(i.object_id) as TableName,
            i.name as IndexName,
            rg.partition_number,
            rg.state,
            rg.state_desc,
            rg.total_rows,
            rg.deleted_rows,
            rg.size_in_bytes / 1048576.0 as SizeMB,
            DATEDIFF(MINUTE, rg.created_time, GETDATE()) as AgeMinutes
        FROM sys.indexes i
        JOIN sys.dm_db_column_store_row_group_physical_stats rg 
            ON i.object_id = rg.object_id
            AND i.index_id = rg.index_id
        WHERE i.type IN (5, 6)  -- Columnstore indexes
        AND rg.state IN (0, 1)  -- COMPRESSED or OPEN
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        partition_number,
        SUM(CASE 
            WHEN state = 1 THEN total_rows 
            ELSE 0 
        END) as OpenRowGroupRows,
        SUM(CASE 
            WHEN state = 0 THEN deleted_rows 
            ELSE 0 
        END) as DeletedDeltaRows,
        SUM(SizeMB) as TotalDeltaSizeMB,
        MAX(AgeMinutes) as OldestDeltaAgeMinutes,
        CASE 
            WHEN SUM(CASE 
                WHEN state = 1 THEN total_rows 
                ELSE 0 
            END) > @OpenRowGroupThreshold 
            THEN 'Large Open Row Groups'
            WHEN SUM(CASE 
                WHEN state = 0 THEN deleted_rows 
                ELSE 0 
            END) > @DeltaRowThreshold 
            THEN 'High Deleted Rows'
            WHEN MAX(AgeMinutes) > 1440  -- 24 hours
            THEN 'Aged Delta Store'
            ELSE 'Normal'
        END as DeltaStatus,
        CASE 
            WHEN SUM(CASE 
                WHEN state = 1 THEN total_rows 
                ELSE 0 
            END) > @OpenRowGroupThreshold 
            THEN 'Force row group compression'
            WHEN SUM(CASE 
                WHEN state = 0 THEN deleted_rows 
                ELSE 0 
            END) > @DeltaRowThreshold 
            THEN 'Reorganize index'
            WHEN MAX(AgeMinutes) > 1440 
            THEN 'Review insert patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM DeltaMetrics
    GROUP BY 
        SchemaName,
        TableName,
        IndexName,
        partition_number
    HAVING 
        SUM(CASE 
            WHEN state = 1 THEN total_rows 
            ELSE 0 
        END) > @OpenRowGroupThreshold
        OR SUM(CASE 
            WHEN state = 0 THEN deleted_rows 
            ELSE 0 
        END) > @DeltaRowThreshold
        OR MAX(AgeMinutes) > 1440
    ORDER BY 
        CASE 
            WHEN SUM(CASE 
                WHEN state = 1 THEN total_rows 
                ELSE 0 
            END) > @OpenRowGroupThreshold THEN 1
            WHEN SUM(CASE 
                WHEN state = 0 THEN deleted_rows 
                ELSE 0 
            END) > @DeltaRowThreshold THEN 2
            ELSE 3
        END,
        TotalDeltaSizeMB DESC;
END;
```

This columnstore analysis framework provides comprehensive tools for:
1. Monitoring segment health and compression efficiency
2. Analyzing columnstore query patterns and performance
3. Managing archival strategies for aged data
4. Optimizing delta store operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
