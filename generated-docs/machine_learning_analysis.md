# SQL Server Machine Learning Services Analysis Framework

## Machine Learning Script Monitoring

### Script Execution Analysis
```sql
CREATE TABLE dbo.MLScriptMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    Language nvarchar(32),
    ScriptId uniqueidentifier,
    DatabaseName sysname,
    ObjectName sysname,
    ExecutionCount int,
    TotalExecutionTimeMs bigint,
    TotalCPUTimeMs bigint,
    TotalMemoryMB decimal(18,2),
    DataVolumeMB decimal(18,2),
    ErrorCount int,
    LastExecutionTime datetime2,
    ResourceUtilization decimal(5,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorMLScriptPerformance
    @HighExecutionThresholdMs decimal(18,2) = 10000.0,
    @HighMemoryThresholdMB decimal(18,2) = 1024.0
AS
BEGIN
    -- Capture ML script execution metrics
    INSERT INTO dbo.MLScriptMetrics
    SELECT 
        ers.language,
        ers.script_id,
        DB_NAME(qt.dbid) as DatabaseName,
        OBJECT_NAME(qt.objectid, qt.dbid) as ObjectName,
        COUNT(*) as ExecutionCount,
        SUM(ers.total_execution_time_ms) as TotalExecutionTimeMs,
        SUM(ers.total_cpu_time_ms) as TotalCPUTimeMs,
        MAX(ers.peak_working_set_mb) as TotalMemoryMB,
        SUM(ers.total_data_processed) / 1048576.0 as DataVolumeMB,
        COUNT(
            CASE 
                WHEN ers.error_code IS NOT NULL 
                THEN 1 
            END
        ) as ErrorCount,
        MAX(ers.end_time) as LastExecutionTime,
        AVG(ers.resource_utilization) as ResourceUtilization,
        GETUTCDATE()
    FROM sys.dm_external_script_requests ers
    CROSS APPLY sys.dm_exec_sql_text(ers.sql_handle) qt
    GROUP BY 
        ers.language,
        ers.script_id,
        DB_NAME(qt.dbid),
        OBJECT_NAME(qt.objectid, qt.dbid);

    -- Analyze script execution patterns
    WITH ScriptMetrics AS (
        SELECT 
            Language,
            DatabaseName,
            ObjectName,
            ExecutionCount,
            TotalExecutionTimeMs / NULLIF(ExecutionCount, 0) 
                as AvgExecutionTimeMs,
            TotalCPUTimeMs / NULLIF(ExecutionCount, 0) 
                as AvgCPUTimeMs,
            TotalMemoryMB,
            DataVolumeMB,
            ErrorCount,
            ResourceUtilization,
            LAG(TotalMemoryMB) OVER (
                PARTITION BY ScriptId 
                ORDER BY CollectionTime
            ) as PreviousMemoryMB
        FROM dbo.MLScriptMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        Language,
        DatabaseName,
        ObjectName,
        ExecutionCount,
        AvgExecutionTimeMs,
        AvgCPUTimeMs,
        TotalMemoryMB,
        DataVolumeMB,
        ErrorCount,
        ResourceUtilization,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Script Errors'
            WHEN AvgExecutionTimeMs > @HighExecutionThresholdMs 
            THEN 'Long Execution'
            WHEN TotalMemoryMB > @HighMemoryThresholdMB 
            THEN 'High Memory'
            WHEN ResourceUtilization > 80 
            THEN 'High Resource Usage'
            ELSE 'Normal'
        END as ScriptStatus,
        CASE 
            WHEN ErrorCount > 0 
            THEN 'Debug script errors'
            WHEN AvgExecutionTimeMs > @HighExecutionThresholdMs 
            THEN 'Optimize script performance'
            WHEN TotalMemoryMB > @HighMemoryThresholdMB 
            THEN 'Review memory management'
            WHEN ResourceUtilization > 80 
            THEN 'Check resource constraints'
            ELSE 'No action needed'
        END as Recommendation
    FROM ScriptMetrics
    WHERE ErrorCount > 0
    OR AvgExecutionTimeMs > @HighExecutionThresholdMs
    OR TotalMemoryMB > @HighMemoryThresholdMB
    OR ResourceUtilization > 80
    ORDER BY 
        CASE 
            WHEN ErrorCount > 0 THEN 1
            WHEN AvgExecutionTimeMs > @HighExecutionThresholdMs THEN 2
            WHEN TotalMemoryMB > @HighMemoryThresholdMB THEN 3
            ELSE 4
        END,
        AvgExecutionTimeMs DESC;
END;
```

### Package Usage Analysis
```sql
CREATE PROCEDURE dbo.AnalyzePackageUsage
AS
BEGIN
    -- Analyze package utilization patterns
    SELECT 
        ep.language,
        ep.package_name,
        ep.package_version,
        COUNT(DISTINCT ers.script_id) as ScriptsUsingPackage,
        COUNT(*) as ExecutionCount,
        AVG(ers.total_execution_time_ms) as AvgExecutionTimeMs,
        MAX(ers.peak_working_set_mb) as PeakMemoryMB,
        COUNT(
            CASE 
                WHEN ers.error_code IS NOT NULL 
                THEN 1 
            END
        ) as ErrorCount,
        STRING_AGG(
            DISTINCT ers.error_message, 
            ' | '
        ) as ErrorMessages,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 
            THEN 'Package Errors'
            WHEN AVG(ers.total_execution_time_ms) > 10000 
            THEN 'High Latency'
            WHEN COUNT(*) > 1000 
            THEN 'High Usage'
            ELSE 'Normal'
        END as PackageStatus,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 
            THEN 'Review package compatibility'
            WHEN AVG(ers.total_execution_time_ms) > 10000 
            THEN 'Optimize package usage'
            WHEN COUNT(*) > 1000 
            THEN 'Monitor package performance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_external_script_execution_packages ep
    JOIN sys.dm_external_script_requests ers 
        ON ep.execution_id = ers.execution_id
    GROUP BY 
        ep.language,
        ep.package_name,
        ep.package_version
    HAVING COUNT(
        CASE 
            WHEN ers.error_code IS NOT NULL 
            THEN 1 
        END
    ) > 0
    OR AVG(ers.total_execution_time_ms) > 10000
    OR COUNT(*) > 1000
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 THEN 1
            WHEN COUNT(*) > 1000 THEN 2
            ELSE 3
        END,
        ExecutionCount DESC;
END;
```

### Resource Monitoring
```sql
CREATE PROCEDURE dbo.MonitorMLResources
AS
BEGIN
    -- Monitor ML resource utilization
    SELECT 
        ers.language,
        ers.counter_name,
        COUNT(*) as ScriptCount,
        AVG(ers.counter_value) as AvgValue,
        MAX(ers.counter_value) as MaxValue,
        MIN(ers.counter_value) as MinValue,
        STDEV(ers.counter_value) as StdDevValue,
        COUNT(
            CASE 
                WHEN ers.error_code IS NOT NULL 
                THEN 1 
            END
        ) as ErrorCount,
        AVG(ers.resource_utilization) as AvgResourceUtilization,
        CASE 
            WHEN ers.counter_name LIKE '%memory%'
                 AND MAX(ers.counter_value) > 1024 
            THEN 'High Memory'
            WHEN ers.counter_name LIKE '%cpu%'
                 AND AVG(ers.resource_utilization) > 80 
            THEN 'High CPU'
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 
            THEN 'Resource Errors'
            ELSE 'Normal'
        END as ResourceStatus,
        CASE 
            WHEN ers.counter_name LIKE '%memory%'
                 AND MAX(ers.counter_value) > 1024 
            THEN 'Review memory allocation'
            WHEN ers.counter_name LIKE '%cpu%'
                 AND AVG(ers.resource_utilization) > 80 
            THEN 'Check CPU utilization'
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 
            THEN 'Investigate resource errors'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_external_script_resource_stats ers
    GROUP BY 
        ers.language,
        ers.counter_name
    HAVING MAX(ers.counter_value) > 
        CASE 
            WHEN ers.counter_name LIKE '%memory%' THEN 1024
            WHEN ers.counter_name LIKE '%cpu%' THEN 80
            ELSE 0
        END
    OR COUNT(
        CASE 
            WHEN ers.error_code IS NOT NULL 
            THEN 1 
        END
    ) > 0
    ORDER BY 
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN ers.error_code IS NOT NULL 
                    THEN 1 
                END
            ) > 0 THEN 1
            ELSE 2
        END,
        MaxValue DESC;
END;
```

This Machine Learning Services analysis framework provides comprehensive tools for:
1. Monitoring script execution performance and resource usage
2. Analyzing package utilization and errors
3. Tracking resource consumption by language and script
4. Optimizing ML workloads

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
