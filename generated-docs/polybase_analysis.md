# SQL Server PolyBase Analysis Framework

## PolyBase Performance Monitoring

### External Data Source Analysis
```sql
CREATE TABLE dbo.PolyBaseMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DataSourceName nvarchar(128),
    SourceType nvarchar(60),
    QueryCount int,
    TotalBytes bigint,
    TotalRows bigint,
    ExecutionTimeMs bigint,
    NetworkLatencyMs decimal(18,2),
    CPUTimeMs bigint,
    MemoryGrantMB decimal(18,2),
    ErrorCount int,
    LastQueryTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorPolyBasePerformance
    @HighLatencyThresholdMs decimal(18,2) = 5000.0,
    @LowThroughputThresholdMBps decimal(18,2) = 10.0
AS
BEGIN
    -- Capture PolyBase metrics
    INSERT INTO dbo.PolyBaseMetrics
    SELECT 
        eds.name as DataSourceName,
        eds.type_desc as SourceType,
        COUNT(*) as QueryCount,
        SUM(dqs.total_bytes_processed) as TotalBytes,
        SUM(dqs.total_rows_processed) as TotalRows,
        SUM(dqs.execution_time_ms) as ExecutionTimeMs,
        AVG(dqs.compute_pool_wait_time_ms) as NetworkLatencyMs,
        SUM(dqs.cpu_time_ms) as CPUTimeMs,
        MAX(dqs.memory_grant_mb) as MemoryGrantMB,
        COUNT(
            CASE 
                WHEN dqs.error_id IS NOT NULL 
                THEN 1 
            END
        ) as ErrorCount,
        MAX(dqs.end_time) as LastQueryTime,
        GETUTCDATE()
    FROM sys.external_data_sources eds
    JOIN sys.dm_exec_distributed_requests dr 
        ON eds.location = dr.data_source_uri
    JOIN sys.dm_exec_distributed_sql_requests dqs 
        ON dr.execution_id = dqs.execution_id
    GROUP BY 
        eds.name,
        eds.type_desc;

    -- Analyze PolyBase patterns
    WITH PolyBaseMetrics AS (
        SELECT 
            DataSourceName,
            SourceType,
            QueryCount,
            TotalBytes / 1048576.0 as TotalMB,
            TotalRows,
            ExecutionTimeMs / 1000.0 as ExecutionTimeSec,
            NetworkLatencyMs,
            CPUTimeMs / 1000.0 as CPUTimeSec,
            MemoryGrantMB,
            ErrorCount,
            CASE 
                WHEN ExecutionTimeMs = 0 THEN 0
                ELSE (TotalBytes / 1048576.0) / 
                     (ExecutionTimeMs / 1000.0)
            END as ThroughputMBps
        FROM dbo.PolyBaseMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DataSourceName,
        SourceType,
        QueryCount,
        TotalMB,
        TotalRows,
        ExecutionTimeSec,
        NetworkLatencyMs,
        CPUTimeSec,
        MemoryGrantMB,
        ErrorCount,
        ThroughputMBps,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Errors Present'
            WHEN NetworkLatencyMs > @HighLatencyThresholdMs 
            THEN 'High Latency'
            WHEN ThroughputMBps < @LowThroughputThresholdMBps 
            THEN 'Low Throughput'
            ELSE 'Normal'
        END as PerformanceStatus,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Investigate external data access errors'
            WHEN NetworkLatencyMs > @HighLatencyThresholdMs 
            THEN 'Review network connectivity'
            WHEN ThroughputMBps < @LowThroughputThresholdMBps 
            THEN 'Optimize data movement'
            ELSE 'No action needed'
        END as Recommendation
    FROM PolyBaseMetrics
    WHERE ErrorCount > 0
    OR NetworkLatencyMs > @HighLatencyThresholdMs
    OR ThroughputMBps < @LowThroughputThresholdMBps
    ORDER BY 
        CASE 
            WHEN ErrorCount > 0 THEN 1
            WHEN NetworkLatencyMs > @HighLatencyThresholdMs THEN 2
            ELSE 3
        END,
        NetworkLatencyMs DESC;
END;
```

### Compute Pool Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeComputePools
AS
BEGIN
    -- Analyze compute pool utilization
    SELECT 
        cp.compute_pool_name,
        cp.state_desc as PoolState,
        cp.min_cpu_count,
        cp.max_cpu_count,
        cp.min_memory_mb,
        cp.max_memory_mb,
        COUNT(DISTINCT dr.execution_id) as ActiveQueries,
        AVG(dr.total_elapsed_time) as AvgElapsedTimeMs,
        MAX(dr.total_elapsed_time) as MaxElapsedTimeMs,
        SUM(dr.total_bytes_processed) / 1048576.0 as TotalDataMB,
        AVG(dr.cpu_time) as AvgCPUTimeMs,
        MAX(dr.memory_usage_mb) as PeakMemoryMB,
        COUNT(
            CASE 
                WHEN dr.status = 'FAILED' 
                THEN 1 
            END
        ) as FailedQueries,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dr.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 
            THEN 'Query Failures'
            WHEN AVG(dr.total_elapsed_time) > 60000 
            THEN 'High Latency'
            WHEN COUNT(DISTINCT dr.execution_id) > 
                 cp.max_cpu_count * 2 
            THEN 'High Concurrency'
            ELSE 'Normal'
        END as PoolStatus,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dr.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 
            THEN 'Review failed queries'
            WHEN AVG(dr.total_elapsed_time) > 60000 
            THEN 'Optimize query performance'
            WHEN COUNT(DISTINCT dr.execution_id) > 
                 cp.max_cpu_count * 2 
            THEN 'Consider scaling pool resources'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_compute_pools cp
    LEFT JOIN sys.dm_exec_distributed_requests dr 
        ON cp.compute_pool_id = dr.compute_pool_id
    GROUP BY 
        cp.compute_pool_name,
        cp.state_desc,
        cp.min_cpu_count,
        cp.max_cpu_count,
        cp.min_memory_mb,
        cp.max_memory_mb
    HAVING COUNT(
        CASE 
            WHEN dr.status = 'FAILED' 
            THEN 1 
        END
    ) > 0
    OR AVG(dr.total_elapsed_time) > 60000
    OR COUNT(DISTINCT dr.execution_id) > cp.max_cpu_count * 2
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dr.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 THEN 1
            WHEN COUNT(DISTINCT dr.execution_id) > 
                 cp.max_cpu_count * 2 THEN 2
            ELSE 3
        END,
        ActiveQueries DESC;
END;
```

### Query Distribution Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeQueryDistribution
AS
BEGIN
    -- Analyze distributed query patterns
    SELECT 
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        dqs.operation_type,
        COUNT(*) as ExecutionCount,
        AVG(dqs.total_elapsed_time) as AvgElapsedTimeMs,
        MAX(dqs.total_elapsed_time) as MaxElapsedTimeMs,
        SUM(dqs.total_bytes_processed) / 1048576.0 as TotalDataMB,
        AVG(dqs.cpu_time) as AvgCPUTimeMs,
        MAX(dqs.memory_grant_mb) as MaxMemoryGrantMB,
        COUNT(
            CASE 
                WHEN dqs.status = 'FAILED' 
                THEN 1 
            END
        ) as FailedExecutions,
        AVG(dqs.total_bytes_processed / 
            NULLIF(dqs.total_elapsed_time, 0)
        ) / 1048576.0 as AvgThroughputMBps,
        STRING_AGG(
            DISTINCT dqs.error_id::varchar(10), 
            ', '
        ) as ErrorIds,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dqs.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 
            THEN 'Failures Present'
            WHEN AVG(dqs.total_elapsed_time) > 60000 
            THEN 'High Duration'
            WHEN AVG(dqs.memory_grant_mb) > 1024 
            THEN 'High Memory'
            ELSE 'Normal'
        END as QueryPattern,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dqs.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 
            THEN 'Review error patterns'
            WHEN AVG(dqs.total_elapsed_time) > 60000 
            THEN 'Optimize query performance'
            WHEN AVG(dqs.memory_grant_mb) > 1024 
            THEN 'Review memory usage'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_exec_distributed_sql_requests dqs
    JOIN sys.dm_exec_distributed_requests dr 
        ON dqs.execution_id = dr.execution_id
    CROSS APPLY sys.dm_exec_sql_text(dr.sql_handle) qt
    GROUP BY 
        OBJECT_NAME(qt.objectid, qt.dbid),
        dqs.operation_type
    HAVING COUNT(
        CASE 
            WHEN dqs.status = 'FAILED' 
            THEN 1 
        END
    ) > 0
    OR AVG(dqs.total_elapsed_time) > 60000
    OR AVG(dqs.memory_grant_mb) > 1024
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN dqs.status = 'FAILED' 
                    THEN 1 
                END
            ) > 0 THEN 1
            WHEN AVG(dqs.total_elapsed_time) > 60000 THEN 2
            ELSE 3
        END,
        ExecutionCount DESC;
END;
```

This PolyBase analysis framework provides comprehensive tools for:
1. Monitoring external data source performance
2. Analyzing compute pool utilization
3. Tracking distributed query patterns
4. Optimizing data movement operations

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
