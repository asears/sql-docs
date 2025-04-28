# Microsoft Purview Integration with SQL Server

## Overview
Microsoft Purview provides unified data governance for SQL Server databases, enabling data discovery, classification, and access control. This document summarizes key Purview features and integration points.

## Key Features and References

### Data Discovery & Classification
- [Data Discovery & Classification Overview](/azure/purview/how-to-automatically-label-your-content)
- Automatically scans and classifies sensitive data
- Assigns sensitivity labels to columns
- Integrates with Azure SQL Database and SQL Server

#### Prerequisites
- Microsoft 365 license
- Active Microsoft Purview account
- Data Source Administrator and Data Reader permissions
- Self-hosted integration runtime

### Access Control Policies
- [Provision access by data owner for SQL Server on Azure Arc](/azure/purview/how-to-policies-data-owner-arc-sql-server)
- [Configure access policies in Purview for Azure SQL Database](/azure/purview/how-to-policies-data-owner-azure-sql-db)
- Enforce deny actions on sensitive columns
- Restrict access based on sensitivity labels

### Security Roles
New security roles introduced:
- SQL Performance Monitor
- SQL Security Auditor
- Aligns with principle of least privilege

## Implementation Tips

### Best Practices
1. Register databases in Purview Data Map first
2. Configure scanning schedule for continuous updates
3. Use hierarchical label inheritance
4. Implement column-level encryption for sensitive data
5. Regular audit of access policies

### Common Challenges and Solutions
1. Geo-replica limitations
   - Labels don't automatically flow to replicas
   - Register and scan replicas separately
   - Configure policies for each replica

2. Performance Considerations
   - Schedule scans during off-peak hours
   - Use incremental scanning when possible
   - Monitor policy evaluation impact

3. Integration Runtime Setup
   - Use latest self-hosted integration runtime
   - Configure proper network access
   - Monitor runtime health

## Features by Version

### SQL Server 2022
- Full Purview integration
- Azure Arc enablement
- Enhanced sensitivity labeling
- Access policy enforcement

### Azure SQL Database
- Native Purview integration
- Real-time policy enforcement
- Automated classification
- Continuous compliance monitoring

## Monitoring and Maintenance

### Health Monitoring
- Regular validation of scanning status
- Policy effectiveness review
- Access pattern analysis
- Label distribution reports

### Maintenance Tasks
1. Regular review of classification accuracy
2. Update sensitivity labels as needed
3. Audit policy effectiveness
4. Monitor false positives/negatives
5. Update scanning schedules

## Data Governance Implementation

### Asset Registration Framework

1. Automated Asset Discovery
```sql
-- Create asset tracking
CREATE TABLE dbo.PurviewAssets (
    AssetId uniqueidentifier DEFAULT NEWID(),
    AssetType varchar(50),
    ServerName nvarchar(128),
    DatabaseName sysname,
    SchemaName sysname,
    ObjectName sysname,
    SensitivityLabel varchar(50),
    ClassificationType varchar(50),
    LastScanned datetime2,
    ScanStatus tinyint
);

-- Track scanning status
CREATE PROCEDURE dbo.UpdatePurviewScanStatus
    @AssetId uniqueidentifier,
    @ScanStatus tinyint
AS
BEGIN
    UPDATE dbo.PurviewAssets
    SET 
        LastScanned = GETUTCDATE(),
        ScanStatus = @ScanStatus
    WHERE AssetId = @AssetId;
END;
```

2. Classification Rules
```json
{
    "classificationRules": [
        {
            "name": "Credit Card Detection",
            "pattern": "^(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|6(?:011|5[0-9]{2})[0-9]{12})$",
            "sensitivity": "Confidential",
            "dataType": "Credit Card Number"
        },
        {
            "name": "Email Detection",
            "pattern": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
            "sensitivity": "Internal",
            "dataType": "Email Address"
        }
    ]
}
```

### Metadata Governance

1. Business Glossary Integration
```sql
CREATE TABLE dbo.BusinessTerms (
    TermId int IDENTITY(1,1),
    BusinessTerm nvarchar(100),
    Definition nvarchar(max),
    Domain nvarchar(50),
    Steward nvarchar(100),
    Status tinyint,
    LastModified datetime2
);

CREATE TABLE dbo.TermRelationships (
    RelationshipId int IDENTITY(1,1),
    SourceTermId int,
    TargetTermId int,
    RelationType varchar(20),
    CONSTRAINT FK_TermRel_Source 
        FOREIGN KEY (SourceTermId) 
        REFERENCES dbo.BusinessTerms(TermId),
    CONSTRAINT FK_TermRel_Target 
        FOREIGN KEY (TargetTermId) 
        REFERENCES dbo.BusinessTerms(TermId)
);
```

2. Data Dictionary Management
```sql
-- Create data dictionary tables
CREATE TABLE dbo.DataDictionary (
    DictionaryId int IDENTITY(1,1),
    TableName sysname,
    ColumnName sysname,
    BusinessDefinition nvarchar(max),
    DataType varchar(50),
    IsNullable bit,
    BusinessTermId int,
    SensitivityLevel varchar(20),
    RetentionPeriod int,
    LastReview datetime2,
    CONSTRAINT FK_Dictionary_Term 
        FOREIGN KEY (BusinessTermId) 
        REFERENCES dbo.BusinessTerms(TermId)
);

-- Track lineage
CREATE TABLE dbo.DataLineage (
    LineageId int IDENTITY(1,1),
    SourceObject nvarchar(128),
    TargetObject nvarchar(128),
    TransformationLogic nvarchar(max),
    LastModified datetime2,
    ModifiedBy nvarchar(128)
);
```

### Access Control Implementation

1. Role-Based Access Control
```sql
-- Create RBAC framework
CREATE TABLE dbo.PurviewRoles (
    RoleId int IDENTITY(1,1),
    RoleName nvarchar(50),
    Description nvarchar(max),
    Permissions nvarchar(max)
);

CREATE TABLE dbo.RoleAssignments (
    AssignmentId int IDENTITY(1,1),
    RoleId int,
    PrincipalName nvarchar(128),
    AssignmentType varchar(20),
    StartDate datetime2,
    EndDate datetime2,
    ApprovedBy nvarchar(128),
    CONSTRAINT FK_RoleAssign_Role 
        FOREIGN KEY (RoleId) 
        REFERENCES dbo.PurviewRoles(RoleId)
);
```

2. Data Access Policies
```sql
-- Create policy framework
CREATE TABLE dbo.AccessPolicies (
    PolicyId int IDENTITY(1,1),
    PolicyName nvarchar(100),
    Description nvarchar(max),
    PolicyType varchar(50),
    PolicyDefinition nvarchar(max),
    IsEnabled bit,
    LastModified datetime2
);

-- Track policy enforcement
CREATE TABLE dbo.PolicyEnforcement (
    EnforcementId bigint IDENTITY(1,1),
    PolicyId int,
    PrincipalName nvarchar(128),
    ResourceName nvarchar(128),
    AccessType varchar(20),
    IsAllowed bit,
    EnforcementTime datetime2,
    CONSTRAINT FK_PolicyEnforce_Policy 
        FOREIGN KEY (PolicyId) 
        REFERENCES dbo.AccessPolicies(PolicyId)
);
```

## Advisory Framework

### Data Quality Monitoring

1. Quality Metrics Collection
```sql
CREATE TABLE dbo.DataQualityMetrics (
    MetricId bigint IDENTITY(1,1),
    TableName sysname,
    ColumnName sysname,
    MetricType varchar(50),
    MetricValue decimal(18,2),
    SampleSize int,
    CollectionDate datetime2
);

CREATE PROCEDURE dbo.CollectQualityMetrics
    @TableName sysname,
    @ColumnName sysname
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @Params nvarchar(max);
    
    -- Calculate completeness
    SET @SQL = '
    INSERT INTO dbo.DataQualityMetrics
    SELECT 
        @TableName,
        @ColumnName,
        ''Completeness'',
        (COUNT(*) - COUNT(CASE WHEN ' + 
        QUOTENAME(@ColumnName) + 
        ' IS NULL THEN 1 END)) * 100.0 / COUNT(*),
        COUNT(*),
        GETUTCDATE()
    FROM ' + QUOTENAME(@TableName);
    
    SET @Params = '@TableName sysname, @ColumnName sysname';
    
    EXEC sp_executesql @SQL, @Params, 
        @TableName, @ColumnName;
        
    -- Calculate uniqueness
    SET @SQL = '
    INSERT INTO dbo.DataQualityMetrics
    SELECT 
        @TableName,
        @ColumnName,
        ''Uniqueness'',
        COUNT(DISTINCT ' + QUOTENAME(@ColumnName) + 
        ') * 100.0 / COUNT(*),
        COUNT(*),
        GETUTCDATE()
    FROM ' + QUOTENAME(@TableName);
    
    EXEC sp_executesql @SQL, @Params, 
        @TableName, @ColumnName;
END;
```

2. Quality Rules Engine
```sql
CREATE TABLE dbo.DataQualityRules (
    RuleId int IDENTITY(1,1),
    RuleName nvarchar(100),
    TableName sysname,
    ColumnName sysname,
    RuleDefinition nvarchar(max),
    ThresholdValue decimal(18,2),
    Severity tinyint,
    IsEnabled bit
);

CREATE PROCEDURE dbo.EvaluateQualityRules
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    SELECT @SQL = STRING_AGG(
        'INSERT INTO dbo.DataQualityMetrics
        SELECT 
            ''' + TableName + ''',
            ''' + ColumnName + ''',
            ''' + RuleName + ''',
            ' + RuleDefinition + ',
            COUNT(*),
            GETUTCDATE()
        FROM ' + QUOTENAME(TableName),
        '; '
    )
    FROM dbo.DataQualityRules
    WHERE IsEnabled = 1;
    
    EXEC sp_executesql @SQL;
END;
```

### Compliance Monitoring

1. Regulatory Compliance Tracking
```sql
CREATE TABLE dbo.ComplianceRequirements (
    RequirementId int IDENTITY(1,1),
    RegulationType varchar(50),
    Requirement nvarchar(max),
    ImplementationStatus tinyint,
    LastAssessment datetime2,
    NextAssessment datetime2
);

CREATE TABLE dbo.ComplianceAudits (
    AuditId bigint IDENTITY(1,1),
    RequirementId int,
    AuditResult varchar(20),
    Findings nvarchar(max),
    AuditorName nvarchar(128),
    AuditDate datetime2,
    CONSTRAINT FK_Audit_Requirement 
        FOREIGN KEY (RequirementId) 
        REFERENCES dbo.ComplianceRequirements(RequirementId)
);
```

2. Audit Log Management
```sql
CREATE TABLE dbo.AuditLogs (
    LogId bigint IDENTITY(1,1),
    EventType varchar(50),
    ObjectName nvarchar(128),
    PrincipalName nvarchar(128),
    AccessType varchar(20),
    AccessResult varchar(20),
    EventTime datetime2,
    ClientApp nvarchar(128),
    ClientIP varchar(45)
);

CREATE PROCEDURE dbo.ArchiveAuditLogs
    @RetentionDays int = 90
AS
BEGIN
    -- Archive old audit logs
    INSERT INTO dbo.AuditLogsArchive
    SELECT *
    FROM dbo.AuditLogs
    WHERE EventTime < DATEADD(DAY, -@RetentionDays, GETUTCDATE());
    
    -- Delete archived logs
    DELETE FROM dbo.AuditLogs
    WHERE EventTime < DATEADD(DAY, -@RetentionDays, GETUTCDATE());
END;
```

### Data Privacy Management

1. Privacy Impact Assessment
```sql
CREATE TABLE dbo.PrivacyAssessments (
    AssessmentId int IDENTITY(1,1),
    ObjectName nvarchar(128),
    DataCategory varchar(50),
    PrivacyRisk varchar(20),
    MitigationControls nvarchar(max),
    ReviewDate datetime2,
    NextReviewDate datetime2
);

CREATE TABLE dbo.PrivacyControls (
    ControlId int IDENTITY(1,1),
    ControlName nvarchar(100),
    ControlType varchar(50),
    Implementation nvarchar(max),
    Effectiveness varchar(20),
    LastTested datetime2
);
```

2. Consent Management
```sql
CREATE TABLE dbo.ConsentRecords (
    ConsentId bigint IDENTITY(1,1),
    SubjectId nvarchar(128),
    ConsentType varchar(50),
    ConsentGiven bit,
    ConsentDate datetime2,
    ExpiryDate datetime2,
    ProofOfConsent nvarchar(max)
);

CREATE PROCEDURE dbo.ValidateConsent
    @SubjectId nvarchar(128),
    @ConsentType varchar(50)
AS
BEGIN
    SELECT 
        ConsentGiven,
        ConsentDate,
        ExpiryDate
    FROM dbo.ConsentRecords
    WHERE SubjectId = @SubjectId
    AND ConsentType = @ConsentType
    AND ExpiryDate > GETUTCDATE()
    AND ConsentGiven = 1;
END;
```

This implementation framework provides a comprehensive approach to data governance, metadata management, and compliance monitoring within Purview.

## References
- [Microsoft Purview account creation](/purview/create-microsoft-purview-portal)
- [Access control in Microsoft Purview governance portal](/purview/catalog-permissions)
- [Security Overview in SQL Server with Azure Arc](../sql-server/azure-arc/security-overview.md)
- [Data Discovery & Classification](../azure-sql/database/data-discovery-and-classification-overview.md)
