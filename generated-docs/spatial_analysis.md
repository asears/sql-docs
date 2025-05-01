# SQL Server Spatial Data Analysis Framework

## Spatial Query Performance Monitoring

### Spatial Index Analysis
```sql
CREATE TABLE dbo.SpatialIndexMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    IndexName sysname,
    SpatialColumnName sysname,
    CellsPerObject int,
    BoundingBoxLevel int,
    GridDensity int,
    IndexedObjects bigint,
    AverageCellsUsed decimal(18,2),
    UsageCount bigint,
    ScanCount bigint,
    SeekCount bigint,
    LastUsedTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorSpatialIndexes
    @HighScanRatioThreshold decimal(5,2) = 80.0,
    @LowUtilizationThreshold decimal(5,2) = 20.0
AS
BEGIN
    -- Capture spatial index metrics
    INSERT INTO dbo.SpatialIndexMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
        OBJECT_NAME(i.object_id) as TableName,
        i.name as IndexName,
        c.name as SpatialColumnName,
        si.cells_per_object,
        si.level_1_grid,
        si.tessellation_scheme as GridDensity,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = i.object_id
            AND ps.index_id = i.index_id
        ) as IndexedObjects,
        CAST(
            (
                SELECT AVG(CAST(cells_covered as float))
                FROM sys.dm_db_missing_index_details mid
                WHERE mid.object_id = i.object_id
            ) as decimal(18,2)
        ) as AverageCellsUsed,
        ius.user_seeks + ius.user_scans as UsageCount,
        ius.user_scans as ScanCount,
        ius.user_seeks as SeekCount,
        ius.last_user_lookup as LastUsedTime,
        GETUTCDATE()
    FROM sys.indexes i
    JOIN sys.columns c ON i.object_id = c.object_id
    JOIN sys.spatial_index_tessellations si 
        ON i.object_id = si.object_id 
        AND i.index_id = si.index_id
    LEFT JOIN sys.dm_db_index_usage_stats ius
        ON i.object_id = ius.object_id 
        AND i.index_id = ius.index_id
    WHERE i.type = 4;  -- Spatial index

    -- Analyze spatial index patterns
    WITH IndexMetrics AS (
        SELECT 
            DatabaseName,
            SchemaName,
            TableName,
            IndexName,
            SpatialColumnName,
            CellsPerObject,
            BoundingBoxLevel,
            GridDensity,
            IndexedObjects,
            AverageCellsUsed,
            UsageCount,
            ScanCount,
            SeekCount,
            CASE 
                WHEN UsageCount = 0 THEN 0
                ELSE CAST(
                    ScanCount * 100.0 / 
                    NULLIF(UsageCount, 0) as decimal(5,2)
                )
            END as ScanRatio,
            CASE 
                WHEN IndexedObjects = 0 THEN 0
                ELSE CAST(
                    SeekCount * 100.0 / 
                    NULLIF(IndexedObjects, 0) as decimal(5,2)
                )
            END as UtilizationRate
        FROM dbo.SpatialIndexMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -24, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        SchemaName,
        TableName,
        IndexName,
        SpatialColumnName,
        CellsPerObject,
        BoundingBoxLevel,
        GridDensity,
        IndexedObjects,
        AverageCellsUsed,
        UsageCount,
        ScanCount,
        SeekCount,
        ScanRatio,
        UtilizationRate,
        CASE 
            WHEN ScanRatio > @HighScanRatioThreshold 
                 AND UtilizationRate < @LowUtilizationThreshold 
            THEN 'Critical Performance'
            WHEN ScanRatio > @HighScanRatioThreshold 
            THEN 'High Scan Rate'
            WHEN UtilizationRate < @LowUtilizationThreshold 
            THEN 'Low Utilization'
            WHEN UsageCount = 0 
            THEN 'Unused Index'
            ELSE 'Normal'
        END as IndexStatus,
        CASE 
            WHEN ScanRatio > @HighScanRatioThreshold 
                 AND UtilizationRate < @LowUtilizationThreshold 
            THEN 'Review:
                  1. Grid density
                  2. Cells per object
                  3. Query patterns'
            WHEN ScanRatio > @HighScanRatioThreshold 
            THEN 'Optimize grid configuration'
            WHEN UtilizationRate < @LowUtilizationThreshold 
            THEN 'Review index usage'
            WHEN UsageCount = 0 
            THEN 'Consider removing index'
            ELSE 'No action needed'
        END as Recommendation
    FROM IndexMetrics
    WHERE ScanRatio > @HighScanRatioThreshold
    OR UtilizationRate < @LowUtilizationThreshold
    OR UsageCount = 0
    ORDER BY 
        CASE 
            WHEN ScanRatio > @HighScanRatioThreshold 
                 AND UtilizationRate < @LowUtilizationThreshold THEN 1
            WHEN ScanRatio > @HighScanRatioThreshold THEN 2
            ELSE 3
        END,
        ScanRatio DESC;
END;
```

### Spatial Query Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeSpatialQueries
AS
BEGIN
    -- Analyze spatial query patterns
    SELECT 
        q.query_id,
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        AVG(rs.avg_cpu_time) as AvgCPUTimeMs,
        AVG(rs.avg_logical_io_reads) as AvgLogicalReads,
        COUNT(
            CASE 
                WHEN p.query_plan.exist(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    //p:RelOp[@PhysicalOp="Spatial Index Scan"]'
                ) = 1 THEN 1 
            END
        ) as SpatialScanCount,
        COUNT(
            CASE 
                WHEN p.query_plan.exist(
                    'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                    //p:RelOp[@PhysicalOp="Spatial Index Seek"]'
                ) = 1 THEN 1 
            END
        ) as SpatialSeekCount,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'High Impact'
            WHEN COUNT(
                CASE 
                    WHEN p.query_plan.exist(
                        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                        //p:RelOp[@PhysicalOp="Spatial Index Scan"]'
                    ) = 1 THEN 1 
                END
            ) > COUNT(
                CASE 
                    WHEN p.query_plan.exist(
                        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                        //p:RelOp[@PhysicalOp="Spatial Index Seek"]'
                    ) = 1 THEN 1 
                END
            ) 
            THEN 'Scan Heavy'
            ELSE 'Normal'
        END as QueryPattern,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(rs.avg_duration) > 1000 
            THEN 'Optimize spatial operations'
            WHEN COUNT(
                CASE 
                    WHEN p.query_plan.exist(
                        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                        //p:RelOp[@PhysicalOp="Spatial Index Scan"]'
                    ) = 1 THEN 1 
                END
            ) > COUNT(
                CASE 
                    WHEN p.query_plan.exist(
                        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
                        //p:RelOp[@PhysicalOp="Spatial Index Seek"]'
                    ) = 1 THEN 1 
                END
            ) 
            THEN 'Review index strategy'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE qt.query_sql_text LIKE '%geography%'
    OR qt.query_sql_text LIKE '%geometry%'
    OR p.query_plan.exist(
        'declare namespace p="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
        //p:RelOp[@PhysicalOp[contains(., "Spatial")]]'
    ) = 1
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
            ELSE 2
        END,
        AvgDurationMs DESC;
END;
```

### Spatial Method Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeSpatialMethods
AS
BEGIN
    -- Analyze spatial method usage patterns
    SELECT 
        OBJECT_SCHEMA_NAME(m.object_id) as SchemaName,
        OBJECT_NAME(m.object_id) as ObjectName,
        m.definition as MethodDefinition,
        COUNT(*) as UsageCount,
        AVG(qs.total_elapsed_time * 1.0 / 
            qs.execution_count) as AvgDurationMs,
        SUM(qs.total_worker_time) / 1000.0 as TotalCPUTimeMs,
        MAX(qs.max_elapsed_time) / 1000.0 as MaxDurationMs,
        COUNT(
            DISTINCT SUBSTRING(
                qt.text,
                (qs.statement_start_offset/2) + 1,
                ((
                    CASE 
                        qs.statement_end_offset
                        WHEN -1 
                        THEN DATALENGTH(qt.text)
                        ELSE qs.statement_end_offset
                    END - qs.statement_start_offset
                )/2) + 1
            )
        ) as UniquePatterns,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(
                    qs.total_elapsed_time * 1.0 / 
                    qs.execution_count
                 ) > 1000000 
            THEN 'High Impact'
            WHEN COUNT(*) > 1000 
            THEN 'Frequently Used'
            WHEN AVG(
                qs.total_elapsed_time * 1.0 / 
                qs.execution_count
            ) > 1000000 
            THEN 'Long Running'
            ELSE 'Normal'
        END as MethodPattern,
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(
                    qs.total_elapsed_time * 1.0 / 
                    qs.execution_count
                 ) > 1000000 
            THEN 'Review method implementation'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor performance'
            WHEN AVG(
                qs.total_elapsed_time * 1.0 / 
                qs.execution_count
            ) > 1000000 
            THEN 'Optimize method logic'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.sql_modules m
    JOIN sys.dm_exec_query_stats qs
        ON OBJECT_NAME(m.object_id) = 
           OBJECT_NAME(
               OBJECT_ID(
                   SUBSTRING(
                       qt.text,
                       (qs.statement_start_offset/2) + 1,
                       ((
                           CASE 
                               qs.statement_end_offset
                               WHEN -1 
                               THEN DATALENGTH(qt.text)
                               ELSE qs.statement_end_offset
                           END - qs.statement_start_offset
                       )/2) + 1
                   )
               )
           )
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    WHERE m.definition LIKE '%geography%'
    OR m.definition LIKE '%geometry%'
    GROUP BY 
        m.object_id,
        m.definition
    HAVING COUNT(*) > 1000
    OR AVG(
        qs.total_elapsed_time * 1.0 / 
        qs.execution_count
    ) > 1000000
    ORDER BY 
        CASE 
            WHEN COUNT(*) > 1000 
                 AND AVG(
                    qs.total_elapsed_time * 1.0 / 
                    qs.execution_count
                 ) > 1000000 THEN 1
            WHEN COUNT(*) > 1000 THEN 2
            ELSE 3
        END,
        UsageCount DESC;
END;
```

This Spatial Data Analysis framework provides comprehensive tools for:
1. Monitoring spatial index performance and utilization
2. Analyzing spatial query patterns and optimization opportunities
3. Tracking spatial method usage and performance
4. Optimizing spatial operations and indexing strategies

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
