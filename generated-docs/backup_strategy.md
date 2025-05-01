# Large Database Backup Strategy During Migration

## Backup Architecture for 5TB+ Databases

### Striped Backup Implementation

1. Multi-File Backup Configuration
```sql
-- Configure backup striping
CREATE TABLE dbo.BackupConfiguration
(
    ConfigId int IDENTITY(1,1) PRIMARY KEY,
    BackupType char(1),  -- F=Full, D=Diff, L=Log
    StripeCount int,
    CompressionEnabled bit,
    MaxTransferSize int,
    BufferCount int,
    BlockSize int,
    IsDefault bit
);

-- Default configurations
INSERT INTO dbo.BackupConfiguration
VALUES
('F', 8, 1, 4194304, 64, 65536, 1),
('D', 4, 1, 4194304, 32, 65536, 1),
('L', 2, 1, 1048576, 16, 65536, 1);

-- Create striped backup procedure
CREATE PROCEDURE dbo.ExecuteStripedBackup
    @DatabaseName sysname,
    @BackupPath nvarchar(1000),
    @BackupType char(1)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL nvarchar(max);
    DECLARE @StripeCount int;
    DECLARE @Compression bit;
    DECLARE @MaxTransfer int;
    DECLARE @BufferCount int;
    DECLARE @BlockSize int;
    
    -- Get configuration
    SELECT 
        @StripeCount = StripeCount,
        @Compression = CompressionEnabled,
        @MaxTransfer = MaxTransferSize,
        @BufferCount = BufferCount,
        @BlockSize = BlockSize
    FROM dbo.BackupConfiguration
    WHERE BackupType = @BackupType
    AND IsDefault = 1;
    
    -- Build backup files string
    DECLARE @Files nvarchar(max) = '';
    DECLARE @i int = 1;
    
    WHILE @i <= @StripeCount
    BEGIN
        SET @Files = @Files + 
            CASE WHEN @i > 1 THEN ',' ELSE '' END +
            'DISK = ''' + @BackupPath + '\' + 
            @DatabaseName + '_' + 
            CAST(@i as varchar(2)) + '.bak''';
        SET @i += 1;
    END;
    
    -- Build backup command
    SET @SQL = 'BACKUP ' + 
        CASE @BackupType 
            WHEN 'F' THEN 'DATABASE'
            WHEN 'D' THEN 'DATABASE'
            WHEN 'L' THEN 'LOG'
        END + ' ' +
        QUOTENAME(@DatabaseName) + '
        TO ' + @Files + '
        WITH INIT, 
        COMPRESSION = ' + CASE @Compression WHEN 1 THEN 'ON' ELSE 'OFF' END + ',
        MAXTRANSFERSIZE = ' + CAST(@MaxTransfer as varchar(10)) + ',
        BUFFERCOUNT = ' + CAST(@BufferCount as varchar(10)) + ',
        BLOCKSIZE = ' + CAST(@BlockSize as varchar(10)) + 
        CASE @BackupType WHEN 'D' THEN ', DIFFERENTIAL' ELSE '' END + ',
        STATS = 10';
    
    -- Execute backup
    EXEC sp_executesql @SQL;
END;
```

### Parallel Backup Processing

1. Filegroup Backup Strategy
```sql
-- Configure filegroup backups
CREATE TABLE dbo.FileGroupBackupConfig
(
    FileGroupId int IDENTITY(1,1) PRIMARY KEY,
    FileGroupName sysname,
    BackupPriority int,
    IsReadOnly bit,
    LastBackupTime datetime2,
    BackupDuration int,
    SizeGB decimal(10,2)
);

-- Create filegroup backup procedure
CREATE PROCEDURE dbo.BackupFileGroups
    @DatabaseName sysname,
    @BackupPath nvarchar(1000)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @FileGroup sysname;
    DECLARE @Priority int;
    
    -- Get filegroup information
    INSERT INTO dbo.FileGroupBackupConfig
    (FileGroupName, IsReadOnly, BackupPriority)
    SELECT 
        name,
        is_read_only,
        CASE 
            WHEN is_read_only = 1 THEN 1
            ELSE 2
        END
    FROM sys.filegroups
    WHERE type = 'FG'
    AND name NOT IN (
        SELECT FileGroupName 
        FROM dbo.FileGroupBackupConfig
    );
    
    -- Process each filegroup
    DECLARE fg_cursor CURSOR FOR
    SELECT FileGroupName, BackupPriority
    FROM dbo.FileGroupBackupConfig
    ORDER BY BackupPriority, FileGroupName;
    
    OPEN fg_cursor;
    FETCH NEXT FROM fg_cursor INTO @FileGroup, @Priority;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = 'BACKUP DATABASE ' + QUOTENAME(@DatabaseName) + '
        FILEGROUP = ''' + @FileGroup + '''
        TO DISK = ''' + @BackupPath + '\' + 
        @DatabaseName + '_' + @FileGroup + '.bak''
        WITH INIT, COMPRESSION,
        STATS = 10';
        
        EXEC sp_executesql @SQL;
        
        -- Update backup history
        UPDATE dbo.FileGroupBackupConfig
        SET LastBackupTime = GETUTCDATE(),
            BackupDuration = DATEDIFF(SECOND, 
                GETUTCDATE(), 
                DATEADD(SECOND, @@ROWCOUNT, GETUTCDATE()))
        WHERE FileGroupName = @FileGroup;
        
        FETCH NEXT FROM fg_cursor INTO @FileGroup, @Priority;
    END;
    
    CLOSE fg_cursor;
    DEALLOCATE fg_cursor;
END;
```

### Point-in-Time Recovery Support

1. Transaction Log Management
```sql
-- Configure log backup chains
CREATE TABLE dbo.LogBackupChain
(
    ChainId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    StartLSN numeric(25,0),
    EndLSN numeric(25,0),
    FirstBackupTime datetime2,
    LastBackupTime datetime2,
    BackupCount int,
    TotalSizeGB decimal(10,2)
);

-- Track log backups
CREATE PROCEDURE dbo.TrackLogBackupChain
    @DatabaseName sysname
AS
BEGIN
    -- Update existing chain or start new one
    DECLARE @CurrentLSN numeric(25,0);
    DECLARE @LastLSN numeric(25,0);
    
    SELECT @CurrentLSN = current_log_lsn
    FROM sys.dm_database_log_stats(DB_ID(@DatabaseName));
    
    SELECT TOP 1 @LastLSN = EndLSN
    FROM dbo.LogBackupChain
    WHERE DatabaseName = @DatabaseName
    ORDER BY ChainId DESC;
    
    IF @LastLSN IS NULL OR @CurrentLSN < @LastLSN
    BEGIN
        -- Start new chain
        INSERT INTO dbo.LogBackupChain
        (DatabaseName, StartLSN, EndLSN, 
         FirstBackupTime, LastBackupTime,
         BackupCount, TotalSizeGB)
        VALUES
        (@DatabaseName, @CurrentLSN, @CurrentLSN,
         GETUTCDATE(), GETUTCDATE(), 0, 0);
    END
    ELSE
    BEGIN
        -- Update existing chain
        UPDATE dbo.LogBackupChain
        SET EndLSN = @CurrentLSN,
            LastBackupTime = GETUTCDATE(),
            BackupCount = BackupCount + 1,
            TotalSizeGB = TotalSizeGB + 
                (SELECT backup_size / 1073741824.0
                 FROM msdb.dbo.backupset
                 WHERE database_name = @DatabaseName
                 AND backup_start_date = 
                     (SELECT MAX(backup_start_date)
                      FROM msdb.dbo.backupset
                      WHERE database_name = @DatabaseName))
        WHERE ChainId = (
            SELECT TOP 1 ChainId
            FROM dbo.LogBackupChain
            WHERE DatabaseName = @DatabaseName
            ORDER BY ChainId DESC
        );
    END;
END;
```

2. Recovery Point Validation
```sql
CREATE PROCEDURE dbo.ValidateRecoveryPoint
    @DatabaseName sysname,
    @TargetTime datetime2
AS
BEGIN
    -- Check if point-in-time recovery is possible
    WITH BackupHistory AS (
        SELECT 
            backup_start_date,
            backup_finish_date,
            first_lsn,
            last_lsn,
            database_backup_lsn,
            backup_type
        FROM msdb.dbo.backupset
        WHERE database_name = @DatabaseName
        AND backup_start_date <= @TargetTime
    )
    SELECT 
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM BackupHistory
                WHERE backup_type = 'D'  -- Full backup
                AND backup_start_date <= @TargetTime
            ) AND
            EXISTS (
                SELECT 1
                FROM BackupHistory
                WHERE backup_type = 'L'  -- Log backup
                AND backup_start_date > @TargetTime
            )
            THEN 'Recovery to ' + 
                 CONVERT(varchar(30), @TargetTime, 121) + 
                 ' is possible'
            ELSE 'Recovery point not available'
        END as RecoveryStatus,
        MIN(CASE 
            WHEN backup_type = 'D' AND 
                 backup_start_date <= @TargetTime
            THEN backup_start_date
        END) as RequiredFullBackup,
        MIN(CASE 
            WHEN backup_type = 'L' AND 
                 backup_start_date > @TargetTime
            THEN backup_start_date
        END) as RequiredLogBackup
    FROM BackupHistory;
END;
```

## Monitoring and Validation

### Backup Performance Tracking
```sql
CREATE TABLE dbo.BackupPerformance
(
    PerformanceId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    BackupType char(1),
    StartTime datetime2,
    EndTime datetime2,
    DurationSeconds int,
    CompressedSizeGB decimal(10,2),
    UncompressedSizeGB decimal(10,2),
    CompressionRatio decimal(5,2),
    ThroughputMBSec decimal(10,2)
);

-- Monitor backup performance
CREATE PROCEDURE dbo.TrackBackupPerformance
    @DatabaseName sysname
AS
BEGIN
    INSERT INTO dbo.BackupPerformance
    SELECT 
        database_name,
        CASE type
            WHEN 'D' THEN 'F'  -- Full
            WHEN 'I' THEN 'D'  -- Differential
            WHEN 'L' THEN 'L'  -- Log
        END,
        backup_start_date,
        backup_finish_date,
        DATEDIFF(SECOND, 
            backup_start_date, 
            backup_finish_date),
        compressed_backup_size / 1073741824.0,
        backup_size / 1073741824.0,
        CAST(
            (1 - (compressed_backup_size * 1.0 / backup_size)) * 100 
            as decimal(5,2)),
        (backup_size / 1048576.0) / 
            NULLIF(DATEDIFF(SECOND, 
                   backup_start_date, 
                   backup_finish_date), 0)
    FROM msdb.dbo.backupset
    WHERE database_name = @DatabaseName
    AND backup_finish_date > 
        (SELECT ISNULL(MAX(EndTime), '1900-01-01')
         FROM dbo.BackupPerformance);
END;
```

This backup strategy provides a robust framework for managing backups of large databases during migration, ensuring minimal impact on production workloads while maintaining point-in-time recovery capabilities.
