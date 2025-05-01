# SQL Server Performance Monitoring Dashboards

## Performance KPI Framework

### Real-Time Monitoring Dashboard
```sql
CREATE TABLE dbo.PerformanceKPIs
(
    KPIId int IDENTITY(1,1) PRIMARY KEY,
    MetricName nvarchar(100),
    CurrentValue decimal(18,2),
    TargetValue decimal(18,2),
    ThresholdValue decimal(18,2),
    CollectionTime datetime2,
    Status varchar(20)
);

CREATE TABLE dbo.KPIHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    KPIId int,
    MetricValue decimal(18,2),
    CollectionTime datetime2,
    CONSTRAINT FK_KPIHistory_KPI 
        FOREIGN KEY (KPIId) 
        REFERENCES dbo.PerformanceKPIs(KPIId)
);

-- Collect core performance metrics
CREATE PROCEDURE dbo.CollectPerformanceKPIs
AS
BEGIN
    SET NOCOUNT ON;

    -- CPU Usage
    INSERT INTO dbo.KPIHistory (KPIId, MetricValue, CollectionTime)
    SELECT 
        1, -- CPU KPI
        (SELECT TOP 1 
            100 - SystemIdle 
         FROM (
             SELECT CAST(record as xml).value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') as SystemIdle
             FROM sys.dm_os_ring_buffers 
             WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
             ORDER BY timestamp DESC
         ) as CPU),
        GETUTCDATE();

    -- Memory Pressure
    INSERT INTO dbo.KPIHistory (KPIId, MetricValue, CollectionTime)
    SELECT 
        2, -- Memory KPI
        (SELECT cntr_value 
         FROM sys.dm_os_performance_counters
         WHERE counter_name = 'Page life expectancy' 
         AND object_name LIKE '%Buffer Manager%'),
        GETUTCDATE();

    -- Disk Latency
    INSERT INTO dbo.KPIHistory (KPIId, MetricValue, CollectionTime)
    SELECT 
        3, -- Disk KPI
        AVG(io_stall_read_ms + io_stall_write_ms) / 
            NULLIF(num_of_reads + num_of_writes, 0),
        GETUTCDATE()
    FROM sys.dm_io_virtual_file_stats(NULL, NULL);

    -- Update current values
    UPDATE p
    SET 
        CurrentValue = h.MetricValue,
        Status = CASE 
            WHEN h.MetricValue > p.ThresholdValue THEN 'Critical'
            WHEN h.MetricValue > p.TargetValue THEN 'Warning'
            ELSE 'Healthy'
        END,
        CollectionTime = h.CollectionTime
    FROM dbo.PerformanceKPIs p
    JOIN (
        SELECT 
            KPIId, 
            MetricValue, 
            CollectionTime,
            ROW_NUMBER() OVER (PARTITION BY KPIId ORDER BY CollectionTime DESC) as rn
        FROM dbo.KPIHistory
    ) h ON p.KPIId = h.KPIId
    WHERE h.rn = 1;
END;
```

### Performance Trend Analysis

```sql
CREATE PROCEDURE dbo.AnalyzePerformanceTrends
    @DaysBack int = 7
AS
BEGIN
    -- Calculate hourly averages
    WITH HourlyMetrics AS (
        SELECT 
            KPIId,
            DATEADD(HOUR, DATEDIFF(HOUR, 0, CollectionTime), 0) as HourBucket,
            AVG(MetricValue) as AvgValue,
            MIN(MetricValue) as MinValue,
            MAX(MetricValue) as MaxValue,
            STDEV(MetricValue) as StdDev
        FROM dbo.KPIHistory
        WHERE CollectionTime >= DATEADD(DAY, -@DaysBack, GETUTCDATE())
        GROUP BY 
            KPIId,
            DATEADD(HOUR, DATEDIFF(HOUR, 0, CollectionTime), 0)
    )
    SELECT 
        p.MetricName,
        hm.HourBucket,
        hm.AvgValue,
        hm.MinValue,
        hm.MaxValue,
        hm.StdDev,
        CASE 
            WHEN hm.AvgValue > p.ThresholdValue THEN 'Critical'
            WHEN hm.AvgValue > p.TargetValue THEN 'Warning'
            ELSE 'Healthy'
        END as Status,
        -- Calculate trend
        LAG(hm.AvgValue) OVER (PARTITION BY p.KPIId ORDER BY hm.HourBucket) as PreviousValue,
        ((hm.AvgValue - LAG(hm.AvgValue) OVER (PARTITION BY p.KPIId ORDER BY hm.HourBucket)) * 100.0 / 
            NULLIF(LAG(hm.AvgValue) OVER (PARTITION BY p.KPIId ORDER BY hm.HourBucket), 0)) as TrendPercent
    FROM HourlyMetrics hm
    JOIN dbo.PerformanceKPIs p ON hm.KPIId = p.KPIId
    ORDER BY 
        p.MetricName,
        hm.HourBucket;
END;
```

### Baseline Comparison

```sql
CREATE TABLE dbo.PerformanceBaselines
(
    BaselineId int IDENTITY(1,1) PRIMARY KEY,
    MetricName nvarchar(100),
    TimeWindow varchar(20),
    BaselineValue decimal(18,2),
    StandardDeviation decimal(18,2),
    SampleSize int,
    StartDate datetime2,
    EndDate datetime2
);

CREATE PROCEDURE dbo.EstablishBaseline
    @MetricName nvarchar(100),
    @TimeWindow varchar(20),
    @DaysToAnalyze int = 30
AS
BEGIN
    -- Calculate baseline metrics
    WITH MetricStats AS (
        SELECT 
            kpi.MetricName,
            AVG(h.MetricValue) as BaselineValue,
            STDEV(h.MetricValue) as StandardDeviation,
            COUNT(*) as SampleSize
        FROM dbo.KPIHistory h
        JOIN dbo.PerformanceKPIs kpi ON h.KPIId = kpi.KPIId
        WHERE 
            kpi.MetricName = @MetricName
            AND h.CollectionTime >= DATEADD(DAY, -@DaysToAnalyze, GETUTCDATE())
            AND DATENAME(dw, h.CollectionTime) NOT IN ('Saturday', 'Sunday')
            AND DATEPART(HOUR, h.CollectionTime) BETWEEN 9 AND 17
        GROUP BY kpi.MetricName
    )
    INSERT INTO dbo.PerformanceBaselines
    (MetricName, TimeWindow, BaselineValue, StandardDeviation, 
     SampleSize, StartDate, EndDate)
    SELECT 
        MetricName,
        @TimeWindow,
        BaselineValue,
        StandardDeviation,
        SampleSize,
        DATEADD(DAY, -@DaysToAnalyze, GETUTCDATE()),
        GETUTCDATE()
    FROM MetricStats;
END;

CREATE PROCEDURE dbo.CompareToBaseline
    @MetricName nvarchar(100)
AS
BEGIN
    WITH CurrentMetrics AS (
        SELECT 
            kpi.MetricName,
            AVG(h.MetricValue) as CurrentValue,
            STDEV(h.MetricValue) as CurrentStdDev
        FROM dbo.KPIHistory h
        JOIN dbo.PerformanceKPIs kpi ON h.KPIId = kpi.KPIId
        WHERE 
            kpi.MetricName = @MetricName
            AND h.CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY kpi.MetricName
    )
    SELECT 
        cm.MetricName,
        b.TimeWindow,
        b.BaselineValue,
        b.StandardDeviation as BaselineStdDev,
        cm.CurrentValue,
        cm.CurrentStdDev,
        ((cm.CurrentValue - b.BaselineValue) * 100.0) / 
            NULLIF(b.BaselineValue, 0) as DeviationPercent,
        CASE 
            WHEN ABS(cm.CurrentValue - b.BaselineValue) > 
                 (2 * b.StandardDeviation) THEN 'Significant Deviation'
            WHEN ABS(cm.CurrentValue - b.BaselineValue) > 
                 b.StandardDeviation THEN 'Moderate Deviation'
            ELSE 'Within Normal Range'
        END as DeviationStatus
    FROM CurrentMetrics cm
    JOIN dbo.PerformanceBaselines b 
        ON cm.MetricName = b.MetricName
    WHERE b.EndDate = (
        SELECT MAX(EndDate)
        FROM dbo.PerformanceBaselines
        WHERE MetricName = @MetricName
    );
END;
```

### Alert Configuration

```sql
CREATE TABLE dbo.AlertConfigurations
(
    AlertId int IDENTITY(1,1) PRIMARY KEY,
    KPIId int,
    AlertLevel varchar(20),
    ThresholdValue decimal(18,2),
    ConsecutiveBreaches int,
    NotificationList nvarchar(max),
    IsEnabled bit,
    CONSTRAINT FK_Alert_KPI 
        FOREIGN KEY (KPIId) 
        REFERENCES dbo.PerformanceKPIs(KPIId)
);

CREATE TABLE dbo.AlertHistory
(
    AlertHistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    AlertId int,
    MetricValue decimal(18,2),
    ThresholdValue decimal(18,2),
    AlertTime datetime2,
    ResolutionTime datetime2,
    Duration int,  -- minutes
    NotifiedUsers nvarchar(max),
    CONSTRAINT FK_AlertHistory_Alert 
        FOREIGN KEY (AlertId) 
        REFERENCES dbo.AlertConfigurations(AlertId)
);

CREATE PROCEDURE dbo.ProcessAlerts
AS
BEGIN
    -- Check for new alerts
    INSERT INTO dbo.AlertHistory
    (AlertId, MetricValue, ThresholdValue, AlertTime, NotifiedUsers)
    SELECT 
        ac.AlertId,
        kh.MetricValue,
        ac.ThresholdValue,
        GETUTCDATE(),
        ac.NotificationList
    FROM dbo.AlertConfigurations ac
    JOIN dbo.PerformanceKPIs kpi ON ac.KPIId = kpi.KPIId
    JOIN dbo.KPIHistory kh ON kpi.KPIId = kh.KPIId
    WHERE 
        ac.IsEnabled = 1
        AND kh.CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        AND kh.MetricValue > ac.ThresholdValue
        AND NOT EXISTS (
            SELECT 1 
            FROM dbo.AlertHistory ah
            WHERE ah.AlertId = ac.AlertId
            AND ah.ResolutionTime IS NULL
        );

    -- Update resolved alerts
    UPDATE ah
    SET 
        ResolutionTime = GETUTCDATE(),
        Duration = DATEDIFF(MINUTE, ah.AlertTime, GETUTCDATE())
    FROM dbo.AlertHistory ah
    JOIN dbo.AlertConfigurations ac ON ah.AlertId = ac.AlertId
    JOIN dbo.PerformanceKPIs kpi ON ac.KPIId = kpi.KPIId
    JOIN dbo.KPIHistory kh ON kpi.KPIId = kh.KPIId
    WHERE 
        ah.ResolutionTime IS NULL
        AND kh.CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        AND kh.MetricValue <= ac.ThresholdValue;
END;
```

This monitoring framework provides comprehensive dashboards and KPIs to track SQL Server performance metrics, establish baselines, and manage alerts. Would you like me to add more specific monitoring components or focus on another aspect of the implementation?
