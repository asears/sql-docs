# SQL Server Backup Analysis Framework

## Backup Performance Monitoring

### Backup History Analysis
```sql
CREATE TABLE dbo.BackupPerformanceMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseName sysname,
    BackupType char(1),
    BackupStartTime datetime2,
    BackupFinishTime datetime2,
    BackupSizeMB decimal(18,2),
    CompressedSizeMB decimal(18,2),
    CompressionRatio decimal(5,2),
    ThroughputMBSec decimal(18,2),
    BackupDevice nvarchar(260),
    IsEncrypted bit,
    IsCompressed bit,
    IsChecksumEnabled bit,
    PhysicalBlockSize int,
    IoLatencyMs decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.TrackBackupPerformance
    @SlowBackupThresholdMins int = 60,
    @LowThroughputMBSec decimal(18,2) = 100.0
AS
BEGIN
    -- Capture backup performance metrics
    INSERT INTO dbo.BackupPerformanceMetrics
    SELECT 
        bs.database_name,
        bs.type,
        bs.backup_start_date,
        bs.backup_finish_date,
        bs.backup_size / 1048576.0,
        bs.compressed_backup_size / 1048576.0,
        CASE 
            WHEN bs.backup_size = 0 THEN 0
            ELSE (1 - bs.compressed_backup_size * 1.0 / 
                     bs.backup_size) * 100
        END,
        CASE 
            WHEN DATEDIFF(
                SECOND, 
                bs.backup_start_date, 
                bs.backup_finish_date
            ) = 0 THEN 0
            ELSE (bs.backup_size / 1048576.0) / 
                 DATEDIFF(
                     SECOND, 
                     bs.backup_start_date, 
                     bs.backup_finish_date
                 )
        END,
        bmf.physical_device_name,
        bs.is_encrypted,
        bs.is_compressed,
        bs.is_checksum_enabled,
        bs.block_size,
        DATEDIFF(
            MILLISECOND,
            bs.backup_start_date,
            bs.backup_finish_date
        ) * 1.0 / NULLIF(bs.backup_size / 1048576.0, 0),
        GETUTCDATE()
    FROM msdb.dbo.backupset bs
    JOIN msdb.dbo.backupmediafamily bmf 
        ON bs.media_set_id = bmf.media_set_id
    WHERE bs.backup_finish_date >= DATEADD(DAY, -7, GETUTCDATE());

    -- Analyze backup performance patterns
    WITH BackupMetrics AS (
        SELECT 
            DatabaseName,
            BackupType,
            BackupStartTime,
            BackupFinishTime,
            BackupSizeMB,
            CompressedSizeMB,
            CompressionRatio,
            ThroughputMBSec,
            IoLatencyMs,
            DATEDIFF(
                MINUTE, 
                BackupStartTime, 
                BackupFinishTime
            ) as DurationMinutes,
            LAG(ThroughputMBSec) OVER (
                PARTITION BY DatabaseName, BackupType 
                ORDER BY BackupStartTime
            ) as PreviousThroughput
        FROM dbo.BackupPerformanceMetrics
        WHERE CollectionTime >= DATEADD(DAY, -7, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        CASE BackupType
            WHEN 'D' THEN 'Full'
            WHEN 'I' THEN 'Differential'
            WHEN 'L' THEN 'Log'
            ELSE 'Unknown'
        END as BackupType,
        BackupStartTime,
        DurationMinutes,
        BackupSizeMB,
        CompressedSizeMB,
        CompressionRatio,
        ThroughputMBSec,
        IoLatencyMs,
        CASE 
            WHEN DurationMinutes > @SlowBackupThresholdMins 
            THEN 'Long Duration'
            WHEN ThroughputMBSec < @LowThroughputMBSec 
            THEN 'Low Throughput'
            WHEN ThroughputMBSec < PreviousThroughput * 0.5 
            THEN 'Performance Degradation'
            ELSE 'Normal'
        END as BackupStatus,
        CASE 
            WHEN DurationMinutes > @SlowBackupThresholdMins 
            THEN 'Consider:
                  1. Stripe across multiple files
                  2. Review backup device performance
                  3. Check for blocking operations'
            WHEN ThroughputMBSec < @LowThroughputMBSec 
            THEN 'Review:
                  1. Backup device throughput
                  2. Network bandwidth
                  3. Compression settings'
            WHEN ThroughputMBSec < PreviousThroughput * 0.5 
            THEN 'Investigate recent changes in:
                  1. Backup configuration
                  2. Storage performance
                  3. Database size'
            ELSE 'No action needed'
        END as Recommendation
    FROM BackupMetrics
    WHERE DurationMinutes > @SlowBackupThresholdMins
    OR ThroughputMBSec < @LowThroughputMBSec
    OR ThroughputMBSec < PreviousThroughput * 0.5
    ORDER BY 
        CASE 
            WHEN DurationMinutes > @SlowBackupThresholdMins THEN 1
            WHEN ThroughputMBSec < @LowThroughputMBSec THEN 2
            ELSE 3
        END,
        BackupStartTime DESC;
END;
```

### Restore Performance Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeRestorePerformance
AS
BEGIN
    -- Analyze restore operations
    SELECT 
        rs.destination_database_name,
        rs.restore_date,
        rs.backup_set_id,
        bs.backup_start_date,
        bs.backup_finish_date,
        bs.backup_size / 1048576.0 as BackupSizeMB,
        bs.compressed_backup_size / 1048576.0 as CompressedSizeMB,
        DATEDIFF(
            MINUTE, 
            rs.restore_date, 
            rs.last_restore_date
        ) as RestoreDurationMinutes,
        CASE 
            WHEN DATEDIFF(
                SECOND, 
                rs.restore_date, 
                rs.last_restore_date
            ) = 0 THEN 0
            ELSE (bs.backup_size / 1048576.0) / 
                 DATEDIFF(
                     SECOND, 
                     rs.restore_date, 
                     rs.last_restore_date
                 )
        END as RestoreThroughputMBSec,
        CASE 
            WHEN DATEDIFF(
                MINUTE, 
                rs.restore_date, 
                rs.last_restore_date
            ) > 60 
            THEN 'Long Duration'
            WHEN (bs.backup_size / 1048576.0) / 
                 NULLIF(
                     DATEDIFF(
                         SECOND, 
                         rs.restore_date, 
                         rs.last_restore_date
                     ),
                     0
                 ) < 50 
            THEN 'Low Throughput'
            ELSE 'Normal'
        END as RestoreStatus,
        CASE 
            WHEN DATEDIFF(
                MINUTE, 
                rs.restore_date, 
                rs.last_restore_date
            ) > 60 
            THEN 'Review restore configuration and storage performance'
            WHEN (bs.backup_size / 1048576.0) / 
                 NULLIF(
                     DATEDIFF(
                         SECOND, 
                         rs.restore_date, 
                         rs.last_restore_date
                     ),
                     0
                 ) < 50 
            THEN 'Investigate storage and memory resources'
            ELSE 'No action needed'
        END as Recommendation
    FROM msdb.dbo.restorehistory rs
    JOIN msdb.dbo.backupset bs 
        ON rs.backup_set_id = bs.backup_set_id
    WHERE rs.restore_date >= DATEADD(DAY, -7, GETUTCDATE())
    AND (
        DATEDIFF(
            MINUTE, 
            rs.restore_date, 
            rs.last_restore_date
        ) > 60
        OR (bs.backup_size / 1048576.0) / 
           NULLIF(
               DATEDIFF(
                   SECOND, 
                   rs.restore_date, 
                   rs.last_restore_date
               ),
               0
           ) < 50
    )
    ORDER BY rs.restore_date DESC;
END;
```

### Recovery Time Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeRecoveryTime
AS
BEGIN
    -- Estimate recovery times
    WITH RecoveryMetrics AS (
        SELECT 
            d.name as DatabaseName,
            d.recovery_model_desc as RecoveryModel,
            (
                SELECT SUM(size) * 8.0 / 1024 
                FROM sys.database_files 
                WHERE database_id = d.database_id
                AND type = 0
            ) as DataFileSizeMB,
            (
                SELECT SUM(size) * 8.0 / 1024 
                FROM sys.database_files 
                WHERE database_id = d.database_id
                AND type = 1
            ) as LogFileSizeMB,
            (
                SELECT backup_size / 1048576.0
                FROM msdb.dbo.backupset
                WHERE database_name = d.name
                AND type = 'D'  -- Full backup
                AND backup_finish_date = (
                    SELECT MAX(backup_finish_date)
                    FROM msdb.dbo.backupset
                    WHERE database_name = d.name
                    AND type = 'D'
                )
            ) as LastFullBackupSizeMB,
            (
                SELECT compressed_backup_size / 1048576.0
                FROM msdb.dbo.backupset
                WHERE database_name = d.name
                AND type = 'L'  -- Log backup
                AND backup_finish_date = (
                    SELECT MAX(backup_finish_date)
                    FROM msdb.dbo.backupset
                    WHERE database_name = d.name
                    AND type = 'L'
                )
            ) as LastLogBackupSizeMB,
            (
                SELECT AVG(
                    DATEDIFF(
                        SECOND,
                        backup_start_date,
                        backup_finish_date
                    )
                )
                FROM msdb.dbo.backupset
                WHERE database_name = d.name
                AND type = 'D'
                AND backup_finish_date >= DATEADD(DAY, -7, GETUTCDATE())
            ) as AvgFullBackupDurationSec,
            (
                SELECT AVG(
                    DATEDIFF(
                        SECOND,
                        restore_date,
                        last_restore_date
                    )
                )
                FROM msdb.dbo.restorehistory r
                JOIN msdb.dbo.backupset b 
                    ON r.backup_set_id = b.backup_set_id
                WHERE b.database_name = d.name
                AND r.restore_date >= DATEADD(DAY, -7, GETUTCDATE())
            ) as AvgRestoreDurationSec
        FROM sys.databases d
        WHERE d.database_id > 4  -- Exclude system databases
    )
    SELECT 
        DatabaseName,
        RecoveryModel,
        DataFileSizeMB,
        LogFileSizeMB,
        LastFullBackupSizeMB,
        LastLogBackupSizeMB,
        AvgFullBackupDurationSec,
        AvgRestoreDurationSec,
        CASE 
            WHEN AvgRestoreDurationSec > 3600  -- 1 hour
            THEN 'Long Recovery Time'
            WHEN AvgRestoreDurationSec > 1800  -- 30 minutes
            THEN 'Moderate Recovery Time'
            ELSE 'Acceptable Recovery Time'
        END as RecoveryStatus,
        CASE 
            WHEN AvgRestoreDurationSec > 3600 
            THEN 'Consider:
                  1. Implementing log shipping
                  2. Using multiple backup files
                  3. Reviewing hardware capabilities'
            WHEN AvgRestoreDurationSec > 1800 
            THEN 'Review recovery process and resources'
            ELSE 'No action needed'
        END as Recommendation
    FROM RecoveryMetrics
    WHERE AvgRestoreDurationSec > 1800
    ORDER BY AvgRestoreDurationSec DESC;
END;
```

This backup analysis framework provides comprehensive tools for:
1. Monitoring backup performance and identifying bottlenecks
2. Analyzing restore operations and throughput
3. Estimating and optimizing recovery times
4. Tracking compression and encryption impacts

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
