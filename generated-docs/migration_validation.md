# Migration Validation Framework

## Data Consistency Verification

### Checksum Validation
```sql
CREATE TABLE dbo.ValidationResults
(
    ValidationId bigint IDENTITY(1,1) PRIMARY KEY,
    TableName sysname,
    PartitionNumber int,
    SourceChecksum binary(32),
    TargetChecksum binary(32),
    RowCount bigint,
    IsConsistent bit,
    ValidationTime datetime2,
    Duration int,  -- seconds
    ErrorMessage nvarchar(max)
);

CREATE PROCEDURE dbo.ValidateTableData
    @TableName sysname,
    @BatchSize int = 100000,
    @ParallelDegree int = 4
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL nvarchar(max);
    DECLARE @PartitionCount int;
    
    -- Get partition count
    SELECT @PartitionCount = COUNT(DISTINCT partition_number)
    FROM sys.partitions
    WHERE object_id = OBJECT_ID(@TableName);
    
    -- Process each partition
    WITH PartitionList AS (
        SELECT DISTINCT partition_number
        FROM sys.partitions
        WHERE object_id = OBJECT_ID(@TableName)
    )
    SELECT @SQL = STRING_AGG(
        'INSERT INTO dbo.ValidationResults
        (TableName, PartitionNumber, SourceChecksum, 
         TargetChecksum, RowCount, IsConsistent, 
         ValidationTime, Duration)
        SELECT 
            ''' + @TableName + ''',
            ' + CAST(partition_number as varchar(10)) + ',
            HASHBYTES(''SHA2_256'', 
                (SELECT * 
                 FROM ' + @TableName + ' WITH (NOLOCK)
                 WHERE $PARTITION.PF_' + @TableName + 
                '(PartitionKey) = ' + 
                CAST(partition_number as varchar(10)) + 
                ' FOR XML RAW, BINARY BASE64)),
            HASHBYTES(''SHA2_256'', 
                (SELECT * 
                 FROM [Target].' + @TableName + ' WITH (NOLOCK)
                 WHERE $PARTITION.PF_' + @TableName + 
                '(PartitionKey) = ' + 
                CAST(partition_number as varchar(10)) + 
                ' FOR XML RAW, BINARY BASE64)),
            COUNT(*),
            CASE WHEN SourceChecksum = TargetChecksum 
                 THEN 1 ELSE 0 END,
            GETUTCDATE(),
            DATEDIFF(SECOND, GETUTCDATE(), GETUTCDATE())',
        '; '
    )
    FROM PartitionList;
    
    -- Execute validation
    EXEC sp_executesql @SQL;
    
    -- Report results
    SELECT 
        TableName,
        SUM(CASE WHEN IsConsistent = 1 THEN 1 ELSE 0 END) as ConsistentPartitions,
        SUM(CASE WHEN IsConsistent = 0 THEN 1 ELSE 0 END) as InconsistentPartitions,
        SUM(RowCount) as TotalRows,
        MAX(Duration) as MaxDurationSeconds
    FROM dbo.ValidationResults
    WHERE TableName = @TableName
    GROUP BY TableName;
END;
```

### Row-Level Validation
```sql
CREATE PROCEDURE dbo.ValidateRowLevel
    @TableName sysname,
    @SampleSize int = 1000,
    @KeyColumn sysname
AS
BEGIN
    SET NOCOUNT ON;
    
    CREATE TABLE #Differences
    (
        RowId bigint IDENTITY(1,1),
        KeyValue sql_variant,
        ColumnName sysname,
        SourceValue sql_variant,
        TargetValue sql_variant,
        DifferenceType varchar(20)  -- Missing, Different, Extra
    );
    
    -- Get random sample of keys
    CREATE TABLE #SampleKeys
    (
        KeyValue sql_variant
    );
    
    DECLARE @SQL nvarchar(max) = '
    INSERT INTO #SampleKeys
    SELECT TOP (@SampleSize) ' + QUOTENAME(@KeyColumn) + '
    FROM ' + QUOTENAME(@TableName) + ' WITH (NOLOCK)
    ORDER BY NEWID()';
    
    EXEC sp_executesql @SQL, N'@SampleSize int', @SampleSize;
    
    -- Compare each column
    SELECT @SQL = STRING_AGG(
        'INSERT INTO #Differences
        SELECT 
            s.KeyValue,
            ''' + name + ''' as ColumnName,
            source.' + QUOTENAME(name) + ',
            target.' + QUOTENAME(name) + ',
            CASE 
                WHEN target.' + QUOTENAME(name) + ' IS NULL 
                     AND source.' + QUOTENAME(name) + ' IS NOT NULL
                THEN ''Missing''
                WHEN source.' + QUOTENAME(name) + ' IS NULL 
                     AND target.' + QUOTENAME(name) + ' IS NOT NULL
                THEN ''Extra''
                ELSE ''Different''
            END
        FROM #SampleKeys s
        LEFT JOIN ' + @TableName + ' source WITH (NOLOCK)
            ON s.KeyValue = source.' + @KeyColumn + '
        LEFT JOIN [Target].' + @TableName + ' target WITH (NOLOCK)
            ON s.KeyValue = target.' + @KeyColumn + '
        WHERE source.' + QUOTENAME(name) + ' <> target.' + 
        QUOTENAME(name) + '
        OR (source.' + QUOTENAME(name) + ' IS NULL AND target.' + 
           QUOTENAME(name) + ' IS NOT NULL)
        OR (source.' + QUOTENAME(name) + ' IS NOT NULL AND target.' + 
           QUOTENAME(name) + ' IS NULL)',
        CHAR(13)
    )
    FROM sys.columns
    WHERE object_id = OBJECT_ID(@TableName);
    
    EXEC sp_executesql @SQL;
    
    -- Report differences
    SELECT 
        ColumnName,
        COUNT(*) as DifferenceCount,
        COUNT(*) * 100.0 / @SampleSize as PercentDifferent,
        STRING_AGG(
            CAST(KeyValue as varchar(100)) + ': ' +
            COALESCE(CAST(SourceValue as varchar(100)), 'NULL') + 
            ' -> ' +
            COALESCE(CAST(TargetValue as varchar(100)), 'NULL'),
            '; '
        ) as Examples
    FROM #Differences
    GROUP BY ColumnName
    ORDER BY DifferenceCount DESC;
END;
```

## Schema Validation

### Object Comparison
```sql
CREATE PROCEDURE dbo.ValidateSchemaObjects
    @TableName sysname
AS
BEGIN
    -- Compare table structure
    SELECT 
        c.name as ColumnName,
        TYPE_NAME(c.system_type_id) as DataType,
        c.max_length,
        c.precision,
        c.scale,
        c.is_nullable,
        c.is_identity,
        CASE 
            WHEN source.column_id IS NULL THEN 'Missing in Source'
            WHEN target.column_id IS NULL THEN 'Missing in Target'
            WHEN source.system_type_id <> target.system_type_id 
            THEN 'Type Mismatch'
            WHEN source.max_length <> target.max_length 
            THEN 'Length Mismatch'
            WHEN source.precision <> target.precision 
            THEN 'Precision Mismatch'
            WHEN source.scale <> target.scale 
            THEN 'Scale Mismatch'
            WHEN source.is_nullable <> target.is_nullable 
            THEN 'Nullability Mismatch'
            ELSE 'OK'
        END as Status
    FROM sys.columns source
    FULL OUTER JOIN [Target].sys.columns target
        ON source.name = target.name
        AND source.object_id = OBJECT_ID(@TableName)
        AND target.object_id = OBJECT_ID('[Target].' + @TableName)
    WHERE source.object_id = OBJECT_ID(@TableName)
    OR target.object_id = OBJECT_ID('[Target].' + @TableName);
    
    -- Compare indexes
    SELECT 
        i.name as IndexName,
        i.type_desc as IndexType,
        i.is_unique,
        i.is_primary_key,
        STRING_AGG(c.name, ', ') as IndexColumns,
        CASE 
            WHEN source.index_id IS NULL THEN 'Missing in Source'
            WHEN target.index_id IS NULL THEN 'Missing in Target'
            WHEN source_cols <> target_cols THEN 'Column Mismatch'
            ELSE 'OK'
        END as Status
    FROM sys.indexes source
    FULL OUTER JOIN [Target].sys.indexes target
        ON source.name = target.name
        AND source.object_id = OBJECT_ID(@TableName)
        AND target.object_id = OBJECT_ID('[Target].' + @TableName)
    JOIN sys.index_columns ic
        ON source.object_id = ic.object_id
        AND source.index_id = ic.index_id
    JOIN sys.columns c
        ON ic.object_id = c.object_id
        AND ic.column_id = c.column_id
    GROUP BY 
        i.name,
        i.type_desc,
        i.is_unique,
        i.is_primary_key,
        source.index_id,
        target.index_id,
        source_cols,
        target_cols;
END;
```

## Performance Validation

### Query Plan Comparison
```sql
CREATE PROCEDURE dbo.ValidateQueryPlans
    @MinimumExecutions int = 100
AS
BEGIN
    WITH SourcePlans AS (
        SELECT 
            qt.query_sql_text,
            qp.query_plan,
            rs.avg_duration as source_duration,
            rs.avg_cpu_time as source_cpu,
            rs.avg_logical_io_reads as source_reads
        FROM sys.query_store_query_text qt
        JOIN sys.query_store_query q
            ON qt.query_text_id = q.query_text_id
        JOIN sys.query_store_plan qp
            ON q.query_id = qp.query_id
        JOIN sys.query_store_runtime_stats rs
            ON qp.plan_id = rs.plan_id
        WHERE rs.count_executions >= @MinimumExecutions
    ),
    TargetPlans AS (
        SELECT 
            qt.query_sql_text,
            qp.query_plan,
            rs.avg_duration as target_duration,
            rs.avg_cpu_time as target_cpu,
            rs.avg_logical_io_reads as target_reads
        FROM [Target].sys.query_store_query_text qt
        JOIN [Target].sys.query_store_query q
            ON qt.query_text_id = q.query_text_id
        JOIN [Target].sys.query_store_plan qp
            ON q.query_id = qp.query_id
        JOIN [Target].sys.query_store_runtime_stats rs
            ON qp.plan_id = rs.plan_id
        WHERE rs.count_executions >= @MinimumExecutions
    )
    SELECT 
        sp.query_sql_text,
        sp.source_duration,
        tp.target_duration,
        ((tp.target_duration - sp.source_duration) * 100.0) / 
            sp.source_duration as DurationDiffPercent,
        sp.source_cpu,
        tp.target_cpu,
        ((tp.target_cpu - sp.source_cpu) * 100.0) / 
            sp.source_cpu as CPUDiffPercent,
        sp.source_reads,
        tp.target_reads,
        ((tp.target_reads - sp.source_reads) * 100.0) / 
            sp.source_reads as ReadsDiffPercent,
        CASE 
            WHEN tp.target_duration > sp.source_duration * 1.2 
            THEN 'Performance Regression'
            WHEN tp.target_duration < sp.source_duration * 0.8 
            THEN 'Performance Improvement'
            ELSE 'Similar Performance'
        END as PerformanceImpact
    FROM SourcePlans sp
    JOIN TargetPlans tp
        ON sp.query_sql_text = tp.query_sql_text
    ORDER BY ABS(((tp.target_duration - sp.source_duration) * 100.0) / 
                 sp.source_duration) DESC;
END;
```

This validation framework provides comprehensive verification of the migration process, ensuring data consistency, schema accuracy, and performance parity between the source and target environments.
