# Observability — Contoso Loan Application Platform

## Observability Strategy

The platform implements the **three pillars of observability** — logs, metrics, and distributed traces — using **OpenTelemetry** as the vendor-neutral instrumentation standard, exporting to Azure Monitor / Application Insights [CTSO-APP-001 §5].

## Instrumentation

### OpenTelemetry Configuration

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SDK | OpenTelemetry .NET SDK | [CTSO-APP-001 §5] |
| Exporter | OTLP → Azure Monitor (Application Insights) | [CTSO-APP-001 §5] |
| Auto-instrumentation | ASP.NET Core, HttpClient, SQL, gRPC, Redis | Best practice |
| Custom spans | Business operations (loan submission, credit check, decision) | Application-specific |
| Correlation | W3C Trace Context (distributed tracing) | [CTSO-APP-001 §5] |

### Logging

| Requirement | Implementation | Standard Reference |
|------------|---------------|-------------------|
| Format | Structured JSON (Serilog / Microsoft.Extensions.Logging) | [CTSO-APP-001 §5] |
| Correlation IDs | Trace ID / Span ID in every log entry | [CTSO-APP-001 §5] |
| PII protection | PII **never** logged; redaction middleware applied | [CTSO-DATA-001 §8] |
| Console.WriteLine | **Prohibited** in production | [CTSO-APP-001 §5] |
| Retention | 90 days in Application Insights; 365 days for security/audit logs in Sentinel | [CTSO-SEC-001 §6] |

### Metrics

| Metric Category | Examples | Collection |
|----------------|---------|-----------|
| Application | Request rate, error rate, response time (P50/P95/P99) | OpenTelemetry |
| Business | Loan submissions/hour, approval rate, SLA compliance | Custom metrics |
| Infrastructure | CPU, memory, disk, network (per pod/node) | Azure Monitor + Prometheus |
| Database | Query duration, connection pool usage, DTU consumption | Azure SQL Insights |
| Messaging | Queue depth, message throughput, dead-letter count | Service Bus metrics |

### Distributed Tracing

End-to-end trace flow for a loan application submission:

```
Customer → Front Door → APIM → App Intake → [Service Bus] → 
  ├── ID Verify → Equifax API
  ├── Credit Scoring → Azure ML
  └── Doc Processing → Document Intelligence
→ [Service Bus] → Decision Engine → [Service Bus] → Notification Service
```

Each service propagates W3C Trace Context headers. Istio service mesh provides automatic span injection for gRPC calls.

## Health Checks

All services expose mandatory health endpoints [CTSO-APP-001 §5]:

| Endpoint | Purpose | Checks |
|----------|---------|--------|
| `/health/live` | Liveness probe | Process is running, not deadlocked |
| `/health/ready` | Readiness probe | Database connectivity, Service Bus connectivity, cache reachable |

Kubernetes probes configured:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Azure Monitor Configuration

| Component | Configuration | Purpose |
|-----------|--------------|---------|
| Application Insights | Per-service instances (shared workspace) | APM, distributed tracing |
| Log Analytics Workspace | Centralized workspace | Unified log querying |
| Azure Monitor for Containers | AKS cluster monitoring | Node/pod/container metrics |
| Azure SQL Insights | Database performance monitoring | Query performance, wait stats |
| Service Bus Metrics | Namespace-level monitoring | Queue health, throughput |

### Diagnostic Settings

All Azure resources have diagnostic settings enabled [CTSO-SEC-001 §6]:

| Resource | Logs | Metrics | Destination |
|----------|------|---------|-------------|
| AKS | kube-audit, kube-apiserver, guard | All metrics | Log Analytics + Sentinel |
| Azure SQL | SQLSecurityAuditEvents, QueryStoreRuntimeStatistics | All metrics | Log Analytics |
| Key Vault | AuditEvent | All metrics | Log Analytics + Sentinel |
| Azure Firewall | AZFWApplicationRule, AZFWNetworkRule | All metrics | Log Analytics + Sentinel |
| Front Door | FrontDoorAccessLog, FrontDoorHealthProbeLog | All metrics | Log Analytics |
| APIM | GatewayLogs | All metrics | Log Analytics + Application Insights |
| Service Bus | OperationalLogs | All metrics | Log Analytics |

## Alerting Strategy

### Critical Alerts (P1 — Immediate Response)

| Alert | Condition | Action |
|-------|-----------|--------|
| Service availability < 99.95% | Application Insights availability test fails | Page on-call engineer |
| Error rate > 5% | 5xx responses exceed 5% in 5-minute window | Page on-call engineer |
| Database failover triggered | Azure SQL failover group state change | Page DBA + on-call |
| Security event | Sentinel incident (High severity) | Page security on-call |

### Warning Alerts (P2 — Business Hours Response)

| Alert | Condition | Action |
|-------|-----------|--------|
| API response time P95 > 500ms (reads) | Application Insights performance degradation | Notify team channel |
| API response time P95 > 2s (writes) | Application Insights performance degradation | Notify team channel |
| Queue depth > 1000 messages | Service Bus metric threshold | Notify team channel |
| Pod restarts > 3 in 15 min | AKS container restart count | Notify team channel |
| CPU > 80% sustained 10 min | AKS node metric | Notify team channel |

### Informational Alerts (P3)

| Alert | Condition | Action |
|-------|-----------|--------|
| Deployment completed | Pipeline status | Log to dashboard |
| Feature flag toggled | App Configuration change event | Log to dashboard |
| Daily loan volume summary | Scheduled query | Email to stakeholders |

## Dashboards

| Dashboard | Audience | Tool | Key Metrics |
|-----------|----------|------|-------------|
| Platform Health | Operations team | Azure Managed Grafana | Availability, error rates, latency, pod health |
| Loan Pipeline | Product Owner | Power BI (via Azure SQL read replica) | Applications in progress, approval rates, SLA compliance |
| Security Operations | CISO Office | Microsoft Sentinel Workbooks | Sign-in anomalies, threat detections, compliance posture |
| Cost Management | Finance | Azure Cost Management | Daily spend, resource utilization, RI coverage |

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| OpenTelemetry for instrumentation | CTSO-APP-001 | §5 |
| Export to Application Insights via OTLP | CTSO-APP-001 | §5 |
| Structured JSON logging | CTSO-APP-001 | §5 |
| Correlation IDs in all logs | CTSO-APP-001 | §5 |
| /health/live and /health/ready endpoints | CTSO-APP-001 | §5 |
| No PII in logs | CTSO-DATA-001 | §8 |
| Diagnostic settings on all resources | CTSO-SEC-001 | §6 |
| Security events to Sentinel within 5 min | CTSO-SEC-001 | §6 |
| 365-day audit log retention | CTSO-SEC-001 | §6 |
| Failed auth alert after 5 failures | CTSO-SEC-001 | §6 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Instrumentation | OpenTelemetry SDK | OpenTelemetry recommended as standard | **Aligns** | [OpenTelemetry on Azure](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-overview) |
| APM | Application Insights | Application Insights recommended for .NET | **Aligns** | [App Insights for .NET](https://learn.microsoft.com/azure/azure-monitor/app/asp-net-core) |
| Container monitoring | Azure Monitor for Containers | Recommended for AKS observability | **Aligns** | [AKS monitoring](https://learn.microsoft.com/azure/aks/monitor-aks) |
| Health endpoints | /health/live and /health/ready | Liveness + readiness probes recommended | **Aligns** | [Health monitoring](https://learn.microsoft.com/azure/architecture/patterns/health-endpoint-monitoring) |
| Structured logging | JSON format mandatory | Recommended for cloud-native apps | **Aligns** | [Logging best practices](https://learn.microsoft.com/azure/azure-monitor/app/ilogger) |
| Log retention | 365 days for security | 90 days default; longer for compliance | **Exceeds** | [Data retention](https://learn.microsoft.com/azure/azure-monitor/logs/data-retention-configure) |
| Dashboarding | Grafana + Power BI + Sentinel | Grafana recommended for infra; Power BI for business | **Aligns** | [Azure Managed Grafana](https://learn.microsoft.com/azure/managed-grafana/overview) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| 365-day Log Analytics retention cost | Use Basic logs for high-volume telemetry; archive to Storage for long-term | Estimate cost |
| OpenTelemetry .NET SDK maturity for all signals | Use stable APIs; fall back to Application Insights SDK for unsupported scenarios | Monitor SDK releases |
| PII leakage risk in traces/logs | Implement PII detection middleware; regular audit of telemetry data | Pre-GA security review |
| Alert fatigue from excessive notifications | Tune thresholds after baseline period; implement progressive escalation | Post-MVP tuning |
