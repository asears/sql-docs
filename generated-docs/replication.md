# SQL Server Replication for Large Database Migration

## Replication Strategies for 5TB+ Databases

### Initial Data Movement Strategy

1. Snapshot Optimization
```sql
-- Configure large tables for chunked snapshot processing
sp_addpublication 'LargeDBMigration'
    @description = N'Large Database Migration Publication',
    @snapshot_in_defaultfolder = N'Y',
    @allow_push = N'Y',
    @repl_freq = N'continuous',
    @retention = 0,
    @allow_initialize_from_backup = N'Y'

-- Configure chunked snapshot processing
sp_addarticle 'LargeDBMigration',
    'LargeTable',
    @source_owner = 'dbo',
    @source_object = 'LargeTable',
    @pre_creation_cmd = 'drop',
    @schema_option = 0x01,
    @identityrangemanagementoption = 'manual',
    @type = 'logbased',
    @split_snapshot = N'Y',
    @chunk_size = 50 -- Size in MB
```

2. Filegroup Migration
```sql
-- Create publication with filegroup articles
sp_addarticle 'LargeDBMigration',
    'FileGroupArticle',
    @source_owner = 'dbo',
    @source_object = 'LargeTable',
    @destination_table = 'LargeTable',
    @included_filegroups = 'PRIMARY;FG_Archive_2020;FG_Archive_2021'
```

### BLOB Data Handling

1. FILESTREAM Configuration
```sql
-- Configure FILESTREAM replication
sp_configure 'filestream access level', 2
RECONFIGURE

-- Add FILESTREAM article
sp_addarticle 'LargeDBMigration',
    'DocumentStore',
    @source_owner = 'dbo',
    @source_object = 'DocumentStore',
    @type = 'logbased',
    @filestream_on = 'FileStreamGroup'
```

2. Chunked BLOB Transfer
```sql
-- Create custom chunked transfer procedure
CREATE PROCEDURE dbo.ReplicateLargeBLOBs
    @BatchSize int = 1000,
    @MaxChunkSize int = 8388608 -- 8MB chunks
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Offset int = 0;
    DECLARE @BlobId int;
    DECLARE @TotalSize bigint;
    
    DECLARE blob_cursor CURSOR FOR
    SELECT BlobId, DATALENGTH(BlobData)
    FROM dbo.LargeBLOBTable
    WHERE IsReplicated = 0;
    
    OPEN blob_cursor;
    FETCH NEXT FROM blob_cursor INTO @BlobId, @TotalSize;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        WHILE @Offset < @TotalSize
        BEGIN
            -- Replicate chunk
            INSERT INTO [Subscriber].BlobChunks
            SELECT 
                @BlobId,
                @Offset,
                SUBSTRING(BlobData, @Offset + 1, @MaxChunkSize)
            FROM dbo.LargeBLOBTable
            WHERE BlobId = @BlobId;
            
            SET @Offset = @Offset + @MaxChunkSize;
        END;
        
        FETCH NEXT FROM blob_cursor INTO @BlobId, @TotalSize;
    END;
    
    CLOSE blob_cursor;
    DEALLOCATE blob_cursor;
END;
```

## Parallel Operation Support

### Multi-Subscriber Configuration

1. Distribution Database Setup
```sql
-- Configure distribution database
sp_adddistributiondb 'distribution'
    @security_mode = 1,
    @history_retention = 48,
    @max_distretention = 72,
    @job_login = null,
    @job_password = null,
    @min_distretention = 0,
    @max_distthread = 4,
    @working_directory = N'E:\SQLDistribution'

-- Add multiple subscribers
sp_addsubscription
    @publication = 'LargeDBMigration',
    @subscriber = 'NEWSQLSERVER',
    @destination_db = 'LargeDB_Copy',
    @subscription_type = 'Push',
    @sync_type = 'automatic',
    @article = 'all',
    @update_mode = 'read only'
```

2. Load Balancing
```sql
-- Create subscriber routing
CREATE TABLE dbo.SubscriberRouting
(
    RoutingId int IDENTITY(1,1),
    SubscriberName nvarchar(128),
    Priority int,
    LoadFactor decimal(5,2),
    IsActive bit,
    LastUpdated datetime2
)

-- Monitor subscriber health
CREATE PROCEDURE dbo.UpdateSubscriberHealth
AS
BEGIN
    UPDATE dbo.SubscriberRouting
    SET LoadFactor = sub.load_factor,
        LastUpdated = GETUTCDATE()
    FROM dbo.SubscriberRouting sr
    CROSS APPLY (
        SELECT 
            AVG(cpu_percent) as load_factor
        FROM sys.dm_os_ring_buffers
        WHERE ring_buffer_type = 'RING_BUFFER_SCHEDULER_MONITOR'
    ) sub;
END;
```

### Data Consistency Validation

1. Checksum Validation
```sql
-- Create checksum tracking
CREATE TABLE dbo.ReplicationValidation
(
    ValidationId bigint IDENTITY(1,1),
    ArticleName sysname,
    PublisherChecksum binary(32),
    SubscriberChecksum binary(32),
    ValidationTime datetime2,
    IsValid bit
)

-- Validate consistency
CREATE PROCEDURE dbo.ValidateReplication
    @ArticleName sysname
AS
BEGIN
    DECLARE @PubChecksum binary(32);
    DECLARE @SubChecksum binary(32);
    
    -- Get publisher checksum
    SELECT @PubChecksum = HASHBYTES('SHA2_256', 
        (SELECT * FROM dbo[@ArticleName] FOR XML AUTO));
        
    -- Get subscriber checksum
    SELECT @SubChecksum = HASHBYTES('SHA2_256',
        (SELECT * FROM [Subscriber].[dbo].[@ArticleName] FOR XML AUTO));
        
    INSERT INTO dbo.ReplicationValidation
    VALUES (@ArticleName, @PubChecksum, @SubChecksum, 
            GETUTCDATE(), 
            CASE WHEN @PubChecksum = @SubChecksum THEN 1 ELSE 0 END);
END;
```

2. Conflict Detection
```sql
-- Create conflict tracking
CREATE TABLE dbo.ReplicationConflicts
(
    ConflictId bigint IDENTITY(1,1),
    ArticleName sysname,
    ConflictType nvarchar(50),
    PublisherValue sql_variant,
    SubscriberValue sql_variant,
    DetectionTime datetime2,
    ResolvedBy nvarchar(128),
    Resolution nvarchar(max)
)

-- Monitor for conflicts
CREATE PROCEDURE dbo.DetectReplicationConflicts
    @ArticleName sysname
AS
BEGIN
    INSERT INTO dbo.ReplicationConflicts
    SELECT 
        @ArticleName,
        'Data Mismatch',
        pub.Value,
        sub.Value,
        GETUTCDATE(),
        NULL,
        NULL
    FROM dbo[@ArticleName] pub
    FULL OUTER JOIN [Subscriber].[dbo].[@ArticleName] sub
    ON pub.PrimaryKey = sub.PrimaryKey
    WHERE pub.Value <> sub.Value
    OR (pub.Value IS NULL AND sub.Value IS NOT NULL)
    OR (pub.Value IS NOT NULL AND sub.Value IS NULL);
END;
```

## Performance Optimization

### Replication Monitor

1. Performance Metrics
```sql
-- Create monitoring tables
CREATE TABLE dbo.ReplicationPerformance
(
    MetricId bigint IDENTITY(1,1),
    ArticleName sysname,
    CommandCount int,
    DeliveryLatency int,
    DeliveryRate decimal(10,2),
    CaptureTime datetime2
)

-- Monitor delivery rates
CREATE PROCEDURE dbo.CaptureReplicationMetrics
AS
BEGIN
    INSERT INTO dbo.ReplicationPerformance
    SELECT 
        art.name,
        perf.delivery_count,
        perf.delivery_latency,
        perf.delivery_rate,
        GETUTCDATE()
    FROM MSreplication_monitordata perf
    JOIN sys.articles art
    ON perf.article_id = art.article_id;
END;
```

2. Resource Management
```sql
-- Configure replication agent profiles
sp_help_agent_profile
    @agent_type = 'distribution'

-- Update agent profile
sp_update_agent_profile
    @agent_type = 'distribution',
    @profile_id = 1,
    @property_name = 'QueryTimeout',
    @property_value = '7200'  -- 2 hours
```

This comprehensive replication framework provides the foundation for maintaining data consistency and parallel operations during the migration of large databases.
