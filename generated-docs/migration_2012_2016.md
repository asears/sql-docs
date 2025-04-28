# SQL Server Migration Guide: 2012 to 2016

## Migration Scenario Analysis

### Key Assumptions
- Large 5TB database with BLOB storage
- Multiple application types (mobile, web, Windows, reporting)
- Legacy application in SQL 2012 compatibility mode
- 24/7 operation requirements
- Minimal downtime requirements
- Complex data access patterns

### Risks
1. Application Compatibility
   - Legacy code dependencies
   - Deprecated feature usage
   - Unsupported API calls
   - Query plan changes

2. Performance Impact
   - Data movement overhead
   - Replication lag
   - Storage I/O bottlenecks
   - Network bandwidth limitations

3. Resource Constraints
   - Storage capacity requirements
   - Memory usage during migration
   - CPU intensive operations
   - Network throughput limitations

### Constraints
1. Operational
   - Minimal downtime windows
   - Business continuity requirements
   - Data consistency requirements
   - Application availability SLAs

2. Technical
   - Hardware limitations
   - Network bandwidth
   - Storage IOPS
   - Memory constraints

## Feature Differences and Improvements

### Major New Features in SQL Server 2016

1. Query Store
   - Automatic query performance monitoring
   - Plan forcing capabilities
   - Historical performance data retention
   - Performance regression detection

2. Always Encrypted
   - Column-level encryption
   - Client-side encryption
   - Key management integration
   - Transparent to applications

3. JSON Support
   - Native JSON parsing
   - FOR JSON clause
   - JSON data type functions
   - JSON indexing strategies

4. Temporal Tables
   - System-versioned tables
   - Historical data tracking
   - Point-in-time analysis
   - Automatic history maintenance

5. Stretch Database
   - Transparent data archiving to Azure
   - Automatic data movement
   - Seamless query execution
   - Reduced on-premises storage

### Performance Improvements
- Enhanced In-Memory OLTP
- Multiple TempDB files
- Query execution improvements
- Live Query Statistics
- Native compilation enhancements

## Migration Planning

### Short-term Migration Plan (1-3 months)
1. Assessment Phase
   - Run Data Migration Assistant
   - Identify compatibility issues
   - Performance baseline creation
   - Application dependencies review

2. Testing Phase
   - Development environment upgrade
   - Application testing
   - Performance testing
   - Backup and restore validation

3. Migration Phase
   - Production backup strategy
   - Downtime window planning
   - Rollback plan
   - Post-migration validation

### Medium-term Migration Plan (3-6 months)
1. Preparation Phase
   - Hardware/infrastructure updates
   - New feature implementation planning
   - Staff training
   - Documentation updates

2. Feature Implementation
   - Query Store enablement
   - Always Encrypted deployment
   - Temporal Tables migration
   - JSON implementation

3. Application Updates
   - Code refactoring
   - Performance optimization
   - New feature adoption
   - Testing and validation

### Long-term Migration Plan (6-12 months)
1. Architecture Updates
   - High Availability redesign
   - Disaster Recovery planning
   - Security enhancement
   - Monitoring solution updates

2. Feature Optimization
   - In-Memory OLTP adoption
   - Columnstore implementation
   - Stretch Database evaluation
   - Performance tuning

## Compatibility Bridge Solutions

### Feature Compatibility Issues
1. Database Compatibility Level
   - Use lower compatibility level temporarily
   - Gradual upgrade approach
   - Query plan baseline creation
   - Performance monitoring

2. Deprecated Features
   - Database Mail alternatives
   - XML indexing changes
   - Full-text search updates
   - Replication modifications

### Proxy Solutions

#### DBCC TRACEON Flags
```sql
-- Enable specific trace flags for compatibility
DBCC TRACEON (4199, -1)
```

#### Compatibility Views
```sql
-- Create compatibility views for legacy queries
CREATE VIEW [Legacy].[CustomerView]
WITH SCHEMABINDING
AS
SELECT /* legacy column mappings */
```

#### Linked Servers
- Configure linked servers for cross-version queries
- Set up distributed transactions
- Implement security mapping
- Monitor performance impact

## Migration Checklists

### Pre-Migration
- [ ] Complete system assessment
- [ ] Verify hardware requirements
- [ ] Check application compatibility
- [ ] Create backup strategy
- [ ] Document current configuration

### During Migration
- [ ] Perform full backup
- [ ] Stop application access
- [ ] Upgrade SQL Server instance
- [ ] Restore databases
- [ ] Update statistics

### Post-Migration
- [ ] Verify database integrity
- [ ] Check application functionality
- [ ] Monitor performance
- [ ] Enable new features
- [ ] Update maintenance plans

## Failover Scenarios

### High Availability Options
1. Always On Availability Groups
   - Mixed version support
   - Rolling upgrades
   - Read-scale deployment
   - Automatic failover

2. Database Mirroring
   - Legacy compatibility
   - Simple failover mechanism
   - Minimal downtime
   - Automatic client redirect

3. Log Shipping
   - Cross-version support
   - Manual failover process
   - Delayed data protection
   - Multiple secondary support

## Reference Documentation
- [Upgrade to SQL Server 2016](../docs/database-engine/install-windows/upgrade-sql-server.md)
- [Migration Best Practices](../docs/sql-server/best-practices/best-practices-for-upgrading-database-engine.md)
- [Backward Compatibility](../docs/database-engine/discontinued-database-engine-functionality-in-sql-server.md)
- [High Availability Solutions](../docs/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server.md)

## Infrastructure Planning

### Virtualization Environment Setup

1. Storage Configuration
   ```powershell
   # PowerShell commands for storage setup
   $dataPath = "E:\SQLData"
   $logPath = "F:\SQLLogs"
   $tempPath = "T:\SQLTemp"
   
   # Create directories with proper permissions
   New-Item -ItemType Directory -Path $dataPath
   New-Item -ItemType Directory -Path $logPath
   New-Item -ItemType Directory -Path $tempPath
   
   # Set permissions
   $acl = Get-Acl $dataPath
   $rule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT Service\MSSQLSERVER","FullControl","ContainerInherit,ObjectInherit","None","Allow")
   $acl.SetAccessRule($rule)
   Set-Acl $dataPath $acl
   ```

2. Storage Layout
   - Data files: Distributed across multiple volumes
   - Log files: Dedicated high-performance SSDs
   - TempDB: Local SSDs for optimal performance
   - BLOB storage: Separate volume with sequential I/O optimization

3. Memory Configuration
   ```sql
   -- Configure memory limits
   EXEC sp_configure 'show advanced options', 1;
   RECONFIGURE;
   EXEC sp_configure 'max server memory (MB)', 262144; -- 256GB
   EXEC sp_configure 'min server memory (MB)', 131072; -- 128GB
   RECONFIGURE;
   ```

### Database Partitioning Strategy

1. Filegroup Creation
   ```sql
   -- Create filegroups for partitioning
   ALTER DATABASE [LargeDB] ADD FILEGROUP [FG_Archive_2020]
   ALTER DATABASE [LargeDB] ADD FILEGROUP [FG_Archive_2021]
   ALTER DATABASE [LargeDB] ADD FILEGROUP [FG_Archive_2022]
   ALTER DATABASE [LargeDB] ADD FILEGROUP [FG_Current]

   -- Add files to filegroups
   ALTER DATABASE [LargeDB] ADD FILE 
   (NAME = N'Archive2020', FILENAME = 'E:\SQLData\Archive2020.ndf')
   TO FILEGROUP [FG_Archive_2020]
   ```

2. Partition Function
   ```sql
   -- Create partition function by year
   CREATE PARTITION FUNCTION [PF_ByYear](datetime)
   AS RANGE RIGHT FOR VALUES 
   ('2020-01-01', '2021-01-01', '2022-01-01')

   -- Create partition scheme
   CREATE PARTITION SCHEME [PS_ByYear]
   AS PARTITION [PF_ByYear]
   TO ([FG_Archive_2020], [FG_Archive_2021], 
       [FG_Archive_2022], [FG_Current])
   ```

### BLOB Management Strategy

1. FileStream Configuration
   ```sql
   -- Enable FileStream
   EXEC sp_configure 'filestream access level', 2
   RECONFIGURE

   -- Add FileStream filegroup
   ALTER DATABASE [LargeDB]
   ADD FILEGROUP [FileStreamGroup] CONTAINS FILESTREAM

   -- Add FileStream container
   ALTER DATABASE [LargeDB] 
   ADD FILE (NAME = 'FSData', FILENAME = 'E:\FileStream')
   TO FILEGROUP [FileStreamGroup]
   ```

2. Table Partitioning
   ```sql
   -- Create partitioned table for BLOB data
   CREATE TABLE dbo.Documents
   (
       DocId uniqueidentifier ROWGUIDCOL NOT NULL,
       FileData varbinary(max) FILESTREAM NULL,
       CreatedDate datetime NOT NULL,
       CONSTRAINT PK_Documents PRIMARY KEY CLUSTERED (DocId)
   )
   ON PS_ByYear(CreatedDate)
   FILESTREAM_ON [FileStreamGroup]
   ```

### High Availability Setup

1. Always On Configuration
   ```sql
   -- Enable Always On
   Enable-SqlAlwaysOn -Path SQLSERVER:\SQL\Node1
   
   -- Create availability group
   CREATE AVAILABILITY GROUP [AG_Primary]
   FOR DATABASE [LargeDB]
   REPLICA ON 'Node1' WITH 
   (
       ENDPOINT_URL = 'TCP://Node1.domain.com:5022',
       FAILOVER_MODE = AUTOMATIC,
       AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
   ),
   'Node2' WITH
   (
       ENDPOINT_URL = 'TCP://Node2.domain.com:5022',
       FAILOVER_MODE = AUTOMATIC,
       AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
   )
   ```

2. Readable Secondary Setup
   ```sql
   -- Configure readable secondary
   ALTER AVAILABILITY GROUP [AG_Primary]
   MODIFY REPLICA ON 'Node2' WITH 
   (SECONDARY_ROLE(ALLOW_CONNECTIONS = ALL))
   ```

### Performance Testing Environment

1. Workload Capture
   ```sql
   -- Start Extended Events trace
   CREATE EVENT SESSION [WorkloadCapture] ON SERVER 
   ADD EVENT sqlserver.sql_batch_completed,
   ADD EVENT sqlserver.sql_statement_completed
   ADD TARGET package0.event_file
   (SET filename=N'E:\Traces\workload.xel')
   WITH (MAX_DISPATCH_LATENCY = 1 SECONDS)
   GO
   
   ALTER EVENT SESSION [WorkloadCapture] 
   ON SERVER STATE = START
   ```

2. Replay Configuration
   ```sql
   -- Prepare Database for replay
   ALTER DATABASE [LargeDB] 
   SET RECOVERY FULL
   
   -- Create Distributed Replay Controller
   dreplay.exe preprocess -i "E:\Traces\workload.xel" 
                         -o "E:\Traces\workload_replay"
   ```

### Compatibility Mode Management

1. Staged Upgrade Path
   ```sql
   -- Check current compatibility
   SELECT name, compatibility_level 
   FROM sys.databases
   WHERE name = 'LargeDB'
   
   -- Update compatibility level gradually
   ALTER DATABASE [LargeDB] 
   SET COMPATIBILITY_LEVEL = 110 -- SQL 2012
   
   -- Enable Query Store for monitoring
   ALTER DATABASE [LargeDB] 
   SET QUERY_STORE = ON
   ```

2. Legacy Feature Support
   ```sql
   -- Create compatibility view
   CREATE VIEW [Legacy].[CustomerView]
   WITH SCHEMABINDING
   AS
   SELECT 
       c.CustomerId,
       c.Name,
       CAST(c.Data AS VARCHAR(MAX)) AS LegacyData
   FROM dbo.Customers c
   ```

## Parallel Operations Strategy

### Database Sharding Implementation

1. Horizontal Partitioning Schema
   ```sql
   -- Create partition map table
   CREATE TABLE dbo.ShardMap
   (
       ShardId int PRIMARY KEY,
       ShardServer nvarchar(128),
       ShardDatabase nvarchar(128),
       DateRangeStart datetime,
       DateRangeEnd datetime,
       Status tinyint
   )

   -- Define shard routing function
   CREATE FUNCTION dbo.GetShardLocation
   (
       @DateKey datetime
   )
   RETURNS TABLE
   AS RETURN
   (
       SELECT ShardServer, ShardDatabase
       FROM dbo.ShardMap
       WHERE @DateKey BETWEEN DateRangeStart AND DateRangeEnd
   )
   ```

2. Data Distribution
   ```sql
   -- Create distributed partitioned view
   CREATE VIEW dbo.GlobalTransactions
   AS
   SELECT TransactionId, TransactionDate, Amount
   FROM Server1.Sales.dbo.Transactions
   UNION ALL
   SELECT TransactionId, TransactionDate, Amount
   FROM Server2.Archive.dbo.Transactions
   ```

### Parallel Environment Setup

1. Replication Configuration
   ```sql
   -- Configure transactional replication
   sp_configure 'repl agent', 1
   RECONFIGURE

   -- Add publication
   EXEC sp_addpublication 
       @publication = 'LiveSync',
       @description = 'Live synchronization publication',
       @sync_method = 'concurrent',
       @allow_push = 'true',
       @allow_pull = 'true',
       @allow_anonymous = 'false',
       @enabled_for_internet = 'false',
       @snapshot_in_defaultfolder = 'true'

   -- Add articles
   EXEC sp_addarticle 
       @publication = 'LiveSync',
       @article = 'Customers',
       @source_owner = 'dbo',
       @source_object = 'Customers',
       @type = 'logbased',
       @description = 'Customer table article'
   ```

2. Load Balancing
   ```sql
   -- Create linked servers
   EXEC sp_addlinkedserver 
       @server = 'NEWSQL2016',
       @srvproduct = 'SQL Server'

   -- Configure routing
   CREATE FUNCTION dbo.RouteQuery
   (
       @QueryType int,
       @Parameters nvarchar(max)
   )
   RETURNS nvarchar(128)
   AS
   BEGIN
       DECLARE @Server nvarchar(128)
       -- Route read queries to secondary
       IF @QueryType = 1 -- Read
           SET @Server = 'NEWSQL2016'
       ELSE
           SET @Server = 'PRIMARYSQL'
       RETURN @Server
   END
   ```

### Gradual Feature Migration

1. Feature Testing Framework
   ```sql
   -- Create testing harness
   CREATE TABLE dbo.FeatureTests
   (
       FeatureId int PRIMARY KEY,
       FeatureName nvarchar(100),
       OldImplementation nvarchar(max),
       NewImplementation nvarchar(max),
       TestStatus tinyint,
       LastTestDate datetime
   )

   -- Monitor feature usage
   CREATE EVENT SESSION [FeatureUsage] ON SERVER 
   ADD EVENT sqlserver.deprecation_announcement,
   ADD EVENT sqlserver.feature_restriction_event_pre_90
   ADD TARGET package0.event_file
   (SET filename=N'E:\Traces\feature_usage.xel')
   ```

2. Compatibility Functions
   ```sql
   -- Create wrapper functions for deprecated features
   CREATE FUNCTION dbo.LegacyTextPointer
   (
       @Column varbinary(max)
   )
   RETURNS binary(16)
   AS
   BEGIN
       RETURN CAST(CAST(@Column AS varchar(max)) AS binary(16))
   END

   -- Create compatibility stored procedures
   CREATE PROCEDURE dbo.LegacyUpdateText
       @Table nvarchar(128),
       @Column nvarchar(128),
       @Text nvarchar(max)
   AS
   BEGIN
       DECLARE @SQL nvarchar(max)
       SET @SQL = 'UPDATE ' + @Table + 
                  ' SET ' + @Column + ' = @Text'
       EXEC sp_executesql @SQL, 
            N'@Text nvarchar(max)', 
            @Text
   END
   ```

### Application Transition Plan

1. Connection String Management
   ```sql
   -- Create routing table
   CREATE TABLE dbo.AppConnections
   (
       AppId int PRIMARY KEY,
       AppName nvarchar(100),
       OldConnection nvarchar(max),
       NewConnection nvarchar(max),
       MigrationStatus tinyint,
       MigrationDate datetime
   )

   -- Create connection string view
   CREATE VIEW dbo.ActiveConnections
   AS
   SELECT 
       CASE 
           WHEN MigrationStatus = 2 
           THEN NewConnection 
           ELSE OldConnection 
       END AS ConnectionString,
       AppName
   FROM dbo.AppConnections
   ```

2. Traffic Migration
   ```sql
   -- Monitor application connections
   CREATE EVENT SESSION [ConnectionAudit] ON SERVER 
   ADD EVENT sqlserver.login,
   ADD EVENT sqlserver.logout
   ADD TARGET package0.event_file
   (SET filename=N'E:\Traces\connections.xel')

   -- Create traffic routing procedure
   CREATE PROCEDURE dbo.RouteTraffic
       @AppId int,
       @Percentage int
   AS
   BEGIN
       UPDATE dbo.AppConnections
       SET MigrationStatus = 
           CASE WHEN RAND() * 100 < @Percentage 
                THEN 2 ELSE 1 END
       WHERE AppId = @AppId
   END
   ```
