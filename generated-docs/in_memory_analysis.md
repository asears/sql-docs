# SQL Server In-Memory OLTP Analysis Framework

## Memory-Optimized Table Monitoring

### Memory Usage Analysis
```sql
CREATE TABLE dbo.MemoryOptimizedMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    TableName sysname,
    AllocatedMemoryKB bigint,
    UsedMemoryKB bigint,
    RowCount bigint,
    IndexCount int,
    MemoryConsumerCount int,
    XtpControllerCount int,
    LastAccessTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorMemoryOptimizedTables
    @MemoryThresholdMB decimal(10,2) = 1024.0,
    @HighMemoryUtilization decimal(5,2) = 90.0
AS
BEGIN
    -- Capture memory usage metrics
    INSERT INTO dbo.MemoryOptimizedMetrics
    SELECT 
        DB_NAME(t.database_id) as DatabaseName,
        OBJECT_NAME(t.object_id) as TableName,
        SUM(m.allocated_bytes) / 1024 as AllocatedMemoryKB,
        SUM(m.used_bytes) / 1024 as UsedMemoryKB,
        SUM(m.row_count) as RowCount,
        (
            SELECT COUNT(*) 
            FROM sys.indexes i
            WHERE i.object_id = t.object_id
        ) as IndexCount,
        COUNT(DISTINCT m.memory_consumer_id) as MemoryConsumerCount,
        COUNT(DISTINCT m.xtp_transaction_id) as XtpControllerCount,
        MAX(m.last_access_time) as LastAccessTime,
        GETUTCDATE()
    FROM sys.dm_db_xtp_table_memory_stats m
    JOIN sys.tables t 
        ON m.object_id = t.object_id
    WHERE t.is_memory_optimized = 1
    GROUP BY t.database_id, t.object_id;

    -- Analyze memory usage patterns
    WITH MemoryMetrics AS (
        SELECT 
            DatabaseName,
            TableName,
            AllocatedMemoryKB / 1024.0 as AllocatedMemoryMB,
            UsedMemoryKB / 1024.0 as UsedMemoryMB,
            RowCount,
            IndexCount,
            MemoryConsumerCount,
            XtpControllerCount,
            LastAccessTime,
            CAST(UsedMemoryKB * 100.0 / 
                 NULLIF(AllocatedMemoryKB, 0) as decimal(5,2)) 
                as MemoryUtilization,
            LAG(UsedMemoryKB) OVER (
                PARTITION BY DatabaseName, TableName 
                ORDER BY CollectionTime
            ) as PreviousUsedMemoryKB
        FROM dbo.MemoryOptimizedMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        TableName,
        AllocatedMemoryMB,
        UsedMemoryMB,
        RowCount,
        IndexCount,
        MemoryConsumerCount,
        MemoryUtilization,
        CASE 
            WHEN AllocatedMemoryMB > @MemoryThresholdMB 
                 AND MemoryUtilization > @HighMemoryUtilization 
            THEN 'High Memory Pressure'
            WHEN AllocatedMemoryMB > @MemoryThresholdMB 
            THEN 'High Memory Usage'
            WHEN MemoryUtilization > @HighMemoryUtilization 
            THEN 'High Utilization'
            ELSE 'Normal'
        END as MemoryStatus,
        CASE 
            WHEN AllocatedMemoryMB > @MemoryThresholdMB 
                 AND MemoryUtilization > @HighMemoryUtilization 
            THEN 'Consider:
                  1. Archiving older data
                  2. Reviewing memory allocation
                  3. Scaling out memory resources'
            WHEN AllocatedMemoryMB > @MemoryThresholdMB 
            THEN 'Review memory allocation strategy'
            WHEN MemoryUtilization > @HighMemoryUtilization 
            THEN 'Monitor for growth patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM MemoryMetrics
    WHERE AllocatedMemoryMB > @MemoryThresholdMB
    OR MemoryUtilization > @HighMemoryUtilization
    ORDER BY 
        CASE 
            WHEN AllocatedMemoryMB > @MemoryThresholdMB 
                 AND MemoryUtilization > @HighMemoryUtilization 
            THEN 1
            WHEN AllocatedMemoryMB > @MemoryThresholdMB THEN 2
            ELSE 3
        END,
        AllocatedMemoryMB DESC;
END;
```

### Native Compilation Performance
```sql
CREATE PROCEDURE dbo.AnalyzeNativeCompilation
AS
BEGIN
    -- Analyze natively compiled procedures
    SELECT 
        OBJECT_NAME(object_id) as ProcedureName,
        execution_count,
        total_worker_time / 1000000.0 as TotalCPUSeconds,
        (total_worker_time * 1.0 / execution_count) / 1000.0 
            as AvgCPUMilliseconds,
        total_elapsed_time / 1000000.0 as TotalDurationSeconds,
        (total_elapsed_time * 1.0 / execution_count) / 1000.0 
            as AvgDurationMilliseconds,
        last_execution_time,
        cached_time,
        CASE 
            WHEN execution_count = 1 
            THEN 'Single Execution'
            WHEN last_execution_time <= 
                 DATEADD(HOUR, -24, GETDATE()) 
            THEN 'Inactive'
            WHEN (total_worker_time * 1.0 / execution_count) > 
                 1000000  -- 1 second
            THEN 'High CPU'
            ELSE 'Normal'
        END as ExecutionPattern,
        CASE 
            WHEN execution_count = 1 
            THEN 'Monitor usage patterns'
            WHEN last_execution_time <= 
                 DATEADD(HOUR, -24, GETDATE()) 
            THEN 'Review procedure relevance'
            WHEN (total_worker_time * 1.0 / execution_count) > 
                 1000000 
            THEN 'Review procedure logic'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_procedure_stats
    WHERE is_natively_compiled = 1
    AND (
        execution_count = 1
        OR last_execution_time <= DATEADD(HOUR, -24, GETDATE())
        OR (total_worker_time * 1.0 / execution_count) > 1000000
    )
    ORDER BY 
        CASE 
            WHEN (total_worker_time * 1.0 / execution_count) > 
                 1000000 THEN 1
            WHEN execution_count = 1 THEN 2
            ELSE 3
        END,
        total_worker_time DESC;
END;
```

### Garbage Collection Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeGarbageCollection
    @HighWatermarkPercent decimal(5,2) = 80.0
AS
BEGIN
    -- Analyze garbage collection metrics
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        object_id as TableObjectId,
        OBJECT_NAME(object_id) as TableName,
        memory_consumer_id,
        memory_consumer_type_desc,
        allocated_bytes / 1048576.0 as AllocatedMB,
        used_bytes / 1048576.0 as UsedMB,
        CAST(used_bytes * 100.0 / 
             NULLIF(allocated_bytes, 0) as decimal(5,2)) 
            as UtilizationPercent,
        allocation_count,
        gc_scan_count,
        gc_update_scan_count,
        CASE 
            WHEN used_bytes * 100.0 / 
                 NULLIF(allocated_bytes, 0) > @HighWatermarkPercent 
            THEN 'High GC Pressure'
            WHEN gc_scan_count > 1000 
            THEN 'Frequent GC'
            ELSE 'Normal'
        END as GCStatus,
        CASE 
            WHEN used_bytes * 100.0 / 
                 NULLIF(allocated_bytes, 0) > @HighWatermarkPercent 
            THEN 'Consider:
                  1. Increasing memory allocation
                  2. Archiving older data
                  3. Reviewing data retention'
            WHEN gc_scan_count > 1000 
            THEN 'Review update patterns'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_db_xtp_memory_consumers
    WHERE (
        used_bytes * 100.0 / NULLIF(allocated_bytes, 0) > 
            @HighWatermarkPercent
        OR gc_scan_count > 1000
    )
    ORDER BY 
        CASE 
            WHEN used_bytes * 100.0 / 
                 NULLIF(allocated_bytes, 0) > @HighWatermarkPercent 
            THEN 1
            ELSE 2
        END,
        used_bytes DESC;
END;
```

### Memory Grant Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeMemoryGrants
AS
BEGIN
    -- Analyze memory grants for in-memory operations
    SELECT 
        s.session_id,
        DB_NAME(r.database_id) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        mg.requested_memory_kb / 1024.0 as RequestedMemoryMB,
        mg.granted_memory_kb / 1024.0 as GrantedMemoryMB,
        mg.required_memory_kb / 1024.0 as RequiredMemoryMB,
        mg.used_memory_kb / 1024.0 as UsedMemoryMB,
        mg.max_used_memory_kb / 1024.0 as MaxUsedMemoryMB,
        mg.query_cost as EstimatedCost,
        r.wait_type,
        r.wait_time / 1000.0 as WaitTimeSeconds,
        CASE 
            WHEN mg.requested_memory_kb > mg.granted_memory_kb 
            THEN 'Memory Pressure'
            WHEN mg.granted_memory_kb > 
                 mg.max_used_memory_kb * 2 
            THEN 'Over-allocation'
            WHEN r.wait_type LIKE 'MEMORY%' 
            THEN 'Memory Wait'
            ELSE 'Normal'
        END as GrantStatus,
        CASE 
            WHEN mg.requested_memory_kb > mg.granted_memory_kb 
            THEN 'Review memory configuration'
            WHEN mg.granted_memory_kb > 
                 mg.max_used_memory_kb * 2 
            THEN 'Optimize memory grant estimation'
            WHEN r.wait_type LIKE 'MEMORY%' 
            THEN 'Investigate memory waits'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_query_memory_grants mg
    JOIN sys.dm_exec_sessions s 
        ON mg.session_id = s.session_id
    JOIN sys.dm_exec_requests r 
        ON mg.session_id = r.session_id
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) qt
    WHERE (
        mg.requested_memory_kb > mg.granted_memory_kb
        OR mg.granted_memory_kb > mg.max_used_memory_kb * 2
        OR r.wait_type LIKE 'MEMORY%'
    )
    ORDER BY 
        CASE 
            WHEN mg.requested_memory_kb > mg.granted_memory_kb THEN 1
            WHEN r.wait_type LIKE 'MEMORY%' THEN 2
            ELSE 3
        END,
        mg.requested_memory_kb DESC;
END;
```

This In-Memory OLTP analysis framework provides comprehensive tools for:
1. Monitoring memory usage and utilization patterns
2. Analyzing native compilation performance
3. Tracking garbage collection efficiency
4. Managing memory grants for in-memory operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
