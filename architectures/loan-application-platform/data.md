# Data — Contoso Loan Application Platform

## Database Selections

### Primary Data Store: Azure SQL Database (Business Critical)

Azure SQL Database is selected for the core loan application data, including applications, decisions, customer profiles, and audit trails.

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | Business Critical | [CTSO-DATA-001 §8] (no Basic/Standard in prod) |
| Zone redundancy | Enabled (3 zones) | [CTSO-DATA-001 §6] |
| Authentication | Microsoft Entra ID only | [CTSO-DATA-001 §4] |
| Connectivity | Private Endpoint only | [CTSO-DATA-001 §4] |
| Encryption | TDE + CMK (Azure Key Vault) | [CTSO-SEC-001 §1], [CTSO-DATA-001 §2] |
| Backup | GRS with 35-day PITR | [CTSO-DATA-001 §5] |
| LTR | Weekly/Monthly/Yearly for 7-year compliance | [CTSO-DATA-001 §5] |
| Advanced Threat Protection | Enabled | [CTSO-DATA-001 §4] |
| Failover group | Canada Central → Canada East (auto-failover) | [CTSO-DATA-001 §5] |
| Connection pooling | HikariCP-equivalent via Microsoft.Data.SqlClient | [CTSO-DATA-001 §6] |

**Data model ownership by service:**

| Service | Database/Schema | Key Entities |
|---------|----------------|--------------|
| Application Intake | `LoanApplications` | Applications, Applicants, Employment, Financial History |
| Decision Engine | `Underwriting` | Decisions, Rules, Scoring Results, Overrides |
| Advisor Portal BFF | `AdvisorWorkflow` | Review Queue, Assignments, Notes |
| Reporting | Read replica | Aggregated views (read-only) |

Each microservice owns its schema — no cross-service database sharing [CTSO-APP-001 §1].

### Document Store: Azure Cosmos DB (NoSQL API)

Azure Cosmos DB stores document metadata, OCR results, and semi-structured intake data that benefit from schema flexibility.

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| API | NoSQL | [CTSO-DATA-001 §1] |
| Consistency | Session (default), Strong for critical reads | Best practice |
| Partition key | `/applicationId` | Performance optimization |
| Throughput | Autoscale (400 – 4,000 RU/s) | [CTSO-DATA-001 §6] |
| Replication | Canada Central (write) → Canada East (read) | [CTSO-DATA-001 §3] |
| Connectivity | Private Endpoint only | [CTSO-DATA-001 §4] |
| Encryption | CMK via Key Vault | [CTSO-SEC-001 §1] |
| Backup | Continuous backup (7-day PITR) | [CTSO-DATA-001 §5] |

### Cache: Azure Cache for Redis (Enterprise)

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | Enterprise (E10) | [CTSO-DATA-001 §1] (Enterprise for production) |
| Zone redundancy | Enabled | [CTSO-DATA-001 §6] |
| Use cases | Session state, credit score caching, API response caching | Performance |
| Connectivity | Private Endpoint only | [CTSO-DATA-001 §4] |
| TTL policy | 15 min for credit scores, 30 min for session data | Application-specific |

### Messaging: Azure Service Bus (Premium)

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | Premium | [CTSO-DATA-001 §1] (Premium for production) |
| Zone redundancy | Enabled | [CTSO-DATA-001 §6] |
| Connectivity | Private Endpoint only | [CTSO-DATA-001 §4] |

**Queue / Topic Design:**

| Queue/Topic | Type | Publisher | Subscriber(s) | Purpose |
|-------------|------|-----------|---------------|---------|
| `loan-application-submitted` | Topic | Application Intake | ID Verify, Credit Scoring, Document Processing | Fan-out on new application |
| `credit-score-completed` | Queue | Credit Scoring | Decision Engine | Score ready for decisioning |
| `document-processed` | Queue | Document Processing | Decision Engine | OCR results available |
| `decision-made` | Queue | Decision Engine | Notification Service, Advisor BFF | Decision outcome routing |
| `notification-request` | Queue | Multiple | Notification Service | Email/SMS delivery |
| `manual-review-required` | Queue | Decision Engine | Advisor BFF | Routes to advisor queue |

## Data Classification

All data classified as **Confidential** per [CTSO-DATA-001 §2]:

| Data Element | Classification | Encryption | Retention |
|-------------|---------------|------------|-----------|
| Customer PII (name, SSN, address) | Confidential | CMK (Key Vault) | 7 years |
| Financial data (income, debts) | Confidential | CMK (Key Vault) | 7 years |
| Credit scores | Confidential | CMK (Key Vault) | 7 years |
| Uploaded documents (ID scans, tax returns) | Confidential | CMK (Key Vault) | 7 years |
| Loan decisions and rationale | Confidential | CMK (Key Vault) | 7 years |
| Advisor notes | Internal | At rest + in transit | 7 years |
| Aggregated reporting data | Internal | At rest + in transit | 3 years |

### PII Protection

- PII must **never** appear in application logs or telemetry [CTSO-DATA-001 §8]
- Dynamic data masking enabled on Azure SQL for PII columns
- Azure Purview data catalog for data governance and lineage tracking
- Right-to-erasure (PIPEDA) implemented via soft delete → hard purge workflow

## Data Residency

All data resides in **Canada Central** (primary) and **Canada East** (DR) only [CTSO-DATA-001 §3], [CTSO-SEC-001 §7]:

- Azure SQL failover group: Canada Central → Canada East
- Cosmos DB multi-region write: Canada Central, read replica Canada East
- Redis geo-replication within Canadian region pair
- Service Bus geo-disaster recovery: Canada Central → Canada East
- Blob Storage (documents): GRS within Canadian pair
- **No cross-border replication** — all backup and DR within Canada

## Schema Migration Strategy

All schema changes via migration scripts [CTSO-DATA-001 §7]:

- **Tool**: Entity Framework Core Migrations (.NET)
- **Process**: Migration scripts checked into repo → reviewed in PR → applied via CI/CD pipeline
- **No manual DDL** in any environment [CTSO-DATA-001 §7]
- Rollback scripts required for every forward migration

## Data Lifecycle Management

| Stage | Policy | Standard Reference |
|-------|--------|-------------------|
| Active | Hot storage, full access | — |
| Archive (>2 years) | Move to cool/archive tier | [CTSO-DATA-001 §7] |
| Soft delete | 30-day recovery window | [CTSO-DATA-001 §7] |
| Regulatory retention | 7-year minimum | Requirement / PIPEDA |
| Purge | After retention + erasure request | [CTSO-DATA-001 §7] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| Azure SQL as primary OLTP store | CTSO-DATA-001 | §1 |
| Cosmos DB for document store | CTSO-DATA-001 | §1 |
| Redis Enterprise for caching | CTSO-DATA-001 | §1 |
| Service Bus Premium for messaging | CTSO-DATA-001 | §1 |
| Entra ID authentication only | CTSO-DATA-001 | §4 |
| Managed Identity for connections | CTSO-DATA-001 | §4 |
| CMK encryption for Confidential data | CTSO-DATA-001 | §2, CTSO-SEC-001 §1 |
| Private Endpoints for all databases | CTSO-DATA-001 | §4 |
| GRS backups, 35-day PITR | CTSO-DATA-001 | §5 |
| Failover group to Canada East | CTSO-DATA-001 | §5 |
| Zone-redundant configurations | CTSO-DATA-001 | §6 |
| No cross-service DB sharing | CTSO-APP-001 | §1 |
| Schema via migration scripts | CTSO-DATA-001 | §7 |
| No PII in logs | CTSO-DATA-001 | §8 |
| Data residency in Canada | CTSO-DATA-001 | §3 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Database auth | Entra ID only; SQL auth prohibited | Entra ID recommended; SQL auth supported | **Exceeds** | [Azure SQL Entra auth](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview) |
| Encryption | CMK for Confidential data | TDE with service-managed keys default; CMK optional | **Exceeds** | [Azure SQL TDE with CMK](https://learn.microsoft.com/azure/azure-sql/database/transparent-data-encryption-byok-overview) |
| Connectivity | Private Endpoints only | Private Endpoints recommended; public access supported | **Exceeds** | [Azure SQL Private Link](https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview) |
| Backup retention | 35-day PITR + LTR (7 years) | 7-35 day PITR configurable; LTR up to 10 years | **Aligns** | [Azure SQL backup](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview) |
| Cosmos DB partition key | Application-aligned partition key | Recommend high-cardinality partition key | **Aligns** | [Cosmos DB partitioning](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview) |
| Service Bus | Premium tier with zone redundancy | Premium recommended for production; geo-DR supported | **Aligns** | [Service Bus Premium](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-premium-messaging) |
| Data per microservice | No cross-service DB sharing | Each service should manage own data | **Aligns** | [Data considerations for microservices](https://learn.microsoft.com/azure/architecture/microservices/design/data-considerations) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| Cosmos DB RU cost may spike with document volume | Autoscale with budget alerts; review partition strategy quarterly | Monitor |
| 7-year retention generates significant storage cost | Archive tier for data >2 years; estimate storage budget | Estimate needed |
| RPO ≤ 30 min requires more aggressive replication than standard (≤ 1h) | Azure SQL auto-failover group provides near-zero RPO; Cosmos DB continuous backup provides sub-minute RPO | **Achievable** |
| PIPEDA right-to-erasure complexity across multiple stores | Build centralized data purge orchestrator service | Design in GA phase |
