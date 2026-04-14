---
description: "Use when discussing databases, data architecture, Azure SQL, PostgreSQL, Cosmos DB, Redis, Event Hubs, Service Bus, data classification, PII handling, data residency, encryption at rest, CMK, Entra ID authentication for databases, backup strategy, disaster recovery for data, connection pooling, schema migrations, or data lifecycle management."
---
# Contoso Data & Database Standards

Read the full Contoso data standards before making recommendations:

- [Data Standards — CTSO-DATA-001](../../../standards/data.md)

## Key Policies

- Approved: Azure SQL, PostgreSQL Flex, Cosmos DB, Redis Enterprise, Event Hubs, Service Bus
- Entra ID auth mandatory; SQL auth (username/password) prohibited in production
- Applications connect via Managed Identities — no credential-based connection strings
- Data classification: Public, Internal, Confidential (CMK), Restricted (CMK + field-level encryption)
- Canadian data residency for customer data
- Private Endpoints only; no public database access
- Automated backups with GRS; 35-day PITR retention
- Zone-redundant configurations for production
- Connection pooling mandatory
- Schema changes via migration scripts only — no manual DDL in production

When making data recommendations, cite sections as `[CTSO-DATA-001 §N]`.
