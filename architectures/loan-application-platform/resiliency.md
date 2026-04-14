# Resiliency — Contoso Loan Application Platform

## Availability & DR Targets

| Metric | Requirement | Contoso Standard | Notes |
|--------|------------|-----------------|-------|
| Availability | 99.95% | [CTSO-INFRA-001 §6] | Translates to ~22 min/month downtime |
| RPO | ≤ 30 minutes | Exceeds [CTSO-INFRA-001 §6] (≤ 1h standard) | Requirement-driven |
| RTO | ≤ 2 hours | Exceeds [CTSO-INFRA-001 §6] (≤ 4h standard) | Requirement-driven |
| DR strategy | Active-Passive | [CTSO-INFRA-001 §6] | Canada Central (active) → Canada East (passive) |

> **Note**: The application requires RPO ≤ 30 min and RTO ≤ 2 hours, which **exceeds** the Contoso baseline of RPO ≤ 1 hour and RTO ≤ 4 hours [CTSO-INFRA-001 §6]. This is driven by regulatory requirements (OSFI operational resilience guidelines).

## High Availability (Within Region)

### Multi-Zone Deployment

All production services deployed across **3 availability zones** in Canada Central [CTSO-INFRA-001 §6]:

| Component | Zone Configuration | HA Mechanism |
|-----------|-------------------|-------------|
| AKS cluster | Nodes across zones 1, 2, 3 | Pod anti-affinity + zone-spread topology |
| Azure SQL Database | Business Critical (zone-redundant) | Built-in zone replication |
| Azure Cosmos DB | Zone-redundant in Canada Central | Automatic failover between zones |
| Azure Cache for Redis | Enterprise (zone-redundant) | Active geo-replication |
| Azure Service Bus | Premium (zone-redundant) | Built-in zone replication |
| Azure Front Door | Global (multi-PoP) | Automatic failover |
| Application Gateway | WAF_v2 (zone-redundant) | Cross-zone distribution |
| Azure Key Vault | Zone-redundant (built-in) | Automatic |

### AKS Resiliency Configuration

```yaml
# Pod Disruption Budget (PDB) — ensure minimum availability during updates
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: application-intake-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: application-intake

# Topology Spread Constraints — distribute across zones
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: application-intake
```

Minimum **2 replicas** per microservice in production. Critical path services (Application Intake, Decision Engine) run **3 replicas**.

## Disaster Recovery (Cross-Region)

### Active-Passive Configuration

| Component | Primary (Canada Central) | DR (Canada East) | Failover Mechanism | RPO |
|-----------|-------------------------|-------------------|-------------------|-----|
| AKS cluster | Active | Standby (scaled-down) | Manual DNS failover via Front Door | ~0 (stateless) |
| Azure SQL Database | Read-write | Read-only (auto-failover group) | Automatic failover group | ~0 (sync replication) |
| Azure Cosmos DB | Write region | Read region | Manual promotion to write | < 30 min |
| Azure Cache for Redis | Active | Passive (geo-replication) | Manual failover | < 5 min |
| Azure Service Bus | Active | Passive (geo-DR pairing) | Manual failover | Messages may be lost |
| Key Vault | Active | Replicated (built-in) | Automatic (read-only during failover) | ~0 |
| Container Registry | Active (geo-replicated) | Replica | Automatic | ~0 |

### Failover Procedure

1. **Detection**: Azure Front Door health probes detect Canada Central backend failure
2. **Decision**: On-call engineer validates via runbook (auto-failover for Azure SQL)
3. **DNS Switch**: Update Front Door origin to point to Canada East Application Gateway
4. **AKS Scale-Up**: Scale Canada East AKS cluster from standby to active node count
5. **Data Failover**: Azure SQL auto-failover group activates; Cosmos DB promoted to write
6. **Validation**: Smoke tests against DR endpoints
7. **Communication**: Status page update and stakeholder notification

**Automated failover testing** required quarterly [CTSO-INFRA-001 §6].

## Resiliency Patterns (Application-Level)

### Retry with Exponential Backoff

All external calls use **Polly** (.NET) retry policies [CTSO-APP-001 §7]:

| Dependency | Max Retries | Initial Delay | Max Delay | Jitter |
|-----------|-------------|---------------|-----------|--------|
| Azure SQL | 3 | 200ms | 5s | Yes |
| Cosmos DB | 3 (SDK built-in) | 200ms | 5s | Yes |
| Service Bus | 5 | 500ms | 30s | Yes |
| Equifax API | 3 | 1s | 10s | Yes |
| Azure ML (credit scoring) | 2 | 2s | 15s | Yes |
| SendGrid / Twilio | 3 | 1s | 30s | Yes |

### Circuit Breaker

Circuit breaker pattern for all downstream service calls [CTSO-APP-001 §7]:

| Dependency | Failure Threshold | Break Duration | Half-Open Attempts |
|-----------|-------------------|----------------|-------------------|
| Equifax API | 5 failures in 30s | 60 seconds | 2 |
| Credit Scoring (ML) | 3 failures in 60s | 120 seconds | 1 |
| Core Banking (on-prem) | 3 failures in 60s | 120 seconds | 1 |
| Notification services | 5 failures in 30s | 60 seconds | 2 |

### Bulkhead Isolation

Bulkhead pattern to prevent cascading failures [CTSO-APP-001 §7]:

| Service | Max Concurrent Calls | Queue Size | Purpose |
|---------|---------------------|-----------|---------|
| Application Intake → Equifax | 50 | 100 | Protect from Equifax slowdown |
| Decision Engine → ML | 30 | 50 | Protect from ML inference latency |
| Notification Service → SendGrid | 100 | 500 | Protect from email provider issues |
| Any service → Core Banking | 20 | 30 | Protect from on-prem latency |

### Timeout Policies

Maximum **30-second timeout** for all synchronous API calls [CTSO-APP-001 §7]:

| Call Type | Timeout | Fallback |
|-----------|---------|----------|
| Internal gRPC calls | 10s | Return cached result or error |
| Database queries | 15s | Return error with retry guidance |
| Equifax API | 20s | Queue for async retry |
| Azure ML inference | 25s | Route to manual underwriting queue |
| Core Banking | 30s | Queue for async retry |

### Graceful Degradation

Services return **partial results** rather than failing entirely [CTSO-APP-001 §7]:

| Scenario | Degraded Behavior |
|----------|------------------|
| Credit scoring service unavailable | Route application to manual underwriting queue |
| Equifax API timeout | Accept application, queue for async credit check |
| Document Intelligence down | Accept document upload, queue for later OCR processing |
| Notification service failure | Log notification intent, retry via background job |
| Reporting database unavailable | Show cached dashboard data with "stale data" indicator |

## Backup & Recovery

| Component | Backup Type | Retention | Recovery Test | Standard Reference |
|-----------|------------|-----------|--------------|-------------------|
| Azure SQL | Automated + LTR | 35-day PITR, 7-year LTR | Quarterly | [CTSO-DATA-001 §5] |
| Cosmos DB | Continuous backup | 7-day PITR | Quarterly | [CTSO-DATA-001 §5] |
| Blob Storage (documents) | GRS | Soft delete 30 days + lifecycle | Quarterly | [CTSO-DATA-001 §5] |
| AKS config | GitOps (declarative) | Git history | Every deployment | IaC |
| Key Vault | Soft delete + purge protection | 90 days | Quarterly | Best practice |

Backup restoration tests required **quarterly** [CTSO-DATA-001 §5].

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| 99.95% SLA target | CTSO-INFRA-001 | §6 |
| Multi-zone deployment | CTSO-INFRA-001 | §6 |
| Active-Passive DR | CTSO-INFRA-001 | §6 |
| Quarterly failover testing | CTSO-INFRA-001 | §6 |
| Retry with exponential backoff | CTSO-APP-001 | §7 |
| Circuit breaker pattern | CTSO-APP-001 | §7 |
| Bulkhead isolation | CTSO-APP-001 | §7 |
| 30-second timeout maximum | CTSO-APP-001 | §7 |
| Graceful degradation | CTSO-APP-001 | §7 |
| Automated backups with GRS | CTSO-DATA-001 | §5 |
| Quarterly backup restore tests | CTSO-DATA-001 | §5 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Availability SLA | 99.95% mandatory | Depends on business criticality; 99.95% for mission-critical | **Aligns** | [Well-Architected reliability](https://learn.microsoft.com/azure/well-architected/reliability/checklist) |
| Multi-zone | Required for production | Recommended for production workloads | **Aligns** | [AKS availability zones](https://learn.microsoft.com/azure/aks/availability-zones) |
| DR strategy | Active-Passive mandatory | Active-Active or Active-Passive based on RPO/RTO | **Aligns** (passive appropriate for budget) | [DR for Azure apps](https://learn.microsoft.com/azure/architecture/framework/resiliency/backup-and-recovery) |
| Retry patterns | Polly for .NET | Polly recommended for .NET resilience | **Aligns** | [Transient fault handling](https://learn.microsoft.com/azure/architecture/best-practices/transient-faults) |
| Circuit breaker | Required for all downstream calls | Recommended pattern for microservices | **Aligns** | [Circuit breaker pattern](https://learn.microsoft.com/azure/architecture/patterns/circuit-breaker) |
| Bulkhead | Required for isolation | Recommended for preventing cascading failures | **Aligns** | [Bulkhead pattern](https://learn.microsoft.com/azure/architecture/patterns/bulkhead) |
| Graceful degradation | Required — partial results over failure | Recommended for user experience | **Aligns** | [Graceful degradation](https://learn.microsoft.com/azure/architecture/framework/resiliency/resilience-checklist) |
| RPO/RTO | RPO ≤ 30 min, RTO ≤ 2h (exceeds baseline) | Define based on business impact analysis | **Exceeds** Contoso baseline | [RPO/RTO guidance](https://learn.microsoft.com/azure/well-architected/reliability/metrics) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| RPO ≤ 30 min exceeds Contoso standard RPO ≤ 1h; requires tighter replication | Azure SQL auto-failover group provides near-zero RPO; Cosmos DB continuous backup provides sub-minute RPO | **Achievable** |
| RTO ≤ 2h exceeds Contoso standard RTO ≤ 4h; requires faster failover | Pre-deployed (scaled-down) DR AKS cluster; automated runbooks for failover | Design for automation |
| DR AKS cluster cost when idle | Scale to minimum nodes (1 per pool); use Spot instances for DR standby | Cost optimization |
| Service Bus geo-DR may lose in-flight messages | Implement idempotent message processing; duplicate detection | Design pattern |
| Quarterly failover tests may impact customers | Schedule during maintenance windows; use progressive failover (canary-style) | Operational procedure |
