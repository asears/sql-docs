# Migration Scheduler and Orchestration

## Migration Task Scheduler

### Task Definition and Tracking
```sql
CREATE TABLE dbo.MigrationTasks
(
    TaskId int IDENTITY(1,1) PRIMARY KEY,
    TaskName nvarchar(200),
    TaskType nvarchar(50),  -- Schema, Data, Index, Stats, etc.
    ObjectName nvarchar(128),
    Priority int,
    EstimatedDuration int,  -- minutes
    Dependencies nvarchar(max),
    Status tinyint,         -- 0=Pending, 1=InProgress, 2=Complete, 3=Failed
    StartTime datetime2,
    EndTime datetime2,
    RetryCount int,
    ErrorMessage nvarchar(max)
);

CREATE TABLE dbo.MigrationWindows
(
    WindowId int IDENTITY(1,1) PRIMARY KEY,
    StartTime datetime2,
    EndTime datetime2,
    MaxConcurrent int,
    MaxIOPS int,
    MaxCPUPercent int,
    IsActive bit
);

CREATE TABLE dbo.TaskSchedule
(
    ScheduleId int IDENTITY(1,1) PRIMARY KEY,
    TaskId int,
    WindowId int,
    PlannedStartTime datetime2,
    ActualStartTime datetime2,
    Status tinyint,
    CONSTRAINT FK_TaskSchedule_Task FOREIGN KEY (TaskId)
        REFERENCES dbo.MigrationTasks(TaskId),
    CONSTRAINT FK_TaskSchedule_Window FOREIGN KEY (WindowId)
        REFERENCES dbo.MigrationWindows(WindowId)
);
```

### Task Orchestration
```sql
CREATE PROCEDURE dbo.ScheduleMigrationTasks
    @WindowStartTime datetime2,
    @WindowDuration int  -- minutes
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create migration window
    INSERT INTO dbo.MigrationWindows
    (StartTime, EndTime, MaxConcurrent, MaxIOPS, MaxCPUPercent, IsActive)
    VALUES
    (@WindowStartTime, 
     DATEADD(MINUTE, @WindowDuration, @WindowStartTime),
     4,   -- Allow 4 concurrent tasks
     5000,-- Max IOPS
     70,  -- Max CPU %
     1);
     
    DECLARE @WindowId int = SCOPE_IDENTITY();
    
    -- Schedule tasks within window
    WITH TaskPriority AS (
        SELECT 
            TaskId,
            TaskName,
            EstimatedDuration,
            ROW_NUMBER() OVER (
                ORDER BY 
                    Priority ASC,
                    EstimatedDuration DESC
            ) as ScheduleOrder
        FROM dbo.MigrationTasks
        WHERE Status = 0  -- Pending tasks
        AND TaskId NOT IN (
            SELECT TaskId 
            FROM dbo.TaskSchedule
            WHERE Status IN (0, 1)  -- Not yet completed
        )
    )
    INSERT INTO dbo.TaskSchedule
    (TaskId, WindowId, PlannedStartTime, Status)
    SELECT 
        TaskId,
        @WindowId,
        DATEADD(MINUTE, 
            (EstimatedDuration * ((ScheduleOrder - 1) / 4)), -- Divide by MaxConcurrent
            @WindowStartTime),
        0  -- Pending
    FROM TaskPriority
    WHERE DATEADD(MINUTE, 
          EstimatedDuration * ((ScheduleOrder - 1) / 4) + EstimatedDuration,
          @WindowStartTime) <= DATEADD(MINUTE, @WindowDuration, @WindowStartTime);
END;
```

### Task Execution Engine
```sql
CREATE PROCEDURE dbo.ExecuteMigrationTask
    @TaskId int
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @TaskType nvarchar(50);
    DECLARE @ObjectName nvarchar(128);
    
    SELECT 
        @TaskType = TaskType,
        @ObjectName = ObjectName
    FROM dbo.MigrationTasks
    WHERE TaskId = @TaskId;
    
    BEGIN TRY
        -- Update task status
        UPDATE dbo.MigrationTasks
        SET Status = 1,  -- In Progress
            StartTime = GETUTCDATE()
        WHERE TaskId = @TaskId;
        
        -- Execute based on task type
        IF @TaskType = 'Schema'
        BEGIN
            -- Schema migration logic
            EXEC dbo.MigrateTableSchema @ObjectName;
        END
        ELSE IF @TaskType = 'Data'
        BEGIN
            -- Data migration logic
            EXEC dbo.MigrateTableData @ObjectName;
        END
        ELSE IF @TaskType = 'Index'
        BEGIN
            -- Index rebuild logic
            EXEC dbo.RebuildTableIndexes @ObjectName;
        END
        ELSE IF @TaskType = 'Stats'
        BEGIN
            -- Statistics update logic
            EXEC dbo.UpdateTableStatistics @ObjectName;
        END
        
        -- Mark task as complete
        UPDATE dbo.MigrationTasks
        SET Status = 2,  -- Complete
            EndTime = GETUTCDATE()
        WHERE TaskId = @TaskId;
        
        -- Update schedule
        UPDATE dbo.TaskSchedule
        SET ActualStartTime = StartTime,
            Status = 2  -- Complete
        WHERE TaskId = @TaskId;
    END TRY
    BEGIN CATCH
        -- Handle failure
        UPDATE dbo.MigrationTasks
        SET Status = 3,  -- Failed
            EndTime = GETUTCDATE(),
            RetryCount = ISNULL(RetryCount, 0) + 1,
            ErrorMessage = ERROR_MESSAGE()
        WHERE TaskId = @TaskId;
        
        -- Update schedule
        UPDATE dbo.TaskSchedule
        SET Status = 3  -- Failed
        WHERE TaskId = @TaskId;
        
        -- Log error
        INSERT INTO dbo.MigrationLog
        (TaskId, LogLevel, Message, LogTime)
        VALUES
        (@TaskId, 'Error', ERROR_MESSAGE(), GETUTCDATE());
        
        -- Raise error for monitoring
        RAISERROR('Task %d failed: %s', 16, 1, 
                  @TaskId, ERROR_MESSAGE());
    END CATCH;
END;
```

### Resource Governor Integration
```sql
-- Create workload groups for migration
CREATE WORKLOAD GROUP MigrationGroup
WITH (
    MAX_DOP = 4,
    REQUEST_MAX_MEMORY_GRANT_PERCENT = 25,
    REQUEST_MAX_CPU_TIME_SEC = 0,
    REQUEST_MEMORY_GRANT_TIMEOUT_SEC = 0,
    MAX_CPU_PERCENT = 70,
    GROUP_MAX_REQUESTS = 0
);

-- Create classifier function
CREATE FUNCTION dbo.MigrationClassifier()
RETURNS sysname
WITH SCHEMABINDING
AS
BEGIN
    DECLARE @GroupName sysname;
    
    -- Check if connection is from migration process
    IF APP_NAME() LIKE 'SQLMigration%'
        SET @GroupName = 'MigrationGroup';
    ELSE
        SET @GroupName = 'default';
        
    RETURN @GroupName;
END;

-- Apply classifier
ALTER RESOURCE GOVERNOR
WITH (CLASSIFIER_FUNCTION = dbo.MigrationClassifier);
ALTER RESOURCE GOVERNOR RECONFIGURE;
```

### Progress Monitoring
```sql
CREATE VIEW dbo.MigrationProgress
AS
SELECT 
    TaskType,
    COUNT(*) as TotalTasks,
    SUM(CASE WHEN Status = 2 THEN 1 ELSE 0 END) as CompletedTasks,
    SUM(CASE WHEN Status = 1 THEN 1 ELSE 0 END) as InProgressTasks,
    SUM(CASE WHEN Status = 0 THEN 1 ELSE 0 END) as PendingTasks,
    SUM(CASE WHEN Status = 3 THEN 1 ELSE 0 END) as FailedTasks,
    AVG(CASE 
        WHEN Status = 2 
        THEN DATEDIFF(MINUTE, StartTime, EndTime)
        ELSE NULL 
    END) as AvgDurationMinutes,
    MAX(CASE 
        WHEN Status = 2 
        THEN DATEDIFF(MINUTE, StartTime, EndTime)
        ELSE NULL 
    END) as MaxDurationMinutes,
    SUM(CASE 
        WHEN Status = 2 
        THEN DATEDIFF(MINUTE, StartTime, EndTime)
        ELSE 0 
    END) as TotalDurationMinutes
FROM dbo.MigrationTasks
GROUP BY TaskType;
```

This migration scheduler provides a robust framework for coordinating all aspects of the database migration, ensuring optimal resource utilization while maintaining system stability. Would you like me to continue by implementing the specific task handlers for schema migration, data movement, or another aspect of the migration process?
