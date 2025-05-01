# SQL Server Database Scoped Configuration Analysis Framework

## Database Configuration Monitoring

### Configuration Impact Analysis
```sql
CREATE TABLE dbo.ConfigurationMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    ConfigurationName nvarchar(60),
    ConfigurationValue sql_variant,
    PreviousValue sql_variant,
    ChangeTime datetime2,
    QueryCount int,
    AvgDurationMs decimal(18,2),
    AvgCPUTimeMs decimal(18,2),
    AvgLogicalReads decimal(18,2),
    AvgMemoryGrantMB decimal(18,2),
    ImpactScore decimal(5,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorConfigurationImpact
    @HighImpactThreshold decimal(5,2) = 20.0,
    @MonitoringWindowHours int = 24
AS
BEGIN
    -- Capture configuration metrics
    INSERT INTO dbo.ConfigurationMetrics
    SELECT 
        DB_NAME() as DatabaseName,
        sc.name as ConfigurationName,
        sc.value as ConfigurationValue,
        sc.value_for_secondary as PreviousValue,
        sc.modification_date as ChangeTime,
        COUNT(DISTINCT qs.query_id) as QueryCount,
        AVG(rs.avg_duration) / 1000.0 as AvgDurationMs,
        AVG(rs.avg_cpu_time) / 1000.0 as AvgCPUTimeMs,
        AVG(rs.avg_logical_io_reads) as AvgLogicalReads,
        AVG(rs.avg_query_max_used_memory) / 1024.0 as AvgMemoryGrantMB,
        CASE
            WHEN sc.modification_date >= DATEADD(HOUR, -@MonitoringWindowHours, GETUTCDATE())
            THEN (
                SELECT AVG(
                    CASE 
                        WHEN rs2.avg_duration > rs.avg_duration THEN 
                            ((rs2.avg_duration - rs.avg_duration) * 100.0 / 
                             NULLIF(rs.avg_duration, 0))
                        ELSE 0
                    END
                )
                FROM sys.query_store_runtime_stats rs2
                WHERE rs2.runtime_stats_interval_id < rs.runtime_stats_interval_id
                AND DATEDIFF(
                    HOUR,
                    rs2.last_execution_time,
                    sc.modification_date
                ) <= @MonitoringWindowHours
            )
            ELSE 0
        END as ImpactScore,
        GETUTCDATE()
    FROM sys.database_scoped_configurations sc
    CROSS APPLY (
        SELECT *
        FROM sys.query_store_runtime_stats rs
        JOIN sys.query_store_plan p ON rs.plan_id = p.plan_id
        WHERE rs.last_execution_time >= 
              DATEADD(HOUR, -@MonitoringWindowHours, GETUTCDATE())
    ) rs
    CROSS APPLY (
        SELECT DISTINCT query_id
        FROM sys.query_store_plan
        WHERE plan_id = rs.plan_id
    ) qs
    GROUP BY 
        sc.name,
        sc.value,
        sc.value_for_secondary,
        sc.modification_date;

    -- Analyze configuration impact patterns
    WITH ConfigMetrics AS (
        SELECT 
            DatabaseName,
            ConfigurationName,
            ConfigurationValue,
            PreviousValue,
            ChangeTime,
            QueryCount,
            AvgDurationMs,
            AvgCPUTimeMs,
            AvgLogicalReads,
            AvgMemoryGrantMB,
            ImpactScore,
            LAG(AvgDurationMs) OVER (
                PARTITION BY ConfigurationName 
                ORDER BY CollectionTime
            ) as PreviousAvgDuration
        FROM dbo.ConfigurationMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -@MonitoringWindowHours, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        ConfigurationName,
        ConfigurationValue,
        PreviousValue,
        QueryCount,
        AvgDurationMs,
        AvgCPUTimeMs,
        AvgLogicalReads,
        AvgMemoryGrantMB,
        ImpactScore,
        CASE 
            WHEN ImpactScore > @HighImpactThreshold 
            THEN 'High Impact'
            WHEN AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Performance Regression'
            WHEN DATEDIFF(
                HOUR,
                ChangeTime,
                GETUTCDATE()
            ) < @MonitoringWindowHours 
            THEN 'Recent Change'
            ELSE 'Normal'
        END as ConfigStatus,
        CASE 
            WHEN ImpactScore > @HighImpactThreshold 
            THEN 'Review configuration impact'
            WHEN AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5 
            THEN 'Investigate performance change'
            WHEN DATEDIFF(
                HOUR,
                ChangeTime,
                GETUTCDATE()
            ) < @MonitoringWindowHours 
            THEN 'Monitor for impact'
            ELSE 'No action needed'
        END as Recommendation
    FROM ConfigMetrics
    WHERE ImpactScore > @HighImpactThreshold
    OR AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5
    OR DATEDIFF(
        HOUR,
        ChangeTime,
        GETUTCDATE()
    ) < @MonitoringWindowHours
    ORDER BY 
        CASE 
            WHEN ImpactScore > @HighImpactThreshold THEN 1
            WHEN AvgDurationMs > COALESCE(PreviousAvgDuration, 0) * 1.5 THEN 2
            ELSE 3
        END,
        ImpactScore DESC;
END;
```

### Parameter Sensitive Configuration Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeParameterConfiguration
AS
BEGIN
    -- Analyze parameter sensitivity configuration impact
    SELECT 
        qt.query_sql_text,
        p.query_plan,
        COUNT(*) as ExecutionCount,
        AVG(rs.avg_duration) as AvgDurationMs,
        MIN(rs.min_duration) as MinDurationMs,
        MAX(rs.max_duration) as MaxDurationMs,
        COUNT(DISTINCT p.plan_id) as PlanCount,
        STRING_AGG(
            CAST(p.plan_id as varchar) + 
            ' (' + 
            CAST(rs.avg_duration as varchar) + 
            'ms)',
            ', '
        ) as PlanDurations,
        CASE 
            WHEN EXISTS (
                SELECT 1
                FROM sys.database_scoped_configurations
                WHERE name = 'PARAMETER_SENSITIVE_PLAN_OPTIMIZATION'
                AND value = 1
            ) THEN 'Enabled'
            ELSE 'Disabled'
        END as PSPStatus,
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 
            THEN 'High Plan Variation'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Variable Performance'
            ELSE 'Normal'
        END as ConfigurationPattern,
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 
            THEN 'Review parameter sensitivity'
            WHEN MAX(rs.max_duration) > 
                 MIN(rs.min_duration) * 2 
            THEN 'Analyze performance variance'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_query_text qt 
        ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE rs.last_execution_time >= 
          DATEADD(DAY, -7, GETUTCDATE())
    AND qt.query_sql_text LIKE N'%@%'
    GROUP BY 
        qt.query_sql_text,
        p.query_plan
    HAVING COUNT(DISTINCT p.plan_id) > 1
    AND (
        COUNT(DISTINCT p.plan_id) > 5
        OR MAX(rs.max_duration) > MIN(rs.min_duration) * 2
    )
    ORDER BY 
        CASE 
            WHEN COUNT(DISTINCT p.plan_id) > 5 THEN 1
            ELSE 2
        END,
        COUNT(DISTINCT p.plan_id) DESC;
END;
```

### Memory Configuration Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeMemoryConfiguration
AS
BEGIN
    -- Analyze memory-related configuration impact
    SELECT 
        DB_NAME() as DatabaseName,
        (
            SELECT value
            FROM sys.database_scoped_configurations
            WHERE name = 'MAXDOP'
        ) as MaxDOP,
        (
            SELECT value
            FROM sys.database_scoped_configurations
            WHERE name = 'MAX_GRANT_PERCENT'
        ) as MaxGrantPercent,
        COUNT(DISTINCT p.query_id) as UniqueQueries,
        AVG(rs.avg_query_max_used_memory) / 1024.0 as AvgMemoryGrantMB,
        MAX(rs.max_query_max_used_memory) / 1024.0 as MaxMemoryGrantMB,
        COUNT(
            CASE 
                WHEN rs.avg_query_max_used_memory > 102400 
                THEN 1 
            END
        ) as HighMemoryQueries,
        AVG(
            CASE 
                WHEN rs.avg_query_max_used_memory > 
                     rs.min_query_max_used_memory * 2 
                THEN 1 
                ELSE 0 
            END
        ) * 100 as MemoryVariancePercent,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN rs.avg_query_max_used_memory > 102400 
                    THEN 1 
                END
            ) > 10 
            THEN 'High Memory Usage'
            WHEN AVG(
                CASE 
                    WHEN rs.avg_query_max_used_memory > 
                         rs.min_query_max_used_memory * 2 
                    THEN 1 
                    ELSE 0 
                END
            ) * 100 > 20 
            THEN 'High Memory Variance'
            ELSE 'Normal'
        END as MemoryPattern,
        CASE 
            WHEN COUNT(
                CASE 
                    WHEN rs.avg_query_max_used_memory > 102400 
                    THEN 1 
                END
            ) > 10 
            THEN 'Review MAX_GRANT_PERCENT'
            WHEN AVG(
                CASE 
                    WHEN rs.avg_query_max_used_memory > 
                         rs.min_query_max_used_memory * 2 
                    THEN 1 
                    ELSE 0 
                END
            ) * 100 > 20 
            THEN 'Analyze memory grant feedback'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.query_store_query q
    JOIN sys.query_store_plan p 
        ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs 
        ON p.plan_id = rs.plan_id
    WHERE rs.last_execution_time >= 
          DATEADD(DAY, -1, GETUTCDATE())
    GROUP BY DB_NAME()
    HAVING COUNT(
        CASE 
            WHEN rs.avg_query_max_used_memory > 102400 
            THEN 1 
        END
    ) > 10
    OR AVG(
        CASE 
            WHEN rs.avg_query_max_used_memory > 
                 rs.min_query_max_used_memory * 2 
            THEN 1 
            ELSE 0 
        END
    ) * 100 > 20;
END;
```

This Database Scoped Configuration analysis framework provides comprehensive tools for:
1. Monitoring configuration change impact on performance
2. Analyzing parameter sensitivity configuration effectiveness
3. Tracking memory-related configuration patterns
4. Optimizing database-level settings

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
