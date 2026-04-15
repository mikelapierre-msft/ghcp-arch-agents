# Resiliency Design: Contoso Loan Application Platform

## Availability & DR Targets

| Requirement | Target | Contoso Standard | Source |
|---|---|---|---|
| Availability | 99.95% uptime | 99.95% minimum | [CTSO-INFRA-001 §6] |
| RPO | ≤ 30 minutes | ≤ 1 hour (standard) | Requirements exceed standard |
| RTO | ≤ 2 hours | ≤ 4 hours (standard) | Requirements exceed standard |
| DR Strategy | Active-Passive | Active-Passive | [CTSO-INFRA-001 §6] |
| Primary Region | Canada Central | Canada Central | [CTSO-INFRA-001 §1] |
| DR Region | Canada East | Canada East | [CTSO-INFRA-001 §1] |

> **Note:** The application requirements specify RPO ≤ 30 min and RTO ≤ 2 hours, which are **stricter** than Contoso's standard DR targets (RPO ≤ 1 hour, RTO ≤ 4 hours). The architecture is designed to meet the stricter requirements.

## High Availability (Within Region)

### Multi-Zone Deployment

All production compute and data services are deployed across **Availability Zones 1, 2, and 3** in Canada Central [CTSO-INFRA-001 §6].

| Component | Zone Configuration |
|---|---|
| AKS Node Pools | Nodes spread across 3 zones |
| Azure SQL (Business Critical) | Zone-redundant replicas |
| Azure Cosmos DB | Zone-redundant (automatic) |
| Azure Cache for Redis (Enterprise) | Zone-redundant |
| Azure Service Bus (Premium) | Zone-redundant |
| Azure Event Hubs | Zone-redundant |
| Azure Key Vault | Zone-redundant (automatic) |
| Azure Blob Storage | ZRS (Zone-Redundant Storage) |

### AKS Pod Resilience

| Configuration | Value |
|---|---|
| Minimum replicas per service | 2 (spread across zones via pod anti-affinity) |
| Horizontal Pod Autoscaler (HPA) | Enabled; CPU/memory-based + custom metrics |
| Pod Disruption Budget (PDB) | `minAvailable: 1` for each service |
| Node auto-scaler | Min 3, max 8 for app workloads |
| Topology spread constraints | Even distribution across zones |

## Disaster Recovery (Cross-Region)

### Active-Passive to Canada East

| Component | DR Strategy | RPO | RTO |
|---|---|---|---|
| AKS Cluster | Standby cluster in Canada East (IaC-provisioned, scaled down) | N/A | < 30 min (scale up) |
| Azure SQL | Auto-failover group (async replication) | < 5 min | < 1 hour (auto) |
| Cosmos DB | Multi-region replication (Canada East read replica) | ~0 (continuous) | < 5 min |
| Redis | Geo-replication to Canada East | < 15 min | < 30 min |
| Service Bus | Geo-DR pairing | < 5 min (metadata) | < 30 min |
| Blob Storage | GRS (automatic) | < 15 min | < 1 hour |
| ACR | Geo-replication to Canada East | ~0 (automatic) | Immediate |
| Key Vault | Manual failover (auto-recovery) | ~0 | < 5 min |
| Front Door | Global (automatic failover via health probes) | N/A | < 1 min |
| APIM | Multi-region deployment or standby | N/A | < 30 min |

### Failover Procedure

1. Azure Front Door detects backend health probe failure in Canada Central
2. Front Door automatically routes traffic to Canada East endpoint
3. Azure SQL auto-failover group promotes Canada East replica to primary
4. Cosmos DB serves reads from Canada East replica; writes redirect automatically
5. AKS standby cluster in Canada East scales up via automated runbook
6. Service Bus Geo-DR activates (metadata failover)
7. Redis geo-replica promoted to primary

### Failover Testing

- **Quarterly** automated failover testing required [CTSO-INFRA-001 §6]
- DR drills include:
  - Full regional failover simulation
  - Azure SQL failover group switchover
  - AKS cluster failover to Canada East
  - Backup restoration verification [CTSO-DATA-001 §5]

## Application Resiliency Patterns

### Retry with Exponential Backoff

Per [CTSO-APP-001 §7]:

```csharp
// Using Polly for .NET
services.AddHttpClient<EquifaxClient>()
    .AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt =>
                TimeSpan.FromSeconds(Math.Pow(2, attempt)),
            onRetry: (outcome, delay, attempt, context) =>
                logger.LogWarning("Retry {Attempt} after {Delay}s", attempt, delay.TotalSeconds)));
```

Applied to all external calls: Equifax API, Core Banking, Azure AI Services, database connections.

### Circuit Breaker

Per [CTSO-APP-001 §7]:

```csharp
// Circuit breaker via Polly
services.AddHttpClient<CoreBankingClient>()
    .AddTransientHttpErrorPolicy(policy =>
        policy.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));
```

- Prevents cascading failures when downstream services are degraded
- Direct service-to-service calls without circuit breakers are **prohibited** [CTSO-APP-001 §9]

### Bulkhead Isolation

Per [CTSO-APP-001 §7]:
- Separate thread pools / concurrent execution limits for each downstream dependency
- Prevents a slow dependency from consuming all service resources
- Implemented via Polly Bulkhead policy or Istio rate limiting

### Timeout Policies

- **30-second maximum** for synchronous API calls [CTSO-APP-001 §7]
- Shorter timeouts for internal service-to-service calls (10 seconds)
- Longer timeouts for external APIs (Equifax: 15 seconds) and ML inference (30 seconds)

### Graceful Degradation

Per [CTSO-APP-001 §7]:
- Credit Scoring Service: if ML model is unavailable, route to manual underwriting queue
- Document Processing: if OCR fails, allow manual document review by advisors
- Notification Service: if SendGrid/Twilio unavailable, queue for retry; do not fail the loan application
- Reporting Service: serve stale cached data if real-time aggregation fails

### Asynchronous Processing Chain Limit

- Synchronous chains limited to **≤ 3 services deep** [CTSO-APP-001 §9]
- Loan processing pipeline uses async messaging (Service Bus) to avoid deep synchronous chains:
  - Loan Intake → (async) → Identity Verification → (async) → Credit Scoring → (async) → Decision Engine

## Statelessness

All services are **stateless** [CTSO-APP-001 §9]:
- No in-memory session state
- Session data stored in Azure Cache for Redis
- All persistent data in external databases (Azure SQL, Cosmos DB, Blob Storage)
- Services can be freely scaled, restarted, or migrated across nodes

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| 99.95% SLA target | CTSO-INFRA-001 | §6 |
| Multi-zone deployment | CTSO-INFRA-001 | §6 |
| Active-Passive DR | CTSO-INFRA-001 | §6 |
| Quarterly failover testing | CTSO-INFRA-001 | §6 |
| Retry with exponential backoff | CTSO-APP-001 | §7 |
| Circuit breaker pattern | CTSO-APP-001 | §7 |
| Bulkhead isolation | CTSO-APP-001 | §7 |
| 30-second timeout maximum | CTSO-APP-001 | §7 |
| Graceful degradation | CTSO-APP-001 | §7 |
| No synchronous chains > 3 deep | CTSO-APP-001 | §9 |
| Stateless services | CTSO-APP-001 | §9 |
| GRS backups for databases | CTSO-DATA-001 | §5 |
| Auto-failover groups for Azure SQL | CTSO-DATA-001 | §5 |
| Quarterly backup restoration tests | CTSO-DATA-001 | §5 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| Multi-zone deployment | Required for all production | Recommended for production AKS; zone-redundant node pools | **Aligns** | [AKS Baseline](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| Active-Passive DR | Required | Active-Passive common for cost-efficient DR; Active-Active for mission-critical | **Aligns** — Active-Active would be preferred for mission-critical, but Active-Passive meets 99.95% SLA | [DR Best Practices](https://learn.microsoft.com/azure/well-architected/reliability/disaster-recovery) |
| Retry + circuit breaker | Polly for .NET required | Recommended; Microsoft.Extensions.Http.Resilience in .NET 8+ | **Aligns** — consider using built-in .NET 8 resilience | [.NET Resilience](https://learn.microsoft.com/dotnet/core/resilience/) |
| Auto-failover groups | Required for Tier-1 databases | Recommended for critical SQL databases | **Aligns** | [Azure SQL Failover Groups](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db) |
| Quarterly DR testing | Required | Recommended as regular practice | **Aligns** | [WAF Reliability Testing](https://learn.microsoft.com/azure/well-architected/reliability/testing-strategy) |
| RPO ≤ 30 min | Stricter than standard (≤ 1h) | Achievable with auto-failover groups (near-zero RPO) | **Exceeds standard** — architecture supports this | [Azure SQL Auto-Failover](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db) |
| Graceful degradation | Required for all services | Recommended; return partial results rather than full failures | **Aligns** | [Well-Architected Reliability](https://learn.microsoft.com/azure/well-architected/reliability/self-preservation) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | RPO ≤ 30 min exceeds Contoso standard (≤ 1h); requires ARB acknowledgment | Medium | Document deviation; auto-failover groups achieve near-zero RPO |
| 2 | RTO ≤ 2h requires AKS standby cluster to scale up quickly | Medium | Pre-provisioned standby with min 1 node; IaC for rapid scale-up |
| 3 | Service Bus Geo-DR only replicates metadata, not messages | Medium | Accept message loss during failover; design for idempotent processing |
| 4 | DR testing quarterly must be scheduled to avoid peak business periods | Low | Coordinate with VP Retail Lending for testing windows |
| 5 | Cost of maintaining standby infrastructure in Canada East | Medium | Use minimal node count in standby; auto-shutdown when not testing |
