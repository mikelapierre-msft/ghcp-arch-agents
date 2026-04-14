# ADR-001: Database Platform Selection for Loan Application Platform

## Status

Accepted

## Date

2026-03-15

## Context

The Contoso Loan Application Platform requires a primary relational database to store loan applications, customer profiles, and decisioning results. The data is classified as **Confidential** (PII + financial records) under Contoso data classification policy [CTSO-DATA-001 §2].

Key requirements:
- ACID transactions for loan application processing
- Encrypted at rest with **customer-managed keys** (CMK) per [CTSO-SEC-001 §1]
- Entra ID authentication — SQL auth prohibited per [CTSO-DATA-001 §4]
- Zone-redundant deployment in Canada Central per [CTSO-INFRA-001 §6]
- Geo-replication to Canada East per [CTSO-DATA-001 §5]
- Private Endpoint access only — no public endpoints per [CTSO-SEC-001 §3] and [CTSO-NET-001 §6]
- 35-day point-in-time restore per [CTSO-DATA-001 §5]
- Connection pooling mandatory per [CTSO-DATA-001 §6]
- Team expertise: strong .NET and SQL Server experience

## Decision Drivers

- Contoso Data Standards compliance [CTSO-DATA-001]
- Data residency in Canada [CTSO-DATA-001 §3]
- Team expertise and time-to-market
- Cost within $15K/month budget
- RPO ≤ 30 min, RTO ≤ 2 hours

## Considered Options

1. **Azure SQL Database** (Business Critical tier)
2. **Azure Database for PostgreSQL Flexible Server** (Memory Optimized tier)
3. **Azure Cosmos DB** (NoSQL API with relational modeling)

## Decision

**Option 1: Azure SQL Database (Business Critical tier)** was selected.

## Options Analysis

### Option 1: Azure SQL Database (Business Critical)

- **Pros**:
  - Strong team expertise with SQL Server/.NET
  - Native Entra ID authentication with managed identity support
  - Built-in zone redundancy in Business Critical tier
  - Auto-failover groups for geo-replication (Canada Central → Canada East)
  - CMK via Azure Key Vault integration
  - Advanced Threat Protection and auditing built-in
  - Excellent .NET SDK and Entity Framework Core support
- **Cons**:
  - Higher cost than General Purpose tier
  - Vendor lock-in to Microsoft ecosystem
- **Contoso Compliance**: ✅ Fully compliant with CTSO-DATA-001 and CTSO-SEC-001
- **Industry Alignment**: Aligns with Microsoft recommended approach for transactional workloads. See [Azure SQL Database documentation](https://learn.microsoft.com/azure/azure-sql/database/sql-database-paas-overview).
- **Cost Estimate**: ~$1,500/month (Business Critical, 4 vCores, zone-redundant, with failover group)

### Option 2: Azure Database for PostgreSQL Flexible Server

- **Pros**:
  - Open-source, avoids vendor lock-in
  - Strong Azure integration and Entra ID support
  - Zone-redundant HA and geo-replication available
  - Lower cost than SQL Database Business Critical
- **Cons**:
  - Limited team expertise with PostgreSQL
  - .NET + PostgreSQL integration less mature than EF Core + SQL Server
  - Migration risk for existing stored procedures and T-SQL patterns
- **Contoso Compliance**: ✅ Compliant — listed as approved in CTSO-DATA-001 §1 ("PostgreSQL for Java/Python" note suggests SQL Server preferred for .NET)
- **Industry Alignment**: Microsoft recommends PostgreSQL for Java/Python workloads. See [Azure PostgreSQL documentation](https://learn.microsoft.com/azure/postgresql/flexible-server/overview).
- **Cost Estimate**: ~$1,100/month (Memory Optimized, 4 vCores, zone-redundant)

### Option 3: Azure Cosmos DB (NoSQL API)

- **Pros**:
  - Global distribution with multi-region writes
  - Guaranteed single-digit millisecond latency
  - Automatic scaling
- **Cons**:
  - Not a relational database — poor fit for ACID transaction requirements
  - Complex data modeling for relational financial data
  - Team has no Cosmos DB experience
  - Higher cost for equivalent workload with strong consistency
- **Contoso Compliance**: ⚠️ Partially compliant — approved service but not recommended for OLTP relational workloads [CTSO-DATA-001 §1]
- **Industry Alignment**: Microsoft recommends Cosmos DB for globally distributed, schema-flexible workloads — not for transactional OLTP. See [Cosmos DB use cases](https://learn.microsoft.com/azure/cosmos-db/use-cases).
- **Cost Estimate**: ~$2,200/month (autoscale, strong consistency, single-region)

## Consequences

### Positive
- Leverages team's existing SQL Server expertise, reducing development risk
- Full compliance with Contoso security and data standards out of the box
- Built-in HA and DR with auto-failover groups meeting RPO/RTO requirements
- Clear migration path from existing Contoso SQL Server workloads

### Negative
- Higher monthly cost compared to PostgreSQL (~$400/month premium)
- Tighter coupling to Microsoft ecosystem
- Business Critical tier may be over-provisioned initially (can scale down if needed)

### Neutral
- Entity Framework Core migrations align with Contoso requirement for migration scripts [CTSO-DATA-001 §7]
- Connection pooling handled by SqlClient with built-in support [CTSO-DATA-001 §6]

## Contoso Standards Compliance

| Standard | Section | Status | Notes |
|----------|---------|--------|-------|
| CTSO-DATA-001 | §1 Approved Services | ✅ | Azure SQL Database is an approved service |
| CTSO-DATA-001 | §2 Data Classification | ✅ | Confidential — CMK via Key Vault |
| CTSO-DATA-001 | §3 Data Residency | ✅ | Canada Central primary, Canada East DR |
| CTSO-DATA-001 | §4 Database Security | ✅ | Entra ID auth via managed identity |
| CTSO-DATA-001 | §5 Backup & Recovery | ✅ | 35-day PITR, GRS, auto-failover group |
| CTSO-DATA-001 | §6 Performance | ✅ | Zone-redundant, connection pooling |
| CTSO-SEC-001 | §1 Encryption | ✅ | TDE + CMK via Key Vault |
| CTSO-SEC-001 | §3 Network Security | ✅ | Private Endpoint access only |
| CTSO-NET-001 | §6 Private Connectivity | ✅ | Private Endpoint with DNS integration |
| CTSO-IAM-001 | §5 Service Identities | ✅ | System-assigned managed identity |

## Industry Best Practices References

| Source | Recommendation | Alignment | URL |
|--------|---------------|-----------|-----|
| Azure WAF - Reliability | Use zone-redundant database deployments | ✅ Aligns | https://learn.microsoft.com/azure/well-architected/reliability/redundancy |
| Azure SQL Best Practices | Use Entra ID authentication, enable ATP | ✅ Aligns | https://learn.microsoft.com/azure/azure-sql/database/security-best-practice |
| Azure SQL Security Baseline | Use private endpoints, disable public access | ✅ Aligns (Contoso requires) | https://learn.microsoft.com/security/benchmark/azure/baselines/azure-sql-database-security-baseline |
| NIST 800-53 | Encrypt data at rest with AES-256 | ✅ Aligns (Contoso exceeds with CMK) | https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final |

## Related Decisions

- ADR-002: Application Compute Platform Selection (pending)
- ADR-003: API Gateway and Ingress Strategy (pending)
