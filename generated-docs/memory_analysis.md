# SQL Server Memory Analysis Framework

## Memory Pressure Detection

### Memory Clerk Analysis
```sql
CREATE TABLE dbo.MemoryClerkHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    ClerkType nvarchar(60),
    MemoryUsedMB decimal(18,2),
    MemoryAllocatedMB decimal(18,2),
    AllocationCount int,
    PartitionCount int,
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.MonitorMemoryClerks
    @WarningThresholdMB int = 1000,
    @CriticalThresholdMB int = 5000
AS
BEGIN
    -- Capture current memory clerk usage
    INSERT INTO dbo.MemoryClerkHistory
    SELECT 
        type as ClerkType,
        pages_kb / 1024.0 as MemoryUsedMB,
        virtual_memory_reserved_kb / 1024.0 as MemoryAllocatedMB,
        virtual_memory_committed_kb / 1024.0 as AllocationCount,
        awe_allocated_kb as PartitionCount,
        GETUTCDATE() as CollectionTime
    FROM sys.dm_os_memory_clerks
    WHERE pages_kb > 0;

    -- Analyze memory pressure patterns
    WITH MemoryTrends AS (
        SELECT 
            ClerkType,
            MemoryUsedMB,
            MemoryAllocatedMB,
            LAG(MemoryUsedMB) OVER (
                PARTITION BY ClerkType 
                ORDER BY CollectionTime
            ) as PreviousUsage,
            CollectionTime
        FROM dbo.MemoryClerkHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        ClerkType,
        MemoryUsedMB,
        MemoryAllocatedMB,
        ((MemoryUsedMB - PreviousUsage) * 100.0) / 
            NULLIF(PreviousUsage, 0) as GrowthPercent,
        CASE 
            WHEN MemoryUsedMB > @CriticalThresholdMB THEN 'Critical'
            WHEN MemoryUsedMB > @WarningThresholdMB THEN 'Warning'
            ELSE 'Normal'
        END as PressureStatus,
        CASE 
            WHEN MemoryUsedMB > @CriticalThresholdMB 
            THEN 'Consider memory clerk cleanup or increasing max server memory'
            WHEN MemoryUsedMB > @WarningThresholdMB 
            THEN 'Monitor for further growth'
            ELSE 'No action needed'
        END as RecommendedAction
    FROM MemoryTrends
    WHERE MemoryUsedMB > @WarningThresholdMB
    ORDER BY MemoryUsedMB DESC;
END;
```

### Buffer Pool Analysis
```sql
CREATE TABLE dbo.BufferPoolHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    DatabaseId int,
    FileId int,
    PageType nvarchar(60),
    PageCount bigint,
    CachedSizeMB decimal(18,2),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.AnalyzeBufferPoolUsage
AS
BEGIN
    -- Capture current buffer pool state
    INSERT INTO dbo.BufferPoolHistory
    SELECT 
        database_id,
        file_id,
        page_type,
        COUNT(*) as page_count,
        COUNT(*) * 8.0 / 1024 as cached_size_mb,
        GETUTCDATE()
    FROM sys.dm_os_buffer_descriptors
    GROUP BY database_id, file_id, page_type;

    -- Analyze buffer distribution
    WITH BufferDistribution AS (
        SELECT 
            DB_NAME(DatabaseId) as DatabaseName,
            PageType,
            CachedSizeMB,
            LAG(CachedSizeMB) OVER (
                PARTITION BY DatabaseId, PageType 
                ORDER BY CollectionTime
            ) as PreviousCachedSize,
            CollectionTime
        FROM dbo.BufferPoolHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
    )
    SELECT 
        DatabaseName,
        PageType,
        CachedSizeMB,
        ((CachedSizeMB - PreviousCachedSize) * 100.0) / 
            NULLIF(PreviousCachedSize, 0) as CacheChangePercent,
        CASE 
            WHEN CachedSizeMB > 10240 THEN 'High Cache Usage'
            WHEN CachedSizeMB > 5120 THEN 'Moderate Cache Usage'
            ELSE 'Normal Cache Usage'
        END as CacheStatus
    FROM BufferDistribution
    ORDER BY CachedSizeMB DESC;

    -- Analyze buffer churn
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        OBJECT_NAME(p.object_id) as TableName,
        p.index_id,
        i.name as IndexName,
        COUNT(*) as BufferCount,
        SUM(CASE 
            WHEN is_modified = 1 THEN 1 
            ELSE 0 
        END) as DirtyPages,
        SUM(CASE 
            WHEN is_modified = 1 THEN 1 
            ELSE 0 
        END) * 100.0 / COUNT(*) as DirtyPagePercent
    FROM sys.dm_os_buffer_descriptors bd
    JOIN sys.allocation_units au 
        ON bd.allocation_unit_id = au.allocation_unit_id
    JOIN sys.partitions p 
        ON au.container_id = p.hobt_id
    LEFT JOIN sys.indexes i 
        ON p.object_id = i.object_id 
        AND p.index_id = i.index_id
    WHERE bd.database_id = DB_ID()
    GROUP BY 
        database_id, p.object_id, 
        p.index_id, i.name
    HAVING COUNT(*) > 1000
    ORDER BY BufferCount DESC;
END;
```

### Memory Grant Analysis
```sql
CREATE TABLE dbo.MemoryGrantHistory
(
    HistoryId bigint IDENTITY(1,1) PRIMARY KEY,
    SessionId int,
    RequestId int,
    GrantedMemoryKB bigint,
    RequestedMemoryKB bigint,
    RequiredMemoryKB bigint,
    QueryPlan xml,
    QueryText nvarchar(max),
    WaitType nvarchar(60),
    CollectionTime datetime2
);

CREATE PROCEDURE dbo.TrackMemoryGrants
AS
BEGIN
    -- Capture current memory grants
    INSERT INTO dbo.MemoryGrantHistory
    SELECT 
        mg.session_id,
        mg.request_id,
        mg.granted_memory_kb,
        mg.requested_memory_kb,
        mg.required_memory_kb,
        qp.query_plan,
        st.text,
        r.wait_type,
        GETUTCDATE()
    FROM sys.dm_exec_query_memory_grants mg
    CROSS APPLY sys.dm_exec_sql_text(mg.sql_handle) st
    CROSS APPLY sys.dm_exec_query_plan(mg.plan_handle) qp
    LEFT JOIN sys.dm_exec_requests r 
        ON mg.session_id = r.session_id
        AND mg.request_id = r.request_id;

    -- Analyze memory grant patterns
    WITH GrantAnalysis AS (
        SELECT 
            QueryText,
            AVG(GrantedMemoryKB) as AvgGrantedKB,
            MAX(GrantedMemoryKB) as MaxGrantedKB,
            AVG(RequestedMemoryKB) as AvgRequestedKB,
            MAX(RequestedMemoryKB) as MaxRequestedKB,
            COUNT(*) as ExecutionCount,
            SUM(CASE 
                WHEN WaitType LIKE 'RESOURCE_SEMAPHORE%' 
                THEN 1 ELSE 0 
            END) as MemoryWaitCount
        FROM dbo.MemoryGrantHistory
        WHERE CollectionTime >= DATEADD(HOUR, -1, GETUTCDATE())
        GROUP BY QueryText
    )
    SELECT 
        QueryText,
        AvgGrantedKB / 1024.0 as AvgGrantedMB,
        MaxGrantedKB / 1024.0 as MaxGrantedMB,
        AvgRequestedKB / 1024.0 as AvgRequestedMB,
        MaxRequestedKB / 1024.0 as MaxRequestedMB,
        ExecutionCount,
        MemoryWaitCount,
        CASE 
            WHEN MemoryWaitCount > 0 THEN 'Memory Pressure'
            WHEN MaxRequestedKB > 1048576 THEN 'High Memory Request'
            ELSE 'Normal'
        END as GrantStatus,
        CASE 
            WHEN MemoryWaitCount > 0 
            THEN 'Consider query optimization or increasing max memory'
            WHEN MaxRequestedKB > 1048576 
            THEN 'Review query for optimization opportunities'
            ELSE 'No action needed'
        END as Recommendation
    FROM GrantAnalysis
    WHERE ExecutionCount > 5
    OR MemoryWaitCount > 0
    ORDER BY MaxRequestedKB DESC;
END;
```

### NUMA Node Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeNUMAMemory
AS
BEGIN
    -- Analyze NUMA node memory distribution
    SELECT 
        memory_node_id,
        virtual_address_space_reserved_kb / 1024.0 as ReservedMB,
        virtual_address_space_committed_kb / 1024.0 as CommittedMB,
        locked_page_allocations_kb / 1024.0 as LockedPagesMB,
        pages_kb / 1024.0 as PagesMB,
        foreign_committed_kb / 1024.0 as ForeignCommittedMB,
        CAST(100.0 * virtual_address_space_committed_kb / 
            NULLIF(virtual_address_space_reserved_kb, 0) 
            as decimal(5,2)) as CommitPercent,
        CASE 
            WHEN foreign_committed_kb > 1048576 
            THEN 'High Foreign Memory'
            WHEN virtual_address_space_committed_kb > 
                 0.9 * virtual_address_space_reserved_kb 
            THEN 'High Commit Rate'
            ELSE 'Normal'
        END as NodeStatus
    FROM sys.dm_os_memory_nodes
    WHERE memory_node_id != 64;  -- Exclude DAC node

    -- Analyze CPU schedulers per NUMA node
    SELECT 
        parent_node_id,
        COUNT(*) as SchedulerCount,
        SUM(current_tasks_count) as CurrentTasks,
        SUM(runnable_tasks_count) as RunnableTasks,
        SUM(active_workers_count) as ActiveWorkers,
        AVG(load_factor) as AvgLoadFactor,
        CASE 
            WHEN AVG(load_factor) > 75 THEN 'High Load'
            WHEN AVG(load_factor) > 50 THEN 'Moderate Load'
            ELSE 'Normal Load'
        END as LoadStatus
    FROM sys.dm_os_schedulers
    WHERE scheduler_id < 255  -- Exclude internal schedulers
    GROUP BY parent_node_id;
END;
```

### Performance Counter Analysis
```sql
CREATE PROCEDURE dbo.AnalyzeMemoryCounters
AS
BEGIN
    -- Collect key memory-related performance counters
    SELECT 
        counter_name,
        cntr_value,
        CASE counter_name
            WHEN 'Page life expectancy' 
            THEN CASE 
                WHEN cntr_value < 300 THEN 'Critical'
                WHEN cntr_value < 1000 THEN 'Warning'
                ELSE 'Normal'
            END
            WHEN 'Free list stalls/sec' 
            THEN CASE 
                WHEN cntr_value > 2 THEN 'Critical'
                WHEN cntr_value > 0 THEN 'Warning'
                ELSE 'Normal'
            END
            WHEN 'Memory Grants Pending' 
            THEN CASE 
                WHEN cntr_value > 0 THEN 'Critical'
                ELSE 'Normal'
            END
            ELSE 'N/A'
        END as Status
    FROM sys.dm_os_performance_counters
    WHERE counter_name IN (
        'Page life expectancy',
        'Free list stalls/sec',
        'Memory Grants Pending',
        'Target Server Memory (KB)',
        'Total Server Memory (KB)'
    );

    -- Calculate memory pressure indicators
    SELECT 
        'Memory Pressure Analysis' as Analysis,
        physical_memory_in_use_kb / 1024.0 as PhysicalMemoryUsedMB,
        virtual_address_space_committed_kb / 1024.0 as CommittedMemoryMB,
        available_commit_limit_kb / 1024.0 as AvailableCommitMB,
        process_physical_memory_low as PhysicalMemoryLow,
        process_virtual_memory_low as VirtualMemoryLow,
        CASE 
            WHEN process_physical_memory_low = 1 
                 OR process_virtual_memory_low = 1 
            THEN 'Critical'
            WHEN physical_memory_in_use_kb > 
                 0.9 * available_commit_limit_kb 
            THEN 'Warning'
            ELSE 'Normal'
        END as MemoryPressureStatus
    FROM sys.dm_os_process_memory;
END;
```

This memory analysis framework provides comprehensive tools for:
1. Monitoring memory clerk allocation and usage patterns
2. Analyzing buffer pool distribution and churn
3. Tracking memory grants and identifying pressure points
4. Analyzing NUMA memory distribution
5. Monitoring key performance counters

Would you like me to continue with any specific aspect of SQL Server performance monitoring or troubleshooting?
