# SQL Server Documentation Repository Overview

This document provides a comprehensive overview of the SQL Server documentation repository structure and contents.

## Repository Structure

### /azure-sql/
Contains documentation for Azure SQL services including:
- Azure SQL Database
- Azure SQL Managed Instance
- Database Watcher and monitoring
- Performance optimization and maintenance
- Security features and Azure AD integration
- Migration guides and tools

Key file types: `.md`, `.yml`

### /data-migration/
Tools and guidance for migrating databases:
- SQL Server migration guides
- Data Migration Assistant docs
- Migration best practices
- Cross-platform migration scenarios

### /docs/
Main documentation content organized by features:

#### Database Engine (/database-engine/)
- Core SQL Server functionality
- Query processing and execution
- Security and authentication
- High availability and disaster recovery
- Backup and restore

#### Integration Services (/integration-services/)
- ETL and data integration
- Package development
- Deployment and administration
- Connectors and adapters

#### Reporting Services (/reporting-services/)
- Report design and development
- Report server administration
- Mobile reports
- Power BI integration

#### Machine Learning (/machine-learning/)
- SQL Server Machine Learning Services
- R and Python integration
- AI model deployment
- Real-time scoring

#### Language and Development
- T-SQL reference (/t-sql/)
- DMX for data mining (/dmx/)
- MDX for OLAP (/mdx/)
- XQuery for XML (/xquery/)

#### Tools and Utilities (/tools/)
- SQL Server Management Studio
- Command line utilities
- PowerShell modules
- Monitoring and profiling tools

### Special File Types

- `.md`: Markdown files containing documentation content
- `.yml`: YAML files for configuration and navigation
- `.codesnippet`: Code samples and examples
- `.include`: Reusable content blocks
- `.svg`, `.png`: Images and diagrams

## Key Features by Section

### Security
- Authentication and authorization
- Encryption and key management
- Auditing and compliance
- Row-level security
- Always Encrypted
- Transparent Data Encryption

### Performance
- Query optimization
- Indexing strategies
- Statistics management
- Resource Governor
- In-memory OLTP
- Columnstore indexes

### High Availability
- Always On Availability Groups
- Failover Cluster Instances
- Database mirroring
- Log shipping
- Replication

### Development
- Modern application development
- JSON support
- Graph database capabilities
- Spatial data
- XML integration
- Programming interfaces

### Cloud and Hybrid
- Azure integration
- Hybrid deployments
- Cloud migration
- Stretch Database
- Managed instances
- Elastic pools

Each section includes detailed documentation, examples, best practices, and troubleshooting guides to help users effectively work with SQL Server and related technologies.
