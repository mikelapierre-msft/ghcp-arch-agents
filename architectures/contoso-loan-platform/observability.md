# Observability Design: Contoso Loan Application Platform

## Observability Strategy

All services implement the **three pillars of observability** — logs, metrics, and distributed traces — using **OpenTelemetry** for vendor-neutral instrumentation [CTSO-APP-001 §5].

## Instrumentation

### OpenTelemetry

| Configuration | Value | Standard |
|---|---|---|
| SDK | OpenTelemetry .NET SDK | [CTSO-APP-001 §5] |
| Exporter | OTLP → Azure Monitor / Application Insights | [CTSO-APP-001 §5] |
| Auto-instrumentation | ASP.NET Core, HttpClient, gRPC, Entity Framework, Redis | Comprehensive coverage |
| Custom spans | Business-critical operations (loan submission, credit scoring, decision making) | Domain-specific tracing |
| Correlation IDs | W3C Trace Context propagation across all services | [CTSO-APP-001 §5] |

### Structured Logging

- **JSON-formatted** structured logging for all services [CTSO-APP-001 §5]
- Unstructured `Console.WriteLine` / `print()` is **prohibited** in production [CTSO-APP-001 §5]
- Correlation IDs included in all log entries for end-to-end tracing
- **PII must never be logged** [CTSO-DATA-001 §8] — sensitive fields redacted at the logging layer

### Health Check Endpoints

Every microservice exposes [CTSO-APP-001 §5]:

| Endpoint | Purpose | Used By |
|---|---|---|
| `/health/live` | Liveness — is the process running? | Kubernetes liveness probe |
| `/health/ready` | Readiness — can the service accept traffic? | Kubernetes readiness probe |

Readiness checks include dependency validation (database connectivity, downstream service availability).

## Monitoring Platform

### Azure Monitor & Application Insights

| Component | Purpose | Configuration |
|---|---|---|
| **Application Insights** | APM, distributed tracing, request/dependency tracking | One workspace per environment; OTLP ingestion |
| **Log Analytics Workspace** | Centralized log storage and KQL queries | Single workspace for all platform services |
| **Azure Monitor Metrics** | Infrastructure and custom metrics | AKS, SQL, Cosmos, Redis, Service Bus metrics |
| **Azure Monitor Alerts** | Proactive alerting on SLA breaches | Action groups → email, SMS, PagerDuty |
| **Azure Workbooks** | Interactive dashboards | Loan pipeline, SLA tracking, error analysis |

### Microsoft Sentinel (Security Monitoring)

Per [CTSO-SEC-001 §6]:
- Security events forwarded within **5 minutes**
- Failed authentication alerts after **5 consecutive failures**
- Audit log retention: **365 days**
- Diagnostic settings enabled on **all Azure resources**

### Azure Monitor for AKS

| Feature | Configuration |
|---|---|
| Container Insights | Enabled — node, pod, container metrics |
| Managed Prometheus | Enabled — Kubernetes metrics collection |
| Managed Grafana | Dashboards for AKS health and performance |
| Azure Monitor Network Observability | Enabled via Advanced Container Networking Services |

## Key Metrics & SLAs

### Application-Level SLIs

| Metric | Target | Alert Threshold |
|---|---|---|
| API availability | 99.95% | < 99.9% |
| Read API latency (P95) | < 500ms | > 500ms |
| Submission API latency (P95) | < 2s | > 2s |
| Error rate (5xx) | < 0.1% | > 0.5% |
| Loan processing time (end-to-end) | < 5 minutes (automated) | > 10 minutes |
| Document processing time | < 2 minutes | > 5 minutes |

### Infrastructure-Level SLIs

| Metric | Target | Alert Threshold |
|---|---|---|
| AKS node CPU utilization | < 70% sustained | > 80% for 5 min |
| AKS node memory utilization | < 75% sustained | > 85% for 5 min |
| Azure SQL DTU/vCore utilization | < 70% | > 80% for 10 min |
| Cosmos DB RU consumption | < 80% provisioned | > 90% for 5 min |
| Redis memory utilization | < 75% | > 85% |
| Service Bus queue depth | < 1000 messages | > 5000 messages |
| Service Bus dead-letter count | 0 | > 0 |

### Business-Level KPIs

| KPI | Dashboard | Data Source |
|---|---|---|
| Loan applications submitted per hour | Reporting Dashboard | Event Hubs → Synapse |
| Approval rate (%) | Reporting Dashboard | Decision Engine events |
| Average time to decision | Reporting Dashboard | Application Insights traces |
| Document upload success rate | Reporting Dashboard | Document Processing service metrics |
| Advisor queue depth (referred applications) | Advisor Portal | Service Bus queue metrics |

## Alerting Strategy

### Alert Severity Levels

| Severity | Response Time | Notification | Example |
|---|---|---|---|
| Sev 0 — Critical | Immediate (< 15 min) | PagerDuty + SMS + Email | Platform down; all APIs failing |
| Sev 1 — High | < 30 min | PagerDuty + Email | SLA breach; error rate > 1% |
| Sev 2 — Medium | < 2 hours | Email + Teams | Elevated latency; queue buildup |
| Sev 3 — Low | Next business day | Email | Capacity warnings; cost alerts |

### Key Alert Rules

| Alert | Condition | Severity | Action Group |
|---|---|---|---|
| API Availability Drop | Availability < 99.9% over 5 min | Sev 0 | On-call SRE |
| High Error Rate | 5xx rate > 0.5% over 5 min | Sev 1 | On-call SRE |
| Database Failover | Auto-failover group state change | Sev 0 | On-call SRE + DBA |
| Credit Scoring Timeout | ML model inference > 30s | Sev 2 | ML Engineering |
| Dead Letter Queue | Messages > 0 in DLQ | Sev 2 | Platform Engineering |
| Security Alert | Defender for Cloud alert | Sev 1 | Security Operations |
| Failed Auth Spike | > 5 consecutive failures per user | Sev 2 | Security Operations |

## Log Retention

| Log Type | Retention | Standard |
|---|---|---|
| Application logs | 90 days in Log Analytics; 365 days in archive | [CTSO-SEC-001 §6] |
| Security / audit logs | 365 days | [CTSO-SEC-001 §6] |
| NSG flow logs | 90 days | [CTSO-NET-001 §7] |
| Azure Firewall logs | 90 days | [CTSO-NET-001 §5] |
| Diagnostic settings | Enabled on all resources | [CTSO-SEC-001 §6] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| Three pillars: logs, metrics, traces | CTSO-APP-001 | §5 |
| OpenTelemetry SDK instrumentation | CTSO-APP-001 | §5 |
| OTLP export to Application Insights | CTSO-APP-001 | §5 |
| Structured JSON logging | CTSO-APP-001 | §5 |
| Correlation IDs in all logs | CTSO-APP-001 | §5 |
| Health check endpoints (/health/live, /health/ready) | CTSO-APP-001 | §5 |
| No PII in logs or telemetry | CTSO-DATA-001 | §8 |
| Security events to Sentinel within 5 min | CTSO-SEC-001 | §6 |
| Failed auth alert after 5 failures | CTSO-SEC-001 | §6 |
| 365-day audit log retention | CTSO-SEC-001 | §6 |
| Diagnostic settings on all resources | CTSO-SEC-001 | §6 |
| NSG flow logs 90-day retention | CTSO-NET-001 | §7 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| OpenTelemetry | Required for all services | Recommended; Azure Monitor supports OTLP natively | **Aligns** | [Azure Monitor OpenTelemetry](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable) |
| Structured logging | JSON format mandatory | Recommended for searchability and correlation | **Aligns** | [Application Insights Logging](https://learn.microsoft.com/azure/azure-monitor/app/ilogger) |
| Container Insights | Enabled on AKS | Recommended for AKS monitoring | **Aligns** | [AKS Monitoring](https://learn.microsoft.com/azure/aks/monitor-aks) |
| Managed Prometheus + Grafana | Used for Kubernetes metrics | Recommended as cloud-native monitoring for AKS | **Aligns** | [Azure Managed Prometheus](https://learn.microsoft.com/azure/azure-monitor/essentials/prometheus-metrics-overview) |
| Health endpoints | `/health/live` and `/health/ready` mandatory | Recommended for Kubernetes liveness/readiness probes | **Aligns** | [AKS Health Probes](https://learn.microsoft.com/azure/aks/concepts-health-probes) |
| Audit log retention | 365 days minimum | Configurable; Sentinel supports long-term retention | **Exceeds** — Many orgs use 90-day defaults | [Sentinel Data Retention](https://learn.microsoft.com/azure/sentinel/configure-data-retention-archive) |
| No PII in telemetry | Strictly prohibited | Recommended; use data scrubbing / custom telemetry processors | **Aligns** | [Application Insights Data Privacy](https://learn.microsoft.com/azure/azure-monitor/app/data-retention-privacy) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | Log Analytics costs can grow significantly with 365-day retention | Medium | Use archival tier for logs > 90 days; set daily ingestion caps |
| 2 | PII scrubbing must be validated in all service logging configurations | High | Implement custom OpenTelemetry processors to redact sensitive fields |
| 3 | Business KPI dashboards require coordination with Product Owner | Low | Define KPIs during MVP planning; implement iteratively |
| 4 | Alert fatigue risk with too many low-severity alerts | Low | Tune thresholds after 2-week burn-in period post-launch |
