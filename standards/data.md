# Contoso Data & Database Standards

**Document ID:** CTSO-DATA-001  
**Version:** 2.3  
**Last Updated:** 2025-12-10  
**Owner:** Contoso Data Platform Team  
**Classification:** Internal

## 1. Approved Database Services

| Use Case | Approved Service | Notes |
|---|---|---|
| Relational (OLTP) | Azure SQL Database, Azure Database for PostgreSQL Flexible Server | SQL Server preferred for .NET; PostgreSQL for Java/Python |
| NoSQL / Document | Azure Cosmos DB (NoSQL API) | Use partition key design guidelines |
| Caching | Azure Cache for Redis (Enterprise tier for production) | Basic/Standard tiers for dev/QA only |
| Search | Azure AI Search | Preferred over Elasticsearch |
| Data Warehouse | Azure Synapse Analytics (dedicated SQL pool) | For analytics and reporting |
| Event Streaming | Azure Event Hubs | Kafka protocol supported |
| Message Queuing | Azure Service Bus | Premium tier for production |

- **Unapproved services**: Self-hosted databases on VMs, MongoDB Atlas, AWS RDS, third-party DBaaS.

## 2. Data Classification

| Level | Examples | Encryption | Access Control | Retention |
|---|---|---|---|---|
| **Public** | Marketing content | Standard | Open | As needed |
| **Internal** | Employee docs, internal tools | At rest + in transit | Role-based | 3 years |
| **Confidential** | Customer PII, financial data | CMK (Key Vault) | Named individuals + MFA | 7 years |
| **Restricted** | Payment card data, health records | CMK + field-level encryption | PIM + break-glass | 10 years |

## 3. Data Residency & Sovereignty

- Canadian customer data must be stored in **Canada Central** or **Canada East** regions.
- Cross-border data transfer requires **Privacy Impact Assessment (PIA)** approval.
- Geo-replication must stay within the approved Canadian region pair.
- Backup data is subject to the same residency requirements as primary data.

## 4. Database Security

- All databases must use **Microsoft Entra ID authentication**. SQL authentication (username/password) is **prohibited** in production.
- Applications must connect using **Managed Identities** — no connection strings with embedded credentials.
- **Transparent Data Encryption (TDE)** must be enabled on all Azure SQL databases.
- **Advanced Threat Protection** must be enabled on all database services.
- Database firewall rules must deny public access. Access via **Private Endpoints** only.

## 5. Backup & Recovery

- **Automated backups** with geo-redundant storage (GRS) for production databases.
- Point-in-time restore: **35-day** retention minimum for production.
- Long-term retention (LTR): weekly, monthly, and yearly backups for compliance.
- DR database replicas in **Canada East** with automatic failover groups for Tier-1 applications.
- Backup restoration tests required **quarterly**.

## 6. Performance & Scaling

- Production databases must use **zone-redundant** configurations.
- Auto-scaling must be configured for Cosmos DB (autoscale throughput) and Azure SQL (elastic pools or serverless).
- Read replicas recommended for read-heavy workloads.
- Query Performance Insight and Intelligent Performance must be enabled on Azure SQL.
- Connection pooling is **mandatory** for all applications.

## 7. Data Lifecycle Management

- Implement **soft delete** for all user-facing data (minimum 30-day recovery window).
- Archival policy: data older than 2 years moved to cool/archive storage tier.
- Data purge procedures must comply with right-to-erasure requests (PIPEDA).
- All schema changes must go through **database migration scripts** — no manual DDL in production.

## 8. Prohibited Practices

- Storing PII in application logs or telemetry.
- Using SQL authentication (username/password) for Azure SQL.
- Deploying databases without Private Endpoints.
- Storing connection strings with credentials in appsettings or environment variables.
- Running production databases on Basic or Standard tiers without Architecture Review Board approval.
- Direct production database access without PIM-activated Just-In-Time roles.
