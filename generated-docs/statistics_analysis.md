# SQL Server Statistics Analysis Framework

## Statistics Monitoring Framework

### Statistics Usage Tracking
```sql
CREATE TABLE dbo.StatisticsUsageHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseId int,
    ObjectId int,
    StatsId int,
    LastUpdated datetime2,
    ModificationCounter bigint,
    SamplingPercent decimal(5,2),
    UnfilteredRows bigint,
    RowsSampled bigint,
    StepsCount int,
    AverageRangeRows decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.TrackStatisticsUsage
    @StaleThresholdRows int = 10000,
    @LowSampleWarningPercent decimal(5,2) = 10.0
AS
BEGIN
    -- Capture current statistics state
    INSERT INTO dbo.StatisticsUsageHistory
    SELECT 
        DB_ID(),
        sp.object_id,
        sp.stats_id,
        sp.last_updated,
        sp.modification_counter,
        sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) as sampling_percent,
        sp.rows as unfiltered_rows,
        sp.rows_sampled,
        sp.steps as steps_count,
        sp.rows * 1.0 / NULLIF(sp.steps, 0) as average_range_rows,
        GETUTCDATE()
    FROM sys.stats s
    CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
    WHERE s.object_id > 100;  -- Exclude system objects

    -- Analyze statistics health
    WITH StatsMetrics AS (
        SELECT 
            DB_NAME(DatabaseId) as DatabaseName,
            OBJECT_NAME(ObjectId) as TableName,
            s.name as StatisticsName,
            LastUpdated,
            ModificationCounter,
            SamplingPercent,
            UnfilteredRows,
            RowsSampled,
            StepsCount,
            AverageRangeRows,
            DATEDIFF(HOUR, LastUpdated, GETUTCDATE()) as HoursSinceUpdate,
            ROW_NUMBER() OVER (
                PARTITION BY DatabaseId, ObjectId, StatsId 
                ORDER BY CollectionTime DESC
            ) as rn
        FROM dbo.StatisticsUsageHistory h
        JOIN sys.stats s 
            ON h.ObjectId = s.object_id 
            AND h.StatsId = s.stats_id
    )
    SELECT 
        DatabaseName,
        TableName,
        StatisticsName,
        LastUpdated,
        ModificationCounter,
        SamplingPercent,
        UnfilteredRows,
        RowsSampled,
        HoursSinceUpdate,
        CASE 
            WHEN ModificationCounter > @StaleThresholdRows 
            THEN 'Statistics Stale'
            WHEN SamplingPercent < @LowSampleWarningPercent 
            THEN 'Low Sample Size'
            WHEN HoursSinceUpdate > 168  -- 1 week
            THEN 'Old Statistics'
            ELSE 'Healthy'
        END as StatsHealth,
        CASE 
            WHEN ModificationCounter > @StaleThresholdRows 
            THEN 'UPDATE STATISTICS ' + QUOTENAME(TableName) + 
                 ' ' + QUOTENAME(StatisticsName) + 
                 ' WITH FULLSCAN'
            WHEN SamplingPercent < @LowSampleWarningPercent 
            THEN 'UPDATE STATISTICS ' + QUOTENAME(TableName) + 
                 ' ' + QUOTENAME(StatisticsName) + 
                 ' WITH SAMPLE ' + 
                 CAST(CEILING(@LowSampleWarningPercent) as varchar(3)) + 
                 ' PERCENT'
            ELSE 'No action needed'
        END as Recommendation
    FROM StatsMetrics
    WHERE rn = 1
    AND (
        ModificationCounter > @StaleThresholdRows
        OR SamplingPercent < @LowSampleWarningPercent
        OR HoursSinceUpdate > 168
    )
    ORDER BY ModificationCounter DESC;
END;
```

### Statistics Distribution Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeStatisticsDistribution
    @TableName sysname,
    @ColumnName sysname = NULL
AS
BEGIN
    -- Analyze column distribution
    WITH ColumnStats AS (
        SELECT 
            c.name as ColumnName,
            s.name as StatisticsName,
            sp.last_updated,
            sp.rows,
            sp.rows_sampled,
            sp.steps as histogram_steps,
            sp.unfiltered_rows,
            sp.modification_counter,
            CAST(rows_sampled * 100.0 / 
                 NULLIF(sp.unfiltered_rows, 0) as decimal(5,2)) 
                as sampling_percent
        FROM sys.stats s
        JOIN sys.stats_columns sc 
            ON s.object_id = sc.object_id 
            AND s.stats_id = sc.stats_id
        JOIN sys.columns c 
            ON sc.object_id = c.object_id 
            AND sc.column_id = c.column_id
        CROSS APPLY sys.dm_db_stats_properties(
            s.object_id, s.stats_id
        ) sp
        WHERE s.object_id = OBJECT_ID(@TableName)
        AND (@ColumnName IS NULL OR c.name = @ColumnName)
    )
    SELECT 
        cs.*,
        (
            SELECT TOP 1 sh.range_high_key
            FROM sys.dm_db_stats_histogram(
                OBJECT_ID(@TableName),
                stats_id
            ) sh
            ORDER BY range_high_key
        ) as min_value,
        (
            SELECT TOP 1 sh.range_high_key
            FROM sys.dm_db_stats_histogram(
                OBJECT_ID(@TableName),
                stats_id
            ) sh
            ORDER BY range_high_key DESC
        ) as max_value,
        CASE 
            WHEN modification_counter > rows * 0.20 
            THEN 'High Modifications'
            WHEN sampling_percent < 10.0 
            THEN 'Low Sample Size'
            WHEN histogram_steps < 100 
            THEN 'Few Histogram Steps'
            ELSE 'Normal'
        END as DistributionStatus,
        CASE 
            WHEN modification_counter > rows * 0.20 
            THEN 'Update statistics with larger sample'
            WHEN sampling_percent < 10.0 
            THEN 'Increase sampling percentage'
            ELSE 'No action needed'
        END as Recommendation
    FROM ColumnStats;

    -- Analyze histogram details
    IF @ColumnName IS NOT NULL
    BEGIN
        SELECT 
            s.name as StatisticsName,
            h.step_number,
            h.range_high_key,
            h.range_rows,
            h.equal_rows,
            h.distinct_range_rows,
            h.average_range_rows,
            CASE 
                WHEN h.average_range_rows > 100 
                THEN 'High Average Range'
                WHEN h.equal_rows > 1000 
                THEN 'High Equal Rows'
                ELSE 'Normal'
            END as StepPattern
        FROM sys.stats s
        JOIN sys.stats_columns sc 
            ON s.object_id = sc.object_id 
            AND s.stats_id = sc.stats_id
        JOIN sys.columns c 
            ON sc.object_id = c.object_id 
            AND sc.column_id = c.column_id
        CROSS APPLY sys.dm_db_stats_histogram(
            s.object_id, s.stats_id
        ) h
        WHERE s.object_id = OBJECT_ID(@TableName)
        AND c.name = @ColumnName
        ORDER BY h.step_number;
    END;
END;
```

### Statistics Dependency Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeStatisticsDependencies
    @TableName sysname = NULL
AS
BEGIN
    -- Analyze statistics dependencies
    WITH StatsDeps AS (
        SELECT 
            OBJECT_NAME(s.object_id) as TableName,
            s.name as StatisticsName,
            c.name as ColumnName,
            i.name as IndexName,
            s.auto_created,
            s.user_created,
            s.no_recompute,
            s.has_filter,
            s.filter_definition,
            sp.last_updated,
            sp.rows_sampled,
            sp.rows as total_rows,
            sp.modification_counter
        FROM sys.stats s
        JOIN sys.stats_columns sc 
            ON s.object_id = sc.object_id 
            AND s.stats_id = sc.stats_id
        JOIN sys.columns c 
            ON sc.object_id = c.object_id 
            AND sc.column_id = c.column_id
        LEFT JOIN sys.indexes i 
            ON s.object_id = i.object_id 
            AND s.stats_id = i.index_id
        CROSS APPLY sys.dm_db_stats_properties(
            s.object_id, s.stats_id
        ) sp
        WHERE @TableName IS NULL 
        OR OBJECT_NAME(s.object_id) = @TableName
    )
    SELECT 
        TableName,
        StatisticsName,
        STRING_AGG(ColumnName, ', ') as Columns,
        IndexName,
        auto_created,
        user_created,
        no_recompute,
        has_filter,
        filter_definition,
        last_updated,
        rows_sampled,
        total_rows,
        modification_counter,
        CASE 
            WHEN auto_created = 1 
            THEN 'Auto Created'
            WHEN IndexName IS NOT NULL 
            THEN 'Index Related'
            ELSE 'User Created'
        END as StatisticsType,
        CASE 
            WHEN modification_counter > total_rows * 0.20 
            THEN 'Statistics Stale'
            WHEN rows_sampled < total_rows * 0.10 
            THEN 'Low Sample Size'
            ELSE 'Healthy'
        END as StatisticsHealth
    FROM StatsDeps
    GROUP BY 
        TableName,
        StatisticsName,
        IndexName,
        auto_created,
        user_created,
        no_recompute,
        has_filter,
        filter_definition,
        last_updated,
        rows_sampled,
        total_rows,
        modification_counter
    ORDER BY 
        TableName,
        StatisticsName;
END;
```

### Statistics Maintenance Planning
```sql
CREATE PROCEDURE dbo.GenerateStatisticsMaintenance
    @TableName sysname = NULL,
    @SamplePercent decimal(5,2) = NULL,
    @ModificationThreshold decimal(5,2) = 20.0
AS
BEGIN
    -- Generate maintenance script
    WITH StatsMaintenance AS (
        SELECT 
            OBJECT_SCHEMA_NAME(s.object_id) as SchemaName,
            OBJECT_NAME(s.object_id) as TableName,
            s.name as StatisticsName,
            sp.modification_counter,
            sp.rows as total_rows,
            CAST(sp.modification_counter * 100.0 / 
                 NULLIF(sp.rows, 0) as decimal(5,2)) 
                as modification_percent,
            sp.rows_sampled,
            CAST(sp.rows_sampled * 100.0 / 
                 NULLIF(sp.rows, 0) as decimal(5,2)) 
                as current_sample_percent
        FROM sys.stats s
        CROSS APPLY sys.dm_db_stats_properties(
            s.object_id, s.stats_id
        ) sp
        WHERE (@TableName IS NULL 
               OR OBJECT_NAME(s.object_id) = @TableName)
        AND s.object_id > 100  -- Exclude system objects
    )
    SELECT 
        'UPDATE STATISTICS ' + 
        QUOTENAME(SchemaName) + '.' + 
        QUOTENAME(TableName) + ' ' + 
        QUOTENAME(StatisticsName) + 
        CASE 
            WHEN @SamplePercent IS NOT NULL 
            THEN ' WITH SAMPLE ' + 
                 CAST(@SamplePercent as varchar(5)) + ' PERCENT'
            WHEN total_rows > 1000000  -- Large tables
            THEN ' WITH SAMPLE 50 PERCENT'
            ELSE ' WITH FULLSCAN'
        END as MaintenanceCommand,
        SchemaName,
        TableName,
        StatisticsName,
        modification_percent,
        current_sample_percent,
        CASE 
            WHEN modification_percent > @ModificationThreshold 
            THEN 'High Priority'
            WHEN current_sample_percent < 10.0 
            THEN 'Medium Priority'
            ELSE 'Low Priority'
        END as MaintenancePriority
    FROM StatsMaintenance
    WHERE modification_percent > @ModificationThreshold
    OR current_sample_percent < 10.0
    ORDER BY 
        CASE 
            WHEN modification_percent > @ModificationThreshold 
            THEN 1
            WHEN current_sample_percent < 10.0 
            THEN 2
            ELSE 3
        END,
        modification_percent DESC;
END;
```

This statistics analysis framework provides comprehensive tools for:
1. Tracking and analyzing statistics usage patterns
2. Monitoring statistics health and staleness
3. Analyzing column value distributions
4. Managing statistics dependencies
5. Generating maintenance plans

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
