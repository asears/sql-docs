# SQL Server Migration Health Check System

## Parallel Environment Health Monitoring

### Cross-Server Health Metrics
```sql
-- Create health check tables
CREATE TABLE dbo.EnvironmentHealth
(
    HealthId bigint IDENTITY(1,1) PRIMARY KEY,
    ServerName nvarchar(128),
    DatabaseName sysname,
    CompatibilityLevel int,
    MetricCategory nvarchar(50),
    MetricName nvarchar(100),
    MetricValue decimal(18,2),
    ThresholdValue decimal(18,2),
    Status tinyint, -- 0=Healthy, 1=Warning, 2=Critical
    CollectionTime datetime2,
    Details nvarchar(max)
);

CREATE TABLE dbo.HealthCheckConfig
(
    ConfigId int IDENTITY(1,1) PRIMARY KEY,
    MetricCategory nvarchar(50),
    MetricName nvarchar(100),
    WarningThreshold decimal(18,2),
    CriticalThreshold decimal(18,2),
    CollectionInterval int, -- seconds
    Enabled bit,
    LastCollection datetime2
);

-- Initialize health check configuration
INSERT INTO dbo.HealthCheckConfig
(MetricCategory, MetricName, WarningThreshold, CriticalThreshold, CollectionInterval, Enabled)
VALUES
('Performance', 'CPU_Usage', 80.0, 90.0, 60, 1),
('Performance', 'Memory_Usage', 85.0, 95.0, 60, 1),
('Performance', 'IOPS', 5000.0, 8000.0, 300, 1),
('Replication', 'Latency_Seconds', 300.0, 600.0, 60, 1),
('Availability', 'Page_Life_Expectancy', 300.0, 100.0, 60, 1),
('Blocking', 'Blocked_Sessions', 5.0, 10.0, 30, 1),
('TempDB', 'Usage_Percent', 80.0, 90.0, 300, 1),
('Log', 'Usage_Percent', 75.0, 85.0, 60, 1);
```

### Health Collection Procedures

1. Performance Health Check
```sql
CREATE PROCEDURE dbo.CollectPerformanceMetrics
    @ServerName nvarchar(128),
    @DatabaseName sysname
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO dbo.EnvironmentHealth
    (ServerName, DatabaseName, CompatibilityLevel,
     MetricCategory, MetricName, MetricValue,
     ThresholdValue, Status, CollectionTime, Details)
    
    -- CPU Usage
    SELECT 
        @ServerName,
        @DatabaseName,
        d.compatibility_level,
        'Performance',
        'CPU_Usage',
        cpu.SystemIdle,
        c.WarningThreshold,
        CASE 
            WHEN cpu.SystemIdle <= c.CriticalThreshold THEN 2
            WHEN cpu.SystemIdle <= c.WarningThreshold THEN 1
            ELSE 0
        END,
        GETUTCDATE(),
        'CPU pressure detected'
    FROM sys.databases d
    CROSS APPLY (
        SELECT 100 - AVG(SystemIdle) as SystemIdle
        FROM (
            SELECT record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') as SystemIdle
            FROM (
                SELECT TOP 5 CONVERT(xml, record) as record
                FROM sys.dm_os_ring_buffers
                WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
                ORDER BY timestamp DESC
            ) as rb
        ) as cpu_samples
    ) as cpu
    CROSS JOIN dbo.HealthCheckConfig c
    WHERE d.name = @DatabaseName
    AND c.MetricName = 'CPU_Usage';
    
    -- Memory Usage
    INSERT INTO dbo.EnvironmentHealth
    (ServerName, DatabaseName, CompatibilityLevel,
     MetricCategory, MetricName, MetricValue,
     ThresholdValue, Status, CollectionTime, Details)
    SELECT 
        @ServerName,
        @DatabaseName,
        d.compatibility_level,
        'Performance',
        'Memory_Usage',
        physical_memory_in_use_kb * 100.0 / total_physical_memory_kb,
        c.WarningThreshold,
        CASE 
            WHEN (physical_memory_in_use_kb * 100.0 / total_physical_memory_kb) >= c.CriticalThreshold THEN 2
            WHEN (physical_memory_in_use_kb * 100.0 / total_physical_memory_kb) >= c.WarningThreshold THEN 1
            ELSE 0
        END,
        GETUTCDATE(),
        'Memory pressure detected'
    FROM sys.dm_os_process_memory
    CROSS JOIN sys.databases d
    CROSS JOIN dbo.HealthCheckConfig c
    WHERE d.name = @DatabaseName
    AND c.MetricName = 'Memory_Usage';
END;
```

2. Replication Health Check
```sql
CREATE PROCEDURE dbo.CollectReplicationMetrics
    @ServerName nvarchar(128),
    @DatabaseName sysname
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO dbo.EnvironmentHealth
    (ServerName, DatabaseName, CompatibilityLevel,
     MetricCategory, MetricName, MetricValue,
     ThresholdValue, Status, CollectionTime, Details)
    SELECT 
        @ServerName,
        @DatabaseName,
        d.compatibility_level,
        'Replication',
        'Latency_Seconds',
        DATEDIFF(SECOND, last_distsync, GETDATE()),
        c.WarningThreshold,
        CASE 
            WHEN DATEDIFF(SECOND, last_distsync, GETDATE()) >= c.CriticalThreshold THEN 2
            WHEN DATEDIFF(SECOND, last_distsync, GETDATE()) >= c.WarningThreshold THEN 1
            ELSE 0
        END,
        GETUTCDATE(),
        'Replication latency detected'
    FROM MSdistribution_status ds
    CROSS JOIN sys.databases d
    CROSS JOIN dbo.HealthCheckConfig c
    WHERE d.name = @DatabaseName
    AND c.MetricName = 'Latency_Seconds';
END;
```

### Automated Response System

1. Health Issue Resolution
```sql
CREATE PROCEDURE dbo.AutomateHealthResolution
    @ServerName nvarchar(128),
    @DatabaseName sysname,
    @MetricCategory nvarchar(50),
    @MetricName nvarchar(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get latest health status
    DECLARE @Status tinyint;
    DECLARE @MetricValue decimal(18,2);
    
    SELECT TOP 1 
        @Status = Status,
        @MetricValue = MetricValue
    FROM dbo.EnvironmentHealth
    WHERE ServerName = @ServerName
    AND DatabaseName = @DatabaseName
    AND MetricCategory = @MetricCategory
    AND MetricName = @MetricName
    ORDER BY CollectionTime DESC;
    
    -- Apply automated fixes based on issue
    IF @MetricCategory = 'Performance' AND @MetricName = 'CPU_Usage'
    AND @Status >= 1
    BEGIN
        -- Implement resource governor limits
        EXEC sp_execute_external_script
            @language = N'Python',
            @script = N'
import subprocess
subprocess.run(["powershell", "-Command", 
    "Set-ProcessPriority -ProcessName sqlservr -Priority BelowNormal"])
'
    END
    
    IF @MetricCategory = 'Memory_Usage' AND @Status = 2
    BEGIN
        -- Clear plan cache for unused plans
        DBCC FREEPROCCACHE WITH NO_INFOMSGS;
        
        -- Reduce max memory if critical
        EXEC sp_configure 'max server memory (MB)', 
            (SELECT CAST(value_in_use * 0.9 as int) 
             FROM sys.configurations 
             WHERE name = 'max server memory (MB)');
        RECONFIGURE WITH OVERRIDE;
    END;
END;
```

2. Cross-Environment Validation
```sql
CREATE PROCEDURE dbo.ValidateEnvironments
    @SourceServer nvarchar(128),
    @TargetServer nvarchar(128),
    @DatabaseName sysname
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Compare performance metrics
    WITH SourceMetrics AS (
        SELECT MetricCategory, MetricName, AVG(MetricValue) as SourceValue
        FROM dbo.EnvironmentHealth
        WHERE ServerName = @SourceServer
        AND DatabaseName = @DatabaseName
        AND CollectionTime > DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY MetricCategory, MetricName
    ),
    TargetMetrics AS (
        SELECT MetricCategory, MetricName, AVG(MetricValue) as TargetValue
        FROM dbo.EnvironmentHealth
        WHERE ServerName = @TargetServer
        AND DatabaseName = @DatabaseName
        AND CollectionTime > DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY MetricCategory, MetricName
    )
    SELECT 
        s.MetricCategory,
        s.MetricName,
        s.SourceValue,
        t.TargetValue,
        ((t.TargetValue - s.SourceValue) * 100.0) / 
            NULLIF(s.SourceValue, 0) as PercentDifference,
        CASE 
            WHEN ABS(((t.TargetValue - s.SourceValue) * 100.0) / 
                     NULLIF(s.SourceValue, 0)) > 20 THEN 'Warning'
            ELSE 'OK'
        END as Status
    FROM SourceMetrics s
    JOIN TargetMetrics t
    ON s.MetricCategory = t.MetricCategory
    AND s.MetricName = t.MetricName;
END;
```

This health check system provides comprehensive monitoring of both environments during the migration process, with automated responses to common issues and validation of cross-environment performance parity.
