# SQL Server Graph Database Analysis Framework

## Graph Query Performance Monitoring

### Node and Edge Analysis
```sql
CREATE TABLE dbo.GraphMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    NodeTableName sysname,
    EdgeTableName sysname,
    NodeCount bigint,
    EdgeCount bigint,
    AvgEdgesPerNode decimal(18,2),
    MaxEdgesPerNode int,
    QueryCount int,
    AvgTraversalTime decimal(18,2),
    MaxTraversalDepth int,
    IndexUsage decimal(5,2),
    MemoryGrantMB decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorGraphPerformance
    @HighTraversalThresholdMs decimal(18,2) = 1000.0,
    @HighMemoryThresholdMB decimal(18,2) = 1024.0
AS
BEGIN
    -- Capture graph metrics
    INSERT INTO dbo.GraphMetrics
    SELECT 
        n.name as NodeTableName,
        e.name as EdgeTableName,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = n.object_id
            AND ps.index_id <= 1
        ) as NodeCount,
        (
            SELECT SUM(row_count)
            FROM sys.dm_db_partition_stats ps
            WHERE ps.object_id = e.object_id
            AND ps.index_id <= 1
        ) as EdgeCount,
        CAST(
            (
                SELECT SUM(row_count)
                FROM sys.dm_db_partition_stats ps
                WHERE ps.object_id = e.object_id
                AND ps.index_id <= 1
            ) * 1.0 / NULLIF(
                (
                    SELECT SUM(row_count)
                    FROM sys.dm_db_partition_stats ps
                    WHERE ps.object_id = n.object_id
                    AND ps.index_id <= 1
                ),
                0
            ) as decimal(18,2)
        ) as AvgEdgesPerNode,
        (
            SELECT MAX(edge_count)
            FROM (
                SELECT COUNT(*) as edge_count
                FROM sys.tables e2
                WHERE e2.is_edge = 1
                GROUP BY e2.$from_id
            ) t
        ) as MaxEdgesPerNode,
        (
            SELECT COUNT(*)
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%MATCH%'
            AND st.text LIKE '%' + n.name + '%'
        ) as QueryCount,
        (
            SELECT AVG(qs.total_elapsed_time * 1.0 / qs.execution_count)
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%MATCH%'
            AND st.text LIKE '%' + n.name + '%'
        ) as AvgTraversalTime,
        (
            SELECT MAX(
                LEN(st.text) - 
                LEN(REPLACE(st.text, 'MATCH', ''))
            ) / LEN('MATCH')
            FROM sys.dm_exec_query_stats qs
            CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
            WHERE st.text LIKE '%MATCH%'
            AND st.text LIKE '%' + n.name + '%'
        ) as MaxTraversalDepth,
        (
            SELECT 100.0 * 
                   SUM(user_seeks + user_scans + user_lookups) /
                   NULLIF(
                       SUM(
                           user_seeks + user_scans + 
                           user_lookups + user_updates
                       ),
                       0
                   )
            FROM sys.dm_db_index_usage_stats us
            WHERE us.object_id IN (n.object_id, e.object_id)
        ) as IndexUsage,
        (
            SELECT MAX(granted_memory_kb) / 1024.0
            FROM sys.dm_exec_query_memory_grants qm
            CROSS APPLY sys.dm_exec_sql_text(qm.sql_handle) st
            WHERE st.text LIKE '%MATCH%'
            AND st.text LIKE '%' + n.name + '%'
        ) as MemoryGrantMB,
        GETUTCDATE()
    FROM sys.tables n
    JOIN sys.tables e ON n.is_node = 1 AND e.is_edge = 1;

    -- Analyze graph patterns
    WITH GraphMetrics AS (
        SELECT 
            NodeTableName,
            EdgeTableName,
            NodeCount,
            EdgeCount,
            AvgEdgesPerNode,
            MaxEdgesPerNode,
            QueryCount,
            AvgTraversalTime,
            MaxTraversalDepth,
            IndexUsage,
            MemoryGrantMB,
            LAG(EdgeCount) OVER (
                PARTITION BY NodeTableName, EdgeTableName 
                ORDER BY CollectionTime
            ) as PreviousEdgeCount
        FROM dbo.GraphMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        NodeTableName,
        EdgeTableName,
        NodeCount,
        EdgeCount,
        AvgEdgesPerNode,
        MaxEdgesPerNode,
        QueryCount,
        AvgTraversalTime,
        MaxTraversalDepth,
        IndexUsage,
        MemoryGrantMB,
        CASE 
            WHEN AvgTraversalTime > @HighTraversalThresholdMs 
                 AND MemoryGrantMB > @HighMemoryThresholdMB 
            THEN 'Critical Performance'
            WHEN AvgTraversalTime > @HighTraversalThresholdMs 
            THEN 'High Traversal Time'
            WHEN MemoryGrantMB > @HighMemoryThresholdMB 
            THEN 'High Memory Usage'
            WHEN MaxTraversalDepth > 5 
            THEN 'Deep Traversal'
            ELSE 'Normal'
        END as GraphStatus,
        CASE 
            WHEN AvgTraversalTime > @HighTraversalThresholdMs 
                 AND MemoryGrantMB > @HighMemoryThresholdMB 
            THEN 'Review:
                  1. Index strategy
                  2. Query patterns
                  3. Graph structure'
            WHEN AvgTraversalTime > @HighTraversalThresholdMs 
            THEN 'Optimize traversal paths'
            WHEN MemoryGrantMB > @HighMemoryThresholdMB 
            THEN 'Review memory grants'
            WHEN MaxTraversalDepth > 5 
            THEN 'Analyze traversal depth'
            ELSE 'No action needed'
        END as Recommendation
    FROM GraphMetrics
    WHERE AvgTraversalTime > @HighTraversalThresholdMs
    OR MemoryGrantMB > @HighMemoryThresholdMB
    OR MaxTraversalDepth > 5
    OR EdgeCount > COALESCE(PreviousEdgeCount, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN AvgTraversalTime > @HighTraversalThresholdMs 
                 AND MemoryGrantMB > @HighMemoryThresholdMB THEN 1
            WHEN AvgTraversalTime > @HighTraversalThresholdMs THEN 2
            ELSE 3
        END,
        AvgTraversalTime DESC;
END;
```

### Pattern Matching Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeGraphPatterns
AS
BEGIN
    -- Analyze MATCH pattern performance
    SELECT 
        DB_NAME(qt.dbid) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        qs.execution_count,
        qs.total_elapsed_time * 1.0 / 
            qs.execution_count as AvgExecutionTimeMs,
        qs.total_worker_time * 1.0 / 
            qs.execution_count as AvgCPUTimeMs,
        qs.total_logical_reads * 1.0 / 
            qs.execution_count as AvgLogicalReads,
        qs.total_physical_reads * 1.0 / 
            qs.execution_count as AvgPhysicalReads,
        qs.min_rows,
        qs.max_rows,
        qp.query_plan,
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 
            THEN 'Long Running'
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 
            THEN 'High I/O'
            WHEN qs.execution_count > 1000 
            THEN 'Frequently Used'
            ELSE 'Normal'
        END as PatternStatus,
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 
            THEN 'Review traversal patterns'
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 
            THEN 'Optimize index usage'
            WHEN qs.execution_count > 1000 
            THEN 'Monitor performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE qt.text LIKE '%MATCH%'
    AND qs.total_elapsed_time * 1.0 / qs.execution_count > 1000
    OR qs.total_logical_reads * 1.0 / qs.execution_count > 10000
    ORDER BY 
        CASE 
            WHEN qs.total_elapsed_time * 1.0 / 
                 qs.execution_count > 1000 THEN 1
            WHEN qs.total_logical_reads * 1.0 / 
                 qs.execution_count > 10000 THEN 2
            ELSE 3
        END,
        qs.total_elapsed_time DESC;
END;
```

### Graph Index Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeGraphIndexes
AS
BEGIN
    -- Analyze graph index usage
    SELECT 
        OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
        OBJECT_NAME(i.object_id) as TableName,
        i.name as IndexName,
        i.type_desc as IndexType,
        us.user_seeks,
        us.user_scans,
        us.user_lookups,
        us.user_updates,
        us.last_user_seek,
        us.last_user_scan,
        ps.row_count,
        ps.used_page_count * 8.0 / 1024 as IndexSizeMB,
        CAST(100.0 * (us.user_seeks + us.user_scans + us.user_lookups) /
             NULLIF(us.user_seeks + us.user_scans + us.user_lookups + 
                    us.user_updates, 0
             ) as decimal(5,2)) as IndexEfficiency,
        CASE 
            WHEN us.user_seeks = 0 
                 AND us.user_scans = 0 
                 AND us.user_lookups = 0 
            THEN 'Unused'
            WHEN us.user_scans > us.user_seeks * 2 
            THEN 'Scan Heavy'
            WHEN us.user_updates > (us.user_seeks + us.user_scans + 
                                  us.user_lookups) * 10 
            THEN 'Update Heavy'
            ELSE 'Normal'
        END as IndexStatus,
        CASE 
            WHEN us.user_seeks = 0 
                 AND us.user_scans = 0 
                 AND us.user_lookups = 0 
            THEN 'Consider removing index'
            WHEN us.user_scans > us.user_seeks * 2 
            THEN 'Review index design'
            WHEN us.user_updates > (us.user_seeks + us.user_scans + 
                                  us.user_lookups) * 10 
            THEN 'Monitor update impact'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.indexes i
    JOIN sys.objects o 
        ON i.object_id = o.object_id
    LEFT JOIN sys.dm_db_index_usage_stats us
        ON i.object_id = us.object_id
        AND i.index_id = us.index_id
    JOIN sys.dm_db_partition_stats ps
        ON i.object_id = ps.object_id
        AND i.index_id = ps.index_id
    WHERE o.is_node = 1 OR o.is_edge = 1
    AND i.index_id > 0  -- Exclude heaps
    AND (
        us.user_seeks = 0 
        AND us.user_scans = 0 
        AND us.user_lookups = 0
        OR us.user_scans > us.user_seeks * 2
        OR us.user_updates > (us.user_seeks + us.user_scans + 
                            us.user_lookups) * 10
    )
    ORDER BY 
        CASE 
            WHEN us.user_seeks = 0 
                 AND us.user_scans = 0 
                 AND us.user_lookups = 0 THEN 1
            WHEN us.user_scans > us.user_seeks * 2 THEN 2
            ELSE 3
        END,
        IndexSizeMB DESC;
END;
```

This Graph Database analysis framework provides comprehensive tools for:
1. Monitoring node and edge table performance
2. Analyzing MATCH pattern efficiency
3. Tracking graph index utilization
4. Optimizing graph traversal operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
