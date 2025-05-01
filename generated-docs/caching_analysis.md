# SQL Server Caching Analysis Framework

## Plan Cache Analysis

### Plan Cache Monitoring
```sql
CREATE TABLE dbo.PlanCacheHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    CacheType nvarchar(60),
    BucketCount int,
    UsedMemoryKB bigint,
    TotalMemoryKB bigint,
    EntriesCount int,
    HitRatio decimal(5,2),
    FlushCount int,
    RecompileCount int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorPlanCache
    @LowHitRatioThreshold decimal(5,2) = 80.0,
    @HighMemoryThresholdMB int = 1000
AS
BEGIN
    -- Capture plan cache metrics
    INSERT INTO dbo.PlanCacheHistory
    SELECT 
        pc.type as CacheType,
        pc.buckets_count,
        pc.used_memory_kb,
        pc.total_memory_kb,
        pc.entries_count,
        CAST(pc.cache_hit_ratio as decimal(5,2)),
        pc.cache_misses_count,
        (
            SELECT COUNT(*) 
            FROM sys.dm_exec_query_stats qs
            WHERE qs.execution_count = 1
        ) as SingleUseCount,
        GETUTCDATE()
    FROM sys.dm_os_memory_cache_counters pc
    WHERE pc.type LIKE '%PLAN%';

    -- Analyze cache health
    WITH CacheMetrics AS (
        SELECT 
            CacheType,
            UsedMemoryKB / 1024.0 as UsedMemoryMB,
            TotalMemoryKB / 1024.0 as TotalMemoryMB,
            EntriesCount,
            HitRatio,
            FlushCount,
            RecompileCount,
            LAG(EntriesCount) OVER (
                PARTITION BY CacheType 
                ORDER BY CollectionTime
            ) as PreviousEntries
        FROM dbo.PlanCacheHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        CacheType,
        UsedMemoryMB,
        TotalMemoryMB,
        EntriesCount,
        HitRatio,
        FlushCount,
        RecompileCount,
        CASE 
            WHEN HitRatio < @LowHitRatioThreshold 
            THEN 'Low Hit Ratio'
            WHEN UsedMemoryMB > @HighMemoryThresholdMB 
            THEN 'High Memory Usage'
            ELSE 'Healthy'
        END as CacheHealth,
        CASE 
            WHEN HitRatio < @LowHitRatioThreshold 
            THEN 'Review query patterns and parameterization'
            WHEN UsedMemoryMB > @HighMemoryThresholdMB 
            THEN 'Consider clearing single-use plans'
            ELSE 'No action needed'
        END as Recommendation
    FROM CacheMetrics
    WHERE HitRatio < @LowHitRatioThreshold
    OR UsedMemoryMB > @HighMemoryThresholdMB
    ORDER BY 
        CASE 
            WHEN HitRatio < @LowHitRatioThreshold THEN 1
            WHEN UsedMemoryMB > @HighMemoryThresholdMB THEN 2
            ELSE 3
        END;
END;
```

### Single-Use Plan Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeSingleUsePlans
    @MinimumSizeMB decimal(10,2) = 100.0
AS
BEGIN
    -- Analyze single-use plan impact
    SELECT TOP 50
        qt.text as QueryText,
        qs.creation_time,
        qs.last_execution_time,
        qs.execution_count,
        qs.total_worker_time / 1000000.0 as TotalCPUSeconds,
        qs.total_logical_reads / qs.execution_count as AvgLogicalReads,
        qs.total_physical_reads / qs.execution_count as AvgPhysicalReads,
        CAST(qp.query_plan as xml) as QueryPlan,
        CAST(
            (SELECT TOP 1 
                pa.used_pages * 8.0 / 1024 
             FROM sys.dm_exec_cached_plans cp
             CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
             CROSS APPLY sys.dm_os_buffer_descriptors pa
             WHERE cp.plan_handle = qs.plan_handle
            ) as decimal(10,2)
        ) as PlanSizeMB,
        CASE 
            WHEN qs.execution_count = 1 
                 AND qs.total_worker_time > 1000000  -- 1 second
            THEN 'High Cost Single Use'
            WHEN qs.execution_count = 1 
            THEN 'Single Use'
            ELSE 'Reused'
        END as PlanType
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE qs.execution_count = 1
    AND (
        SELECT pa.used_pages * 8.0 / 1024 
        FROM sys.dm_exec_cached_plans cp
        CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
        CROSS APPLY sys.dm_os_buffer_descriptors pa
        WHERE cp.plan_handle = qs.plan_handle
    ) > @MinimumSizeMB
    ORDER BY qs.total_worker_time DESC;
END;
```

### Buffer Cache Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeBufferCache
    @MinBufferMB decimal(10,2) = 100.0
AS
BEGIN
    -- Analyze buffer cache contents
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(p.object_id) as TableName,
        i.name as IndexName,
        COUNT(*) * 8 / 1024.0 as BufferSizeMB,
        COUNT(*) as PageCount,
        SUM(CAST(is_modified AS int)) as DirtyPages,
        AVG(read_microsec) / 1000.0 as AvgReadLatencyMs,
        CASE 
            WHEN COUNT(*) * 8 / 1024.0 > @MinBufferMB 
            THEN 'Large Buffer Usage'
            WHEN SUM(CAST(is_modified AS int)) > COUNT(*) * 0.2 
            THEN 'High Dirty Pages'
            ELSE 'Normal'
        END as BufferStatus
    FROM sys.dm_os_buffer_descriptors b
    JOIN sys.allocation_units a 
        ON b.allocation_unit_id = a.allocation_unit_id
    JOIN sys.partitions p 
        ON a.container_id = p.hobt_id
    LEFT JOIN sys.indexes i 
        ON p.object_id = i.object_id 
        AND p.index_id = i.index_id
    WHERE b.database_id = DB_ID()
    AND p.object_id > 100
    GROUP BY 
        database_id,
        p.object_id,
        i.name
    HAVING COUNT(*) * 8 / 1024.0 > @MinBufferMB
    ORDER BY COUNT(*) DESC;
END;
```

### Procedure Cache Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeProcedureCache
AS
BEGIN
    -- Analyze procedure cache usage
    SELECT TOP 50
        OBJECT_NAME(ps.object_id) as ProcedureName,
        ps.cached_time,
        ps.last_execution_time,
        ps.execution_count,
        ps.total_worker_time / 1000000.0 as TotalCPUSeconds,
        ps.total_elapsed_time / 1000000.0 as TotalDurationSeconds,
        ps.total_logical_reads / ps.execution_count as AvgLogicalReads,
        ps.total_physical_reads / ps.execution_count as AvgPhysicalReads,
        CAST(
            (SELECT TOP 1 
                pa.used_pages * 8.0 / 1024 
             FROM sys.dm_exec_cached_plans cp
             CROSS APPLY sys.dm_os_buffer_descriptors pa
             WHERE cp.plan_handle = ps.plan_handle
            ) as decimal(10,2)
        ) as CacheSizeMB,
        CASE 
            WHEN ps.execution_count = 1 
            THEN 'Single Execution'
            WHEN ps.execution_count > 1000 
            THEN 'Frequently Used'
            ELSE 'Moderate Usage'
        END as UsagePattern,
        CASE 
            WHEN ps.total_worker_time / ps.execution_count > 1000000 
            THEN 'High CPU Per Execution'
            WHEN ps.total_logical_reads / ps.execution_count > 1000 
            THEN 'High IO Per Execution'
            ELSE 'Normal'
        END as PerformancePattern
    FROM sys.dm_exec_procedure_stats ps
    WHERE ps.database_id = DB_ID()
    ORDER BY ps.total_worker_time DESC;

    -- Analyze cache churn
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(object_id) as ProcedureName,
        cached_time,
        last_execution_time,
        execution_count,
        DATEDIFF(MINUTE, cached_time, last_execution_time) as CacheLifetimeMinutes,
        CASE 
            WHEN DATEDIFF(MINUTE, cached_time, last_execution_time) < 10 
                 AND execution_count > 100 
            THEN 'High Churn'
            WHEN DATEDIFF(MINUTE, cached_time, last_execution_time) < 60 
            THEN 'Moderate Churn'
            ELSE 'Stable'
        END as ChurnPattern
    FROM sys.dm_exec_procedure_stats
    WHERE database_id = DB_ID()
    AND cached_time >= DATEADD(HOUR, -1, GETUTCDATE())
    ORDER BY cached_time DESC;
END;
```

This caching analysis framework provides comprehensive tools for:
1. Monitoring plan cache health and efficiency
2. Analyzing single-use plan impact
3. Buffer cache content analysis
4. Procedure cache usage monitoring
5. Cache churn detection and optimization

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
