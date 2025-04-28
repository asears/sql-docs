# SQL Server Traffic Routing and Load Management

## Application Request Routing

### Connection String Management
```sql
-- Create application routing configuration
CREATE TABLE dbo.ApplicationRouting
(
    ApplicationId int IDENTITY(1,1) PRIMARY KEY,
    ApplicationName nvarchar(128),
    ConnectionPattern nvarchar(max),
    SourceServer nvarchar(128),
    TargetServer nvarchar(128),
    ReadWriteMode tinyint, -- 1=Read-Only, 2=Write-Only, 3=Read-Write
    RoutingPercentage decimal(5,2),
    IsActive bit DEFAULT 1,
    LastModified datetime2 DEFAULT GETUTCDATE()
);

-- Create routing history for analysis
CREATE TABLE dbo.RoutingHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    ApplicationId int,
    RoutedServer nvarchar(128),
    RequestType char(1), -- 'R'ead or 'W'rite
    ExecutionTime datetime2,
    Duration_ms int,
    WasSuccessful bit,
    ErrorMessage nvarchar(max) NULL
);
```

### Gradual Traffic Migration

1. Progressive Routing Implementation
```sql
CREATE PROCEDURE dbo.UpdateRoutingPercentage
    @ApplicationName nvarchar(128),
    @NewPercentage decimal(5,2),
    @StepSize decimal(5,2) = 10.0,
    @MonitoringMinutes int = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentPct decimal(5,2);
    DECLARE @TargetPct decimal(5,2) = @NewPercentage;
    DECLARE @AppId int;
    
    SELECT @CurrentPct = RoutingPercentage,
           @AppId = ApplicationId
    FROM dbo.ApplicationRouting
    WHERE ApplicationName = @ApplicationName;
    
    WHILE @CurrentPct <> @TargetPct
    BEGIN
        -- Increment or decrement by step size
        SET @CurrentPct = @CurrentPct + 
            CASE 
                WHEN @TargetPct > @CurrentPct 
                THEN @StepSize
                ELSE -@StepSize
            END;
        
        -- Ensure we don't exceed bounds
        SET @CurrentPct = 
            CASE
                WHEN @CurrentPct > 100 THEN 100
                WHEN @CurrentPct < 0 THEN 0
                ELSE @CurrentPct
            END;
        
        -- Update routing
        UPDATE dbo.ApplicationRouting
        SET RoutingPercentage = @CurrentPct,
            LastModified = GETUTCDATE()
        WHERE ApplicationId = @AppId;
        
        -- Monitor for errors
        WAITFOR DELAY '00:01:00';
        
        DECLARE @ErrorCount int;
        SELECT @ErrorCount = COUNT(*)
        FROM dbo.RoutingHistory
        WHERE ApplicationId = @AppId
        AND WasSuccessful = 0
        AND ExecutionTime > DATEADD(MINUTE, -@MonitoringMinutes, GETUTCDATE());
        
        -- Rollback if error threshold exceeded
        IF @ErrorCount > 10
        BEGIN
            UPDATE dbo.ApplicationRouting
            SET RoutingPercentage = @CurrentPct - @StepSize,
                LastModified = GETUTCDATE()
            WHERE ApplicationId = @AppId;
            
            RAISERROR('Error threshold exceeded. Rolling back routing change.', 16, 1);
            RETURN;
        END;
    END;
END;
```

2. Load Balancing Strategy
```sql
CREATE FUNCTION dbo.GetOptimalServer
(
    @ApplicationName nvarchar(128),
    @RequestType char(1)
)
RETURNS TABLE
AS
RETURN
(
    SELECT TOP 1
        CASE
            WHEN RAND() * 100 <= ar.RoutingPercentage 
                 AND sl.CPULoad < 80
                 AND sl.MemoryLoad < 90
            THEN ar.TargetServer
            ELSE ar.SourceServer
        END AS ServerName
    FROM dbo.ApplicationRouting ar
    LEFT JOIN dbo.ServerLoad sl
    ON ar.TargetServer = sl.ServerName
    WHERE ar.ApplicationName = @ApplicationName
    AND ar.IsActive = 1
    AND (
        (@RequestType = 'R' AND ar.ReadWriteMode IN (1,3))
        OR
        (@RequestType = 'W' AND ar.ReadWriteMode IN (2,3))
    )
);
```

## Performance Monitoring

### Real-time Metrics Collection
```sql
CREATE TABLE dbo.PerformanceMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    ServerName nvarchar(128),
    MetricName nvarchar(100),
    MetricValue decimal(18,2),
    CollectionTime datetime2 DEFAULT GETUTCDATE(),
    INDEX IX_Collection CLUSTERED (CollectionTime, ServerName)
);

CREATE PROCEDURE dbo.CollectPerformanceMetrics
    @ServerName nvarchar(128)
AS
BEGIN
    INSERT INTO dbo.PerformanceMetrics
    (ServerName, MetricName, MetricValue)
    SELECT 
        @ServerName,
        counter_name,
        cntr_value
    FROM sys.dm_os_performance_counters
    WHERE counter_name IN (
        'Batch Requests/sec',
        'SQL Compilations/sec',
        'SQL Re-Compilations/sec',
        'User Connections',
        'Buffer cache hit ratio',
        'Page life expectancy'
    );
END;
```

### Automated Response System
```sql
CREATE PROCEDURE dbo.AdjustRoutingBasedOnLoad
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for overloaded servers
    DECLARE @OverloadedServers TABLE (ServerName nvarchar(128));
    
    INSERT INTO @OverloadedServers
    SELECT DISTINCT ServerName
    FROM dbo.PerformanceMetrics
    WHERE CollectionTime > DATEADD(MINUTE, -5, GETUTCDATE())
    GROUP BY ServerName
    HAVING MAX(CASE 
                WHEN MetricName = 'CPU Usage' 
                THEN MetricValue 
                ELSE 0 
              END) > 85
    OR MAX(CASE 
            WHEN MetricName = 'Page life expectancy' 
            THEN MetricValue 
            ELSE 0 
           END) < 300;
    
    -- Adjust routing for overloaded servers
    UPDATE ar
    SET RoutingPercentage = RoutingPercentage * 0.8,
        LastModified = GETUTCDATE()
    FROM dbo.ApplicationRouting ar
    JOIN @OverloadedServers os
    ON ar.TargetServer = os.ServerName
    WHERE ar.IsActive = 1;
    
    -- Log adjustments
    INSERT INTO dbo.RoutingHistory
    (ApplicationId, RoutedServer, RequestType, 
     ExecutionTime, Duration_ms, WasSuccessful)
    SELECT 
        ar.ApplicationId,
        ar.TargetServer,
        'A', -- Automatic adjustment
        GETUTCDATE(),
        0,
        1
    FROM dbo.ApplicationRouting ar
    JOIN @OverloadedServers os
    ON ar.TargetServer = os.ServerName;
END;
```

## Migration Validation

### Traffic Analysis
```sql
CREATE VIEW dbo.TrafficAnalysis
AS
SELECT 
    ar.ApplicationName,
    ar.SourceServer,
    ar.TargetServer,
    ar.RoutingPercentage,
    COUNT(*) as RequestCount,
    AVG(CAST(rh.WasSuccessful as decimal(5,2))) * 100 as SuccessRate,
    AVG(rh.Duration_ms) as AvgDuration,
    MAX(rh.Duration_ms) as MaxDuration
FROM dbo.ApplicationRouting ar
JOIN dbo.RoutingHistory rh
ON ar.ApplicationId = rh.ApplicationId
WHERE rh.ExecutionTime > DATEADD(HOUR, -1, GETUTCDATE())
GROUP BY 
    ar.ApplicationName,
    ar.SourceServer,
    ar.TargetServer,
    ar.RoutingPercentage;
```

### Health Check System
```sql
CREATE PROCEDURE dbo.ValidateServerHealth
    @ServerName nvarchar(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check connectivity
    DECLARE @IsAlive bit = 0;
    BEGIN TRY
        DECLARE @SQL nvarchar(max) = '
        SELECT 1 FROM OPENQUERY(' + 
        QUOTENAME(@ServerName) + 
        ', ''SELECT 1 AS IsAlive'')';
        
        EXEC sp_executesql @SQL;
        SET @IsAlive = 1;
    END TRY
    BEGIN CATCH
        SET @IsAlive = 0;
    END CATCH;
    
    -- Check performance metrics
    IF @IsAlive = 1
    BEGIN
        -- Collect current metrics
        EXEC dbo.CollectPerformanceMetrics @ServerName;
        
        -- Analyze recent performance
        SELECT 
            'Health Check Result' as Report,
            CASE 
                WHEN MAX(CASE 
                         WHEN MetricName = 'CPU Usage' 
                         THEN MetricValue 
                         END) > 90 THEN 'Critical'
                WHEN AVG(CASE 
                         WHEN MetricName = 'Page life expectancy' 
                         THEN MetricValue 
                         END) < 300 THEN 'Warning'
                WHEN @IsAlive = 0 THEN 'Offline'
                ELSE 'Healthy'
            END as Status,
            MAX(CollectionTime) as LastCheck
        FROM dbo.PerformanceMetrics
        WHERE ServerName = @ServerName
        AND CollectionTime > DATEADD(MINUTE, -5, GETUTCDATE());
    END;
END;
```

This traffic routing framework provides a robust system for gradually migrating application workloads while maintaining performance and reliability monitoring.
