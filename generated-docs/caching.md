# SQL Server Caching and Performance Management During Migration

## Memory Management Strategies

### Buffer Pool Extensions
```sql
-- Configure Buffer Pool Extension
ALTER SERVER CONFIGURATION
SET BUFFER POOL EXTENSION ON
(FILENAME = 'E:\BPE\bpe.bpe', SIZE = 102400 MB);

-- Monitor BPE usage
SELECT 
    BPE_CURRENT_SIZE_MB,
    BPE_AVAILABLE_MB
FROM sys.dm_os_buffer_pool_extension_configuration;
```

### Distributed Cache Management

1. Cache Invalidation Strategy
```sql
CREATE TABLE dbo.CacheInvalidation
(
    CacheKey nvarchar(900) PRIMARY KEY,
    LastInvalidated datetime2,
    InvalidationReason nvarchar(100),
    SourceServer nvarchar(128),
    IsProcessed bit DEFAULT 0
);

CREATE PROCEDURE dbo.InvalidateCache
    @CacheKey nvarchar(900),
    @Reason nvarchar(100)
AS
BEGIN
    INSERT INTO dbo.CacheInvalidation 
    (CacheKey, LastInvalidated, InvalidationReason, SourceServer)
    VALUES 
    (@CacheKey, GETUTCDATE(), @Reason, @@SERVERNAME);
END;
```

2. Memory-Optimized Cache Tables
```sql
CREATE TABLE dbo.DistributedCache
(
    CacheKey nvarchar(900) COLLATE Latin1_General_100_BIN2 
        PRIMARY KEY NONCLUSTERED,
    CacheValue varbinary(max),
    ExpirationTime datetime2,
    LastAccessed datetime2,
    AccessCount bigint,
    INDEX idx_expiration NONCLUSTERED (ExpirationTime)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY);

-- Cache management procedures
CREATE PROCEDURE dbo.UpsertCacheItem
    @Key nvarchar(900),
    @Value varbinary(max),
    @ExpirationMinutes int = 60
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC WITH
(
    TRANSACTION ISOLATION LEVEL = SNAPSHOT,
    LANGUAGE = N'us_english'
)
    MERGE dbo.DistributedCache AS target
    USING (SELECT @Key as CacheKey) AS source
    ON (target.CacheKey = source.CacheKey)
    WHEN MATCHED THEN
        UPDATE SET 
            CacheValue = @Value,
            ExpirationTime = DATEADD(MINUTE, @ExpirationMinutes, GETUTCDATE()),
            LastAccessed = GETUTCDATE(),
            AccessCount = AccessCount + 1
    WHEN NOT MATCHED THEN
        INSERT (CacheKey, CacheValue, ExpirationTime, LastAccessed, AccessCount)
        VALUES (@Key, @Value, 
                DATEADD(MINUTE, @ExpirationMinutes, GETUTCDATE()),
                GETUTCDATE(), 1);
END;
```

## Query Performance Optimization

### Dynamic Statistics Management
```sql
-- Create statistics monitoring
CREATE TABLE dbo.StatisticsMonitoring
(
    StatsId int IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    SchemaName sysname,
    TableName sysname,
    ColumnName sysname,
    StatsName sysname,
    LastUpdated datetime2,
    RowModifications bigint,
    SamplingRate float
);

-- Auto-update statistics procedure
CREATE PROCEDURE dbo.UpdateStatisticsAsync
    @TableName nvarchar(128),
    @SampleRate float = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL nvarchar(max);
    DECLARE @StatsName nvarchar(128);
    
    DECLARE stats_cursor CURSOR FOR
    SELECT name
    FROM sys.stats
    WHERE object_id = OBJECT_ID(@TableName);
    
    OPEN stats_cursor;
    FETCH NEXT FROM stats_cursor INTO @StatsName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = 'UPDATE STATISTICS ' + @TableName + 
                  '(' + @StatsName + ')';
        
        IF @SampleRate IS NOT NULL
            SET @SQL = @SQL + ' WITH SAMPLE ' + 
                      CAST(@SampleRate AS nvarchar(10)) + ' PERCENT';
        
        EXEC sp_executesql @SQL;
        
        -- Log statistics update
        INSERT INTO dbo.StatisticsMonitoring
        (DatabaseName, SchemaName, TableName, 
         StatsName, LastUpdated, SamplingRate)
        SELECT 
            DB_NAME(),
            OBJECT_SCHEMA_NAME(OBJECT_ID(@TableName)),
            OBJECT_NAME(OBJECT_ID(@TableName)),
            @StatsName,
            GETUTCDATE(),
            @SampleRate;
        
        FETCH NEXT FROM stats_cursor INTO @StatsName;
    END;
    
    CLOSE stats_cursor;
    DEALLOCATE stats_cursor;
END;
```

### Query Plan Management

1. Plan Baseline Collection
```sql
CREATE TABLE dbo.QueryPlanBaselines
(
    PlanId int IDENTITY(1,1) PRIMARY KEY,
    QueryHash binary(8),
    PlanHandle varbinary(64),
    QueryPlan xml,
    ExecutionStats xml,
    CaptureTime datetime2,
    IsApproved bit DEFAULT 0
);

-- Capture baseline procedure
CREATE PROCEDURE dbo.CapturePlanBaseline
    @DatabaseName sysname,
    @SchemaName sysname = NULL,
    @ObjectName sysname = NULL
AS
BEGIN
    INSERT INTO dbo.QueryPlanBaselines
    (QueryHash, PlanHandle, QueryPlan, 
     ExecutionStats, CaptureTime)
    SELECT 
        qs.query_hash,
        qs.plan_handle,
        qp.query_plan,
        (
            SELECT 
                execution_count,
                total_worker_time,
                total_elapsed_time,
                total_logical_reads,
                total_logical_writes
            FROM sys.dm_exec_query_stats
            WHERE plan_handle = qs.plan_handle
            FOR XML AUTO
        ),
        GETUTCDATE()
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
    WHERE DB_NAME(qp.dbid) = @DatabaseName
    AND (@SchemaName IS NULL OR 
         OBJECT_SCHEMA_NAME(qp.objectid, qp.dbid) = @SchemaName)
    AND (@ObjectName IS NULL OR 
         OBJECT_NAME(qp.objectid, qp.dbid) = @ObjectName);
END;
```

2. Plan Forcing Framework
```sql
CREATE TABLE dbo.PlanForcePolicy
(
    PolicyId int IDENTITY(1,1) PRIMARY KEY,
    QueryHash binary(8),
    ForcedPlanId int,
    IsEnabled bit DEFAULT 1,
    CreatedDate datetime2 DEFAULT GETUTCDATE(),
    LastModified datetime2,
    ModifiedBy nvarchar(128),
    CONSTRAINT FK_ForcedPlan 
        FOREIGN KEY (ForcedPlanId) 
        REFERENCES dbo.QueryPlanBaselines(PlanId)
);

-- Plan forcing procedure
CREATE PROCEDURE dbo.ApplyPlanForcing
    @QueryHash binary(8),
    @PlanId int
AS
BEGIN
    DECLARE @PlanHandle varbinary(64);
    DECLARE @QueryPlan xml;
    
    SELECT @PlanHandle = PlanHandle,
           @QueryPlan = QueryPlan
    FROM dbo.QueryPlanBaselines
    WHERE PlanId = @PlanId;
    
    IF @PlanHandle IS NOT NULL
    BEGIN
        -- Apply plan forcing hint
        DECLARE @SQL nvarchar(max) = '
        SELECT value
        FROM sys.dm_exec_plan_attributes(' + 
        CAST(@PlanHandle AS nvarchar(max)) + ')
        WHERE attribute = ''dbid''';
        
        EXEC sp_executesql @SQL;
        
        -- Log plan forcing
        INSERT INTO dbo.PlanForcePolicy
        (QueryHash, ForcedPlanId, ModifiedBy)
        VALUES
        (@QueryHash, @PlanId, SYSTEM_USER);
    END;
END;
```

## Cache Synchronization

### Cross-Server Cache Coordination
```sql
CREATE TABLE dbo.CacheCoordination
(
    CoordinationId int IDENTITY(1,1) PRIMARY KEY,
    ServerName nvarchar(128),
    CacheKey nvarchar(900),
    Operation char(1), -- 'I'nsert, 'U'pdate, 'D'elete
    OperationTime datetime2,
    Status tinyint DEFAULT 0
);

CREATE PROCEDURE dbo.SynchronizeCaches
    @SourceServer nvarchar(128),
    @TargetServer nvarchar(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify cache differences
    INSERT INTO dbo.CacheCoordination
    (ServerName, CacheKey, Operation, OperationTime)
    SELECT 
        @TargetServer,
        source.CacheKey,
        CASE 
            WHEN target.CacheKey IS NULL THEN 'I'
            WHEN source.LastAccessed > target.LastAccessed THEN 'U'
            ELSE NULL
        END,
        GETUTCDATE()
    FROM 
    OPENQUERY(@SourceServer, 
        'SELECT CacheKey, LastAccessed 
         FROM dbo.DistributedCache') source
    LEFT JOIN
    OPENQUERY(@TargetServer,
        'SELECT CacheKey, LastAccessed 
         FROM dbo.DistributedCache') target
    ON source.CacheKey = target.CacheKey
    WHERE 
        target.CacheKey IS NULL OR
        source.LastAccessed > target.LastAccessed;
    
    -- Synchronize caches
    DECLARE @CacheKey nvarchar(900);
    DECLARE @Operation char(1);
    
    DECLARE sync_cursor CURSOR FOR
    SELECT CacheKey, Operation
    FROM dbo.CacheCoordination
    WHERE Status = 0
    AND ServerName = @TargetServer;
    
    OPEN sync_cursor;
    FETCH NEXT FROM sync_cursor 
    INTO @CacheKey, @Operation;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        IF @Operation IN ('I', 'U')
        BEGIN
            -- Copy cache entry
            DECLARE @SQL nvarchar(max) = '
            INSERT INTO OPENQUERY(' + 
            QUOTENAME(@TargetServer) + ',
            ''SELECT CacheKey, CacheValue, 
                     ExpirationTime, LastAccessed, 
                     AccessCount 
             FROM dbo.DistributedCache'')
            SELECT CacheKey, CacheValue,
                   ExpirationTime, LastAccessed,
                   AccessCount
            FROM dbo.DistributedCache
            WHERE CacheKey = @Key';
            
            EXEC sp_executesql @SQL,
                 N'@Key nvarchar(900)',
                 @CacheKey;
        END;
        
        -- Update coordination status
        UPDATE dbo.CacheCoordination
        SET Status = 1
        WHERE CacheKey = @CacheKey
        AND ServerName = @TargetServer;
        
        FETCH NEXT FROM sync_cursor 
        INTO @CacheKey, @Operation;
    END;
    
    CLOSE sync_cursor;
    DEALLOCATE sync_cursor;
END;
```

## Cross-Environment Data Synchronization

### Change Tracking Framework

1. Change Detection Implementation
```sql
-- Enable change tracking
ALTER DATABASE [LargeDB]
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 14 DAYS, AUTO_CLEANUP = ON);

-- Configure change tracking for critical tables
ALTER TABLE dbo.Customers
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = ON);

-- Create change monitoring
CREATE TABLE dbo.ChangeTrackingStatus
(
    TrackingId bigint IDENTITY(1,1) PRIMARY KEY,
    TableName sysname,
    LastSyncVersion bigint,
    LastSyncTime datetime2,
    ChangeCount int,
    SyncDuration int,
    Status tinyint,
    ErrorMessage nvarchar(max)
);

-- Synchronization procedure
CREATE PROCEDURE dbo.SyncEnvironmentChanges
    @TableName sysname,
    @BatchSize int = 1000
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentVersion bigint;
    DECLARE @LastVersion bigint;
    
    -- Get current version
    SELECT @CurrentVersion = CHANGE_TRACKING_CURRENT_VERSION();
    
    -- Get last synced version
    SELECT @LastVersion = LastSyncVersion
    FROM dbo.ChangeTrackingStatus
    WHERE TableName = @TableName;
    
    -- Get changes
    SELECT 
        c.SYS_CHANGE_VERSION,
        c.SYS_CHANGE_OPERATION,
        t.*
    FROM CHANGETABLE(CHANGES dbo[@TableName], @LastVersion) AS c
    JOIN dbo[@TableName] t
    ON c.Id = t.Id
    ORDER BY c.SYS_CHANGE_VERSION;
    
    -- Update tracking
    UPDATE dbo.ChangeTrackingStatus
    SET LastSyncVersion = @CurrentVersion,
        LastSyncTime = GETUTCDATE(),
        ChangeCount = @@ROWCOUNT
    WHERE TableName = @TableName;
END;
```

2. Consistency Validation
```sql
CREATE TABLE dbo.DataConsistencyCheck
(
    CheckId bigint IDENTITY(1,1) PRIMARY KEY,
    TableName sysname,
    PrimaryKeyValue sql_variant,
    SourceHash varbinary(32),
    TargetHash varbinary(32),
    IsConsistent bit,
    CheckTime datetime2,
    RepairRequired bit
);

-- Validate data consistency
CREATE PROCEDURE dbo.ValidateDataConsistency
    @TableName sysname,
    @SampleSize int = 1000
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    SET @SQL = '
    WITH SampleData AS (
        SELECT TOP(@SampleSize)
            pk.PrimaryKeyValue,
            HASHBYTES(''SHA2_256'', 
                      (SELECT s.* 
                       FROM [SourceDB].' + @TableName + ' s 
                       WHERE s.Id = pk.PrimaryKeyValue 
                       FOR XML RAW)) as SourceHash,
            HASHBYTES(''SHA2_256'', 
                      (SELECT t.* 
                       FROM [TargetDB].' + @TableName + ' t 
                       WHERE t.Id = pk.PrimaryKeyValue 
                       FOR XML RAW)) as TargetHash
        FROM (
            SELECT DISTINCT Id as PrimaryKeyValue
            FROM [SourceDB].' + @TableName + '
        ) pk
        ORDER BY NEWID()
    )
    INSERT INTO dbo.DataConsistencyCheck
    (TableName, PrimaryKeyValue, SourceHash, 
     TargetHash, IsConsistent, CheckTime, RepairRequired)
    SELECT 
        @TableName,
        PrimaryKeyValue,
        SourceHash,
        TargetHash,
        CASE WHEN SourceHash = TargetHash THEN 1 ELSE 0 END,
        GETUTCDATE(),
        CASE WHEN SourceHash <> TargetHash THEN 1 ELSE 0 END
    FROM SampleData;';
    
    EXEC sp_executesql @SQL, 
         N'@TableName sysname, @SampleSize int',
         @TableName, @SampleSize;
END;
```

### Memory-Optimized Operations

1. Hybrid Buffer Management
```sql
-- Create memory-optimized staging
CREATE TABLE dbo.StagingOperations
(
    OperationId bigint IDENTITY(1,1)
        PRIMARY KEY NONCLUSTERED,
    TableName sysname,
    OperationType char(1),
    PayloadJSON nvarchar(max),
    Status tinyint,
    CreatedTime datetime2,
    ProcessedTime datetime2,
    INDEX ix_status NONCLUSTERED (Status)
) WITH (
    MEMORY_OPTIMIZED = ON,
    DURABILITY = SCHEMA_AND_DATA
);

-- Process staging operations
CREATE PROCEDURE dbo.ProcessStagingOperations
    @BatchSize int = 1000
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC WITH
(
    TRANSACTION ISOLATION LEVEL = SNAPSHOT,
    LANGUAGE = N'us_english'
)
    DECLARE @CurrentTime datetime2 = SYSDATETIME();
    
    UPDATE TOP(@BatchSize) dbo.StagingOperations
    SET Status = 1,
        ProcessedTime = @CurrentTime
    WHERE Status = 0;
    
    SELECT *
    FROM dbo.StagingOperations
    WHERE Status = 1
    AND ProcessedTime = @CurrentTime;
END;
```

2. In-Memory Caching Strategy
```sql
-- Create memory-optimized cache
CREATE TABLE dbo.GlobalCache
(
    CacheKey nvarchar(900) COLLATE Latin1_General_100_BIN2
        PRIMARY KEY NONCLUSTERED,
    CacheValue varbinary(max),
    ExpirationTime datetime2,
    UpdateCount int,
    LastUpdated datetime2,
    INDEX ix_expiration NONCLUSTERED (ExpirationTime)
) WITH (
    MEMORY_OPTIMIZED = ON,
    DURABILITY = SCHEMA_AND_DATA
);

-- Cache management procedure
CREATE PROCEDURE dbo.ManageGlobalCache
    @Operation char(1),
    @Key nvarchar(900),
    @Value varbinary(max) = NULL,
    @ExpirationMinutes int = 60
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC WITH
(
    TRANSACTION ISOLATION LEVEL = SNAPSHOT,
    LANGUAGE = N'us_english'
)
    IF @Operation = 'I'  -- Insert
    BEGIN
        MERGE dbo.GlobalCache AS target
        USING (SELECT @Key as CacheKey) AS source
        ON (target.CacheKey = source.CacheKey)
        WHEN MATCHED THEN
            UPDATE SET 
                CacheValue = @Value,
                ExpirationTime = DATEADD(MINUTE, @ExpirationMinutes, SYSDATETIME()),
                UpdateCount = UpdateCount + 1,
                LastUpdated = SYSDATETIME()
        WHEN NOT MATCHED THEN
            INSERT (CacheKey, CacheValue, ExpirationTime, UpdateCount, LastUpdated)
            VALUES (@Key, @Value, 
                    DATEADD(MINUTE, @ExpirationMinutes, SYSDATETIME()),
                    1, SYSDATETIME());
    END
    ELSE IF @Operation = 'D'  -- Delete
    BEGIN
        DELETE FROM dbo.GlobalCache
        WHERE CacheKey = @Key;
    END;
END;
```

This framework provides robust caching and performance management during the migration process, ensuring data consistency and optimal query performance across servers.
