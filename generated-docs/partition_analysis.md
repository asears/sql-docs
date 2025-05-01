# SQL Server Partition Analysis Framework

## Partition Monitoring Framework

### Partition Size Analysis
```sql
CREATE TABLE dbo.PartitionMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    SchemaName sysname,
    TableName sysname,
    IndexName sysname,
    PartitionNumber int,
    PartitionFunction sysname,
    BoundaryValue sql_variant,
    RowCount bigint,
    ReservedSpaceMB decimal(18,2),
    UsedSpaceMB decimal(18,2),
    DataCompressionDesc nvarchar(60),
    FragmentationPercent decimal(5,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorPartitionSizes
    @LargePartitionThresholdGB decimal(10,2) = 100.0,
    @SkewThresholdPercent decimal(5,2) = 20.0
AS
BEGIN
    -- Capture partition metrics
    INSERT INTO dbo.PartitionMetrics
    SELECT 
        OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
        t.name as TableName,
        i.name as IndexName,
        p.partition_number,
        pf.name as PartitionFunction,
        prv.value as BoundaryValue,
        p.rows as RowCount,
        ps.reserved_page_count * 8.0 / 1024 as ReservedSpaceMB,
        ps.used_page_count * 8.0 / 1024 as UsedSpaceMB,
        p.data_compression_desc,
        ISNULL(
            idx.avg_fragmentation_in_percent,
            0
        ) as FragmentationPercent,
        GETUTCDATE()
    FROM sys.tables t
    JOIN sys.indexes i 
        ON t.object_id = i.object_id
    JOIN sys.partitions p 
        ON i.object_id = p.object_id 
        AND i.index_id = p.index_id
    JOIN sys.dm_db_partition_stats ps 
        ON p.partition_id = ps.partition_id
    JOIN sys.partition_schemes ps 
        ON i.data_space_id = ps.data_space_id
    JOIN sys.partition_functions pf 
        ON ps.function_id = pf.function_id
    LEFT JOIN sys.partition_range_values prv 
        ON pf.function_id = prv.function_id 
        AND p.partition_number = 
            prv.boundary_id + 1
    OUTER APPLY sys.dm_db_index_physical_stats(
        DB_ID(),
        t.object_id,
        i.index_id,
        p.partition_number,
        'LIMITED'
    ) idx;

    -- Analyze partition patterns
    WITH PartitionStats AS (
        SELECT 
            SchemaName,
            TableName,
            IndexName,
            PartitionNumber,
            PartitionFunction,
            BoundaryValue,
            RowCount,
            ReservedSpaceMB,
            UsedSpaceMB,
            DataCompressionDesc,
            FragmentationPercent,
            AVG(ReservedSpaceMB) OVER (
                PARTITION BY TableName
            ) as AvgPartitionSize,
            STDEV(ReservedSpaceMB) OVER (
                PARTITION BY TableName
            ) as StdevPartitionSize
        FROM dbo.PartitionMetrics
        WHERE CollectionTime = (
            SELECT MAX(CollectionTime) 
            FROM dbo.PartitionMetrics
        )
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        PartitionNumber,
        PartitionFunction,
        BoundaryValue,
        RowCount,
        ReservedSpaceMB / 1024.0 as ReservedSpaceGB,
        UsedSpaceMB / 1024.0 as UsedSpaceGB,
        DataCompressionDesc,
        FragmentationPercent,
        CASE 
            WHEN ReservedSpaceMB > @LargePartitionThresholdGB * 1024 
            THEN 'Oversized Partition'
            WHEN ABS(
                ReservedSpaceMB - AvgPartitionSize
            ) > AvgPartitionSize * (@SkewThresholdPercent / 100.0) 
            THEN 'Size Skew'
            WHEN FragmentationPercent > 30.0 
            THEN 'High Fragmentation'
            ELSE 'Normal'
        END as PartitionStatus,
        CASE 
            WHEN ReservedSpaceMB > @LargePartitionThresholdGB * 1024 
            THEN 'Consider:
                  1. Adjusting partition boundaries
                  2. Implementing sliding window
                  3. Reviewing partitioning strategy'
            WHEN ABS(
                ReservedSpaceMB - AvgPartitionSize
            ) > AvgPartitionSize * (@SkewThresholdPercent / 100.0) 
            THEN 'Review partition key distribution'
            WHEN FragmentationPercent > 30.0 
            THEN 'Schedule partition maintenance'
            ELSE 'No action needed'
        END as Recommendation
    FROM PartitionStats
    WHERE ReservedSpaceMB > @LargePartitionThresholdGB * 1024
    OR ABS(
        ReservedSpaceMB - AvgPartitionSize
    ) > AvgPartitionSize * (@SkewThresholdPercent / 100.0)
    OR FragmentationPercent > 30.0
    ORDER BY 
        CASE 
            WHEN ReservedSpaceMB > @LargePartitionThresholdGB * 1024 
            THEN 1
            WHEN FragmentationPercent > 30.0 THEN 2
            ELSE 3
        END,
        ReservedSpaceMB DESC;
END;
```

### Partition Alignment Analysis
```sql
CREATE PROCEDURE dbo.AnalyzePartitionAlignment
AS
BEGIN
    -- Analyze partition alignment
    WITH IndexPartitions AS (
        SELECT 
            OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
            t.name as TableName,
            i.name as IndexName,
            i.type_desc as IndexType,
            ps.name as PartitionScheme,
            pf.name as PartitionFunction,
            p.partition_number,
            p.rows,
            LAG(p.rows) OVER (
                PARTITION BY t.object_id, i.index_id 
                ORDER BY p.partition_number
            ) as PreviousPartitionRows,
            LEAD(p.rows) OVER (
                PARTITION BY t.object_id, i.index_id 
                ORDER BY p.partition_number
            ) as NextPartitionRows
        FROM sys.tables t
        JOIN sys.indexes i 
            ON t.object_id = i.object_id
        JOIN sys.partitions p 
            ON i.object_id = p.object_id 
            AND i.index_id = p.index_id
        JOIN sys.partition_schemes ps 
            ON i.data_space_id = ps.data_space_id
        JOIN sys.partition_functions pf 
            ON ps.function_id = pf.function_id
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        IndexType,
        PartitionScheme,
        PartitionFunction,
        partition_number,
        rows as PartitionRows,
        CASE 
            WHEN rows = 0 AND (
                PreviousPartitionRows > 0 OR 
                NextPartitionRows > 0
            ) THEN 'Empty Partition'
            WHEN rows > COALESCE(
                PreviousPartitionRows, 
                0
            ) * 2 OR rows > COALESCE(
                NextPartitionRows, 
                0
            ) * 2 
            THEN 'Partition Skew'
            ELSE 'Normal'
        END as AlignmentStatus,
        CASE 
            WHEN rows = 0 AND (
                PreviousPartitionRows > 0 OR 
                NextPartitionRows > 0
            ) THEN 'Consider merging empty partition'
            WHEN rows > COALESCE(
                PreviousPartitionRows, 
                0
            ) * 2 OR rows > COALESCE(
                NextPartitionRows, 
                0
            ) * 2 
            THEN 'Review partition boundaries'
            ELSE 'No action needed'
        END as Recommendation
    FROM IndexPartitions
    WHERE rows = 0 AND (
        PreviousPartitionRows > 0 OR 
        NextPartitionRows > 0
    )
    OR rows > COALESCE(PreviousPartitionRows, 0) * 2
    OR rows > COALESCE(NextPartitionRows, 0) * 2
    ORDER BY 
        CASE 
            WHEN rows = 0 THEN 1
            ELSE 2
        END,
        rows DESC;
END;
```

### Partition Maintenance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzePartitionMaintenance
    @RetentionDays int = 90,
    @HighFragmentationThreshold decimal(5,2) = 30.0
AS
BEGIN
    -- Analyze partition maintenance needs
    WITH MaintenanceMetrics AS (
        SELECT 
            OBJECT_SCHEMA_NAME(t.object_id) as SchemaName,
            t.name as TableName,
            i.name as IndexName,
            p.partition_number,
            p.rows,
            ps.used_page_count * 8.0 / 1024 as UsedSpaceMB,
            p.data_compression_desc,
            idx.avg_fragmentation_in_percent,
            CONVERT(
                datetime2,
                COALESCE(
                    boundary.value,
                    '1900-01-01'
                )
            ) as BoundaryDate,
            DATEDIFF(
                DAY,
                CONVERT(
                    datetime2,
                    COALESCE(
                        boundary.value,
                        '1900-01-01'
                    )
                ),
                GETDATE()
            ) as DaysOld
        FROM sys.tables t
        JOIN sys.indexes i 
            ON t.object_id = i.object_id
        JOIN sys.partitions p 
            ON i.object_id = p.object_id 
            AND i.index_id = p.index_id
        JOIN sys.dm_db_partition_stats ps 
            ON p.partition_id = ps.partition_id
        JOIN sys.partition_schemes pscheme 
            ON i.data_space_id = pscheme.data_space_id
        JOIN sys.partition_functions pfunc 
            ON pscheme.function_id = pfunc.function_id
        LEFT JOIN sys.partition_range_values boundary 
            ON pfunc.function_id = boundary.function_id 
            AND p.partition_number = 
                boundary.boundary_id + 1
        OUTER APPLY sys.dm_db_index_physical_stats(
            DB_ID(),
            t.object_id,
            i.index_id,
            p.partition_number,
            'LIMITED'
        ) idx
        WHERE boundary.value IS NOT NULL
    )
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        partition_number,
        rows as PartitionRows,
        UsedSpaceMB,
        data_compression_desc as CompressionType,
        avg_fragmentation_in_percent as FragmentationPercent,
        BoundaryDate,
        DaysOld,
        CASE 
            WHEN DaysOld > @RetentionDays 
            THEN 'Aged Partition'
            WHEN avg_fragmentation_in_percent > 
                 @HighFragmentationThreshold 
            THEN 'High Fragmentation'
            WHEN data_compression_desc = 'NONE' 
                 AND UsedSpaceMB > 1024 
            THEN 'Compression Candidate'
            ELSE 'Normal'
        END as MaintenanceStatus,
        CASE 
            WHEN DaysOld > @RetentionDays 
            THEN 'Consider:
                  1. Archiving data
                  2. Implementing sliding window
                  3. Reviewing retention policy'
            WHEN avg_fragmentation_in_percent > 
                 @HighFragmentationThreshold 
            THEN 'Schedule partition rebuild'
            WHEN data_compression_desc = 'NONE' 
                 AND UsedSpaceMB > 1024 
            THEN 'Evaluate compression options'
            ELSE 'No action needed'
        END as Recommendation
    FROM MaintenanceMetrics
    WHERE DaysOld > @RetentionDays
    OR avg_fragmentation_in_percent > @HighFragmentationThreshold
    OR (data_compression_desc = 'NONE' AND UsedSpaceMB > 1024)
    ORDER BY 
        CASE 
            WHEN DaysOld > @RetentionDays THEN 1
            WHEN avg_fragmentation_in_percent > 
                 @HighFragmentationThreshold THEN 2
            ELSE 3
        END,
        DaysOld DESC;
END;
```

This partition analysis framework provides comprehensive tools for:
1. Monitoring partition sizes and distribution patterns
2. Analyzing partition alignment and skew
3. Managing partition maintenance and aging
4. Optimizing compression and fragmentation

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
