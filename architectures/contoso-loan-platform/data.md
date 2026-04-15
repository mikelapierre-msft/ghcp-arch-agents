# Data Design: Contoso Loan Application Platform

## Data Classification

All customer data in this platform is classified as **Confidential** (PII + financial data) per [CTSO-DATA-001 §2]. This mandates:
- **Customer-Managed Keys (CMK)** via Azure Key Vault for encryption at rest
- **Named individual + MFA** access control
- **7-year** data retention (regulatory requirement under PIPEDA / OSFI)

## Database Selection

### Primary Databases

| Database | Service | Use Case | Justification |
|---|---|---|---|
| **Azure SQL Database** | Business Critical tier, zone-redundant | Loan applications, underwriting decisions, advisor actions, audit trail | Primary OLTP for .NET workloads per [CTSO-DATA-001 §1]; relational integrity for financial transactions |
| **Azure Cosmos DB** | NoSQL API, autoscale | Application state machine, document metadata, KYC results cache | Low-latency reads for status lookups; flexible schema for varying document types per [CTSO-DATA-001 §1] |
| **Azure Cache for Redis** | Enterprise tier | Session cache, API response cache, rate limiting state | Production requires Enterprise tier per [CTSO-DATA-001 §1]; sub-millisecond latency for high-throughput reads |
| **Azure Blob Storage** | General Purpose v2 + CMK | Uploaded documents (pay stubs, tax returns, ID scans) | Immutable storage for regulatory retention; lifecycle policies for archival |

### Messaging & Streaming

| Service | Use Case | Tier | Justification |
|---|---|---|---|
| **Azure Service Bus** | Async service-to-service messaging (loan pipeline orchestration) | Premium | Production requires Premium per [CTSO-DATA-001 §1]; supports Private Endpoints, large messages |
| **Azure Event Hubs** | Event streaming for reporting, audit events, analytics | Standard | Real-time event ingestion for reporting dashboard; Kafka protocol compatibility |

### Search & Analytics

| Service | Use Case | Justification |
|---|---|---|
| **Azure AI Search** | Loan application search for advisors | Preferred over Elasticsearch per [CTSO-DATA-001 §1] |
| **Azure Synapse Analytics** | Long-term analytics and regulatory reporting | For 7-year data analytics requirements per [CTSO-DATA-001 §1] |

## Data Architecture

### Service-to-Database Mapping (No Shared Databases)

Per [CTSO-APP-001 §1], each microservice owns its data store. Direct database sharing between services is **prohibited**.

| Service | Primary Database | Data Owned |
|---|---|---|
| Loan Intake Service | Azure SQL (LoanApplications DB) | Applications, applicant profiles |
| Identity Verification Service | Azure SQL (IdentityVerification DB) | KYC results, verification status |
| Credit Scoring Service | Azure SQL (CreditScoring DB) | Scores, model versions, decisions |
| Document Processing Service | Azure Cosmos DB + Blob Storage | Document metadata, extracted data, raw files |
| Decision Engine Service | Azure SQL (Decisions DB) | Rules, decision outcomes, overrides |
| Notification Service | Azure Cosmos DB | Notification history, delivery status |
| Reporting Service | Azure Synapse (read replicas) | Aggregated analytics |
| Advisor Portal BFF | N/A (queries via service APIs) | None (aggregates from other services) |

### Data Flow — Loan Application Pipeline

```
Customer → Loan Intake Service → [Service Bus: loan-submitted]
                                        │
    ┌───────────────────────────────────┤
    ▼                                   ▼
Identity Verification       Document Processing
    │                                   │
    └──► [Service Bus:               [Service Bus:
         id-verified]               docs-processed]
              │                         │
              ▼                         ▼
         Credit Scoring ◄──────────────┘
              │
              ▼
    [Service Bus: score-completed]
              │
              ▼
       Decision Engine
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Approved   Declined   Referred
    │         │         │
    ▼         ▼         ▼
[Service Bus: decision-made]
              │
              ▼
    Notification Service ──► Email/SMS
              │
              ▼
    [Event Hubs: loan-events] ──► Reporting Service
```

## Database Security

### Authentication
- All databases use **Microsoft Entra ID authentication** exclusively [CTSO-DATA-001 §4]
- SQL authentication (username/password) is **prohibited** in production [CTSO-DATA-001 §4]
- Applications connect using **Managed Identities** [CTSO-DATA-001 §4]

### Encryption
- **Transparent Data Encryption (TDE)** with CMK on all Azure SQL databases [CTSO-DATA-001 §4], [CTSO-SEC-001 §1]
- **Always Encrypted** for highly sensitive columns (SSN, SIN, financial account numbers) with secure enclaves
- CMK stored in Azure Key Vault with 90-day rotation [CTSO-SEC-001 §2]
- Azure Cosmos DB CMK encryption enabled

### Network Access
- All databases accessed via **Private Endpoints** only [CTSO-DATA-001 §4], [CTSO-SEC-001 §3]
- Public access disabled on all database services
- **Advanced Threat Protection** enabled on all database services [CTSO-DATA-001 §4]

## Backup & Recovery

| Database | Backup Strategy | PITR Retention | LTR | DR Replica |
|---|---|---|---|---|
| Azure SQL | Automated + GRS | 35 days | Weekly/Monthly/Yearly | Auto-failover group to Canada East |
| Cosmos DB | Continuous backup | 30 days | N/A (continuous) | Multi-region write (Canada East) |
| Redis | AOF persistence + snapshot | N/A | Daily export to Blob | Geo-replication to Canada East |
| Blob Storage | GRS | N/A | Lifecycle policies | GRS replicates to Canada East |

Per [CTSO-DATA-001 §5]:
- **Geo-redundant storage (GRS)** for all production databases
- **35-day** point-in-time restore retention minimum
- **Long-term retention** for Azure SQL (weekly, monthly, yearly) for 7-year regulatory compliance
- **Quarterly backup restoration tests** required
- DR replicas in **Canada East** with automatic failover groups

## Data Residency

- All customer data stored in **Canada Central** (primary) and **Canada East** (DR) only [CTSO-DATA-001 §3]
- Geo-replication stays within the approved Canadian region pair [CTSO-DATA-001 §3]
- Backup data subject to same residency requirements [CTSO-DATA-001 §3]
- No cross-border data transfer without Privacy Impact Assessment approval [CTSO-DATA-001 §3]

## Performance & Scaling

- Azure SQL: **Zone-redundant** Business Critical tier with auto-scaling (serverless or elastic pools) [CTSO-DATA-001 §6]
- Cosmos DB: **Autoscale throughput** with partition key design per workload patterns [CTSO-DATA-001 §6]
- Redis: Enterprise tier with zone redundancy [CTSO-DATA-001 §6]
- **Connection pooling mandatory** for all database connections [CTSO-DATA-001 §6]
- **Read replicas** for Azure SQL to offload reporting queries [CTSO-DATA-001 §6]
- Query Performance Insight and Intelligent Performance enabled on Azure SQL [CTSO-DATA-001 §6]

## Data Lifecycle Management

- **Soft delete** on all user-facing data with 30-day recovery window [CTSO-DATA-001 §7]
- Data older than 2 years archived to cool/archive Blob Storage tier [CTSO-DATA-001 §7]
- Right-to-erasure (PIPEDA) procedures implemented for data purge requests [CTSO-DATA-001 §7]
- All schema changes via **database migration scripts** (Entity Framework Migrations for .NET) [CTSO-DATA-001 §7]
- No manual DDL in production [CTSO-DATA-001 §7]

## PII Handling

- PII must **never** be stored in application logs or telemetry [CTSO-DATA-001 §8]
- Data masking applied in non-production environments
- Dynamic Data Masking enabled on Azure SQL for advisor read access to sensitive fields
- Data Discovery and Classification enabled on Azure SQL for automated PII identification

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| Azure SQL for primary OLTP | CTSO-DATA-001 | §1 |
| Cosmos DB for document metadata | CTSO-DATA-001 | §1 |
| Redis Enterprise for caching | CTSO-DATA-001 | §1 |
| Service Bus Premium for messaging | CTSO-DATA-001 | §1 |
| Confidential classification with CMK | CTSO-DATA-001 | §2 |
| Canada Central/East residency | CTSO-DATA-001 | §3 |
| Entra ID authentication only | CTSO-DATA-001 | §4 |
| Managed Identity connections | CTSO-DATA-001 | §4 |
| TDE with CMK | CTSO-DATA-001 §4, CTSO-SEC-001 | §1 |
| Private Endpoints for all databases | CTSO-DATA-001 §4, CTSO-SEC-001 | §3 |
| GRS backups with 35-day PITR | CTSO-DATA-001 | §5 |
| Auto-failover groups to Canada East | CTSO-DATA-001 | §5 |
| Zone-redundant production databases | CTSO-DATA-001 | §6 |
| Connection pooling mandatory | CTSO-DATA-001 | §6 |
| Soft delete and archival policies | CTSO-DATA-001 | §7 |
| Schema migrations via scripts only | CTSO-DATA-001 | §7 |
| No PII in logs | CTSO-DATA-001 | §8 |
| No shared databases between services | CTSO-APP-001 | §1 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| Database authentication | Entra ID only; SQL auth prohibited | Entra ID authentication recommended; disable SQL auth | **Aligns** | [Secure Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/secure-database?view=azuresql#identity-management) |
| Private Endpoints | Mandatory for all databases | Recommended; disable public access when using PE | **Aligns** | [Azure SQL Private Link](https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview) |
| CMK encryption | Required for Confidential data | TDE with CMK recommended for compliance workloads | **Aligns** | [Azure SQL Security Best Practices](https://learn.microsoft.com/azure/well-architected/service-guides/azure-sql-database#security) |
| Always Encrypted | Used for SSN/SIN fields | Recommended with secure enclaves for sensitive data | **Aligns** | [Always Encrypted with Secure Enclaves](https://learn.microsoft.com/azure/azure-sql/database/always-encrypted-enclaves-getting-started) |
| Managed Identities | Required for all connections | Recommended to eliminate credentials in code | **Aligns** | [Managed Identities for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity) |
| Database per service | Prohibited to share databases | Recommended in microservices to avoid coupling | **Aligns** | [AKS Microservices Data Considerations](https://learn.microsoft.com/azure/architecture/microservices/design/data-considerations) |
| Backup retention | 35-day PITR + LTR | 35 days available on Business Critical; LTR for compliance | **Aligns** | [Azure SQL Backup](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview) |
| Advanced Threat Protection | Required on all databases | Recommended; integrated with Defender for Cloud | **Aligns** | [SQL Advanced Threat Protection](https://learn.microsoft.com/azure/azure-sql/database/threat-detection-configure) |
| Connection pooling | Mandatory | Best practice for performance | **Aligns** | [Azure SQL Best Practices](https://learn.microsoft.com/azure/well-architected/service-guides/azure-sql-database) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | 7-year retention requires careful LTR backup cost planning | Medium | Configure LTR policies early; estimate storage costs for 7 years |
| 2 | Cross-service data queries require API composition vs. direct DB access | Low | Reporting Service aggregates via Event Hubs; Synapse for complex analytics |
| 3 | RPO ≤ 30 min is stricter than Contoso standard (≤ 1 hour) | Medium | Azure SQL auto-failover groups support near-zero RPO; Cosmos DB continuous backup meets this |
| 4 | Document storage sizing for OCR documents over 7 years | Medium | Implement lifecycle policies; archive to cool/archive tier after 2 years |
| 5 | Always Encrypted with secure enclaves adds latency to query processing | Low | Profile query performance during MVP; apply only to most sensitive columns |
