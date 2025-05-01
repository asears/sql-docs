# SQL Server TempDB Analysis Framework

## TempDB Monitoring Framework

### Space Usage Analysis
```sql
CREATE TABLE dbo.TempDBSpaceMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    FileId int,
    FileGroup int,
    TotalSpaceMB decimal(18,2),
    UsedSpaceMB decimal(18,2),
    FreeSpaceMB decimal(18,2),
    FilePath nvarchar(260),
    FileGrowthSettings nvarchar(100),
    IsAutoGrowthEnabled bit,
    LastGrowthTime datetime2,
    LastShrinkTime datetime2,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorTempDBSpace
    @HighUtilizationThreshold decimal(5,2) = 80.0,
    @LowFreeSpaceThresholdMB decimal(18,2) = 1024.0
AS
BEGIN
    -- Capture TempDB metrics
    INSERT INTO dbo.TempDBSpaceMetrics
    SELECT 
        f.file_id,
        f.data_space_id,
        f.size * 8.0 / 1024 as TotalSpaceMB,
        FILEPROPERTY(f.name, 'SpaceUsed') * 8.0 / 1024 as UsedSpaceMB,
        (f.size - FILEPROPERTY(f.name, 'SpaceUsed')) * 8.0 / 1024 
            as FreeSpaceMB,
        f.physical_name,
        CASE 
            WHEN is_percent_growth = 1 
            THEN CAST(growth as varchar(10)) + '%'
            ELSE CAST(growth * 8.0 / 1024 as varchar(10)) + ' MB'
        END as FileGrowthSettings,
        CAST(
            CASE WHEN growth > 0 THEN 1 ELSE 0 END as bit
        ) as IsAutoGrowthEnabled,
        NULL as LastGrowthTime,  -- To be populated from trace data
        NULL as LastShrinkTime,  -- To be populated from trace data
        GETUTCDATE()
    FROM tempdb.sys.database_files f;

    -- Analyze space usage patterns
    WITH SpaceMetrics AS (
        SELECT 
            FileId,
            TotalSpaceMB,
            UsedSpaceMB,
            FreeSpaceMB,
            FilePath,
            FileGrowthSettings,
            IsAutoGrowthEnabled,
            UsedSpaceMB * 100.0 / TotalSpaceMB as UtilizationPercent,
            LAG(UsedSpaceMB) OVER (
                PARTITION BY FileId 
                ORDER BY CollectionTime
            ) as PreviousUsedSpace
        FROM dbo.TempDBSpaceMetrics
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        FileId,
        TotalSpaceMB,
        UsedSpaceMB,
        FreeSpaceMB,
        FilePath,
        FileGrowthSettings,
        IsAutoGrowthEnabled,
        UtilizationPercent,
        CASE 
            WHEN UtilizationPercent > @HighUtilizationThreshold 
                 AND FreeSpaceMB < @LowFreeSpaceThresholdMB 
            THEN 'Critical Space Pressure'
            WHEN UtilizationPercent > @HighUtilizationThreshold 
            THEN 'High Utilization'
            WHEN FreeSpaceMB < @LowFreeSpaceThresholdMB 
            THEN 'Low Free Space'
            WHEN UsedSpaceMB > COALESCE(PreviousUsedSpace, 0) * 1.5 
            THEN 'Rapid Growth'
            ELSE 'Normal'
        END as SpaceStatus,
        CASE 
            WHEN UtilizationPercent > @HighUtilizationThreshold 
                 AND FreeSpaceMB < @LowFreeSpaceThresholdMB 
            THEN 'Immediate action required:
                  1. Add tempdb files
                  2. Increase file size
                  3. Review large temp table usage'
            WHEN UtilizationPercent > @HighUtilizationThreshold 
            THEN 'Monitor growth patterns'
            WHEN FreeSpaceMB < @LowFreeSpaceThresholdMB 
            THEN 'Plan for space increase'
            WHEN UsedSpaceMB > COALESCE(PreviousUsedSpace, 0) * 1.5 
            THEN 'Investigate growth cause'
            ELSE 'No action needed'
        END as Recommendation
    FROM SpaceMetrics
    WHERE UtilizationPercent > @HighUtilizationThreshold
    OR FreeSpaceMB < @LowFreeSpaceThresholdMB
    OR UsedSpaceMB > COALESCE(PreviousUsedSpace, 0) * 1.5
    ORDER BY 
        CASE 
            WHEN UtilizationPercent > @HighUtilizationThreshold 
                 AND FreeSpaceMB < @LowFreeSpaceThresholdMB THEN 1
            WHEN UtilizationPercent > @HighUtilizationThreshold THEN 2
            WHEN FreeSpaceMB < @LowFreeSpaceThresholdMB THEN 3
            ELSE 4
        END,
        UtilizationPercent DESC;
END;
```

### Contention Analysis
```sql
CREATE TABLE dbo.TempDBContentionMetrics
(
    MetricId bigint IDENTITY(1,1) PRIMARY KEY,
    WaitType nvarchar(60),
    WaitingTaskCount int,
    WaitTimeMs bigint,
    SignalWaitTimeMs bigint,
    ResourceDescription nvarchar(max),
    SessionId int,
    RequestId int,
    BlockingSessionId int,
    QueryText nvarchar(max),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeTempDBContention
    @HighWaitThresholdMs int = 1000,
    @ContentionThreshold int = 5
AS
BEGIN
    -- Capture contention metrics
    INSERT INTO dbo.TempDBContentionMetrics
    SELECT 
        tws.wait_type,
        tws.waiting_tasks_count,
        tws.wait_time_ms,
        tws.signal_wait_time_ms,
        tws.resource_description,
        er.session_id,
        er.request_id,
        er.blocking_session_id,
        SUBSTRING(
            qt.text,
            er.statement_start_offset / 2 + 1,
            (CASE 
                WHEN er.statement_end_offset = -1 
                THEN LEN(CONVERT(nvarchar(max), qt.text)) * 2
                ELSE er.statement_end_offset
            END - er.statement_start_offset) / 2
        ) as QueryText,
        GETUTCDATE()
    FROM sys.dm_os_waiting_tasks tws
    JOIN sys.dm_exec_requests er 
        ON tws.session_id = er.session_id
    CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) qt
    WHERE tws.wait_type LIKE 'PAGELATCH%'
    AND tws.resource_description LIKE '2:%';

    -- Analyze contention patterns
    WITH ContentionMetrics AS (
        SELECT 
            WaitType,
            ResourceDescription,
            COUNT(DISTINCT SessionId) as ConcurrentWaiters,
            AVG(WaitTimeMs) as AvgWaitTimeMs,
            MAX(WaitTimeMs) as MaxWaitTimeMs,
            SUM(WaitingTaskCount) as TotalWaits,
            STRING_AGG(
                CAST(SessionId as varchar(10)) + 
                CASE 
                    WHEN BlockingSessionId IS NOT NULL 
                    THEN ' <- ' + CAST(BlockingSessionId as varchar(10))
                    ELSE ''
                END,
                ', '
            ) as WaitChain
        FROM dbo.TempDBContentionMetrics
        WHERE CollectionTime >= DATEADD(MINUTE, -5, GETUTCDATE())
        GROUP BY WaitType, ResourceDescription
    )
    SELECT 
        WaitType,
        ResourceDescription,
        ConcurrentWaiters,
        AvgWaitTimeMs,
        MaxWaitTimeMs,
        TotalWaits,
        WaitChain,
        CASE 
            WHEN ConcurrentWaiters >= @ContentionThreshold 
                 AND MaxWaitTimeMs >= @HighWaitThresholdMs 
            THEN 'High Contention'
            WHEN ConcurrentWaiters >= @ContentionThreshold 
            THEN 'Moderate Contention'
            WHEN MaxWaitTimeMs >= @HighWaitThresholdMs 
            THEN 'Long Waits'
            ELSE 'Normal'
        END as ContentionStatus,
        CASE 
            WHEN ConcurrentWaiters >= @ContentionThreshold 
                 AND MaxWaitTimeMs >= @HighWaitThresholdMs 
            THEN 'Consider:
                  1. Adding TempDB files
                  2. Moving heavily accessed temp tables to memory
                  3. Reviewing concurrent workload patterns'
            WHEN ConcurrentWaiters >= @ContentionThreshold 
            THEN 'Monitor wait patterns'
            WHEN MaxWaitTimeMs >= @HighWaitThresholdMs 
            THEN 'Investigate long-running operations'
            ELSE 'No action needed'
        END as Recommendation
    FROM ContentionMetrics
    WHERE ConcurrentWaiters >= @ContentionThreshold
    OR MaxWaitTimeMs >= @HighWaitThresholdMs
    ORDER BY 
        CASE 
            WHEN ConcurrentWaiters >= @ContentionThreshold 
                 AND MaxWaitTimeMs >= @HighWaitThresholdMs THEN 1
            WHEN ConcurrentWaiters >= @ContentionThreshold THEN 2
            ELSE 3
        END,
        MaxWaitTimeMs DESC;
END;
```

### TempDB Consumer Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeTempDBConsumers
AS
BEGIN
    -- Analyze tempdb usage by session
    SELECT 
        s.session_id,
        s.login_name,
        s.host_name,
        s.program_name,
        SUM(tsu.user_objects_alloc_page_count) * 8.0 / 1024 
            as UserObjectsMB,
        SUM(tsu.internal_objects_alloc_page_count) * 8.0 / 1024 
            as InternalObjectsMB,
        SUM(tsu.user_objects_dealloc_page_count) * 8 / 1024 
            as DeallocatedUserObjectsMB,
        st.text as LastQuery,
        CASE 
            WHEN SUM(
                tsu.user_objects_alloc_page_count + 
                tsu.internal_objects_alloc_page_count
            ) * 8.0 / 1024 > 1024  -- 1GB
            THEN 'High Usage'
            WHEN SUM(
                tsu.user_objects_dealloc_page_count
            ) * 8.0 / 1024 > 1024 
            THEN 'High Churn'
            ELSE 'Normal'
        END as UsagePattern,
        CASE 
            WHEN SUM(
                tsu.user_objects_alloc_page_count + 
                tsu.internal_objects_alloc_page_count
            ) * 8.0 / 1024 > 1024 
            THEN 'Review temp space usage patterns'
            WHEN SUM(
                tsu.user_objects_dealloc_page_count
            ) * 8.0 / 1024 > 1024 
            THEN 'Investigate object lifecycle'
            ELSE 'No action needed'
        END as Recommendation
    FROM sys.dm_db_session_space_usage tsu
    JOIN sys.dm_exec_sessions s 
        ON tsu.session_id = s.session_id
    LEFT JOIN sys.dm_exec_connections c 
        ON s.session_id = c.session_id
    OUTER APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) st
    WHERE s.session_id > 50  -- Exclude system sessions
    GROUP BY 
        s.session_id,
        s.login_name,
        s.host_name,
        s.program_name,
        st.text
    HAVING SUM(
        tsu.user_objects_alloc_page_count + 
        tsu.internal_objects_alloc_page_count
    ) * 8.0 / 1024 > 100  -- Show sessions using more than 100MB
    ORDER BY 
        SUM(
            tsu.user_objects_alloc_page_count + 
            tsu.internal_objects_alloc_page_count
        ) DESC;
END;
```

This TempDB analysis framework provides comprehensive tools for:
1. Monitoring TempDB space usage and growth patterns
2. Analyzing contention and wait statistics
3. Tracking heavy TempDB consumers
4. Optimizing TempDB performance

Would you like me to continue with another aspect of SQL Server performance monitoring or troubleshooting?
