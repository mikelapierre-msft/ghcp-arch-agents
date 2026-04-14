# Compute — Contoso Loan Application Platform

## Platform Selection: Azure Kubernetes Service (AKS)

AKS is selected as the primary compute platform based on:
- **Contoso preference order**: AKS is the #1 approved compute platform [CTSO-INFRA-001 §3]
- **Microservices scale**: 8+ microservices with independent scaling requirements
- **Service mesh support**: Istio for mTLS between services [CTSO-SEC-001 §4]
- **Team familiarity**: Contoso KYC service already runs on AKS, providing operational familiarity

### AKS Cluster Configuration

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| Cluster mode | Private cluster | [CTSO-INFRA-001 §3] |
| Network plugin | Azure CNI overlay | [CTSO-INFRA-001 §3] |
| Identity | Managed Identity (system-assigned) | [CTSO-INFRA-001 §3], [CTSO-IAM-001 §5] |
| Node pools | System pool + User pool (per workload tier) | Best practice |
| Availability zones | Zones 1, 2, 3 (Canada Central) | [CTSO-INFRA-001 §6] |
| Node autoscaling | Enabled (min 3 – max 15 nodes per pool) | [CTSO-INFRA-001 §8] |
| Container registry | Contoso ACR (geo-replicated) | [CTSO-INFRA-001 §3], [CTSO-APP-001 §4] |
| Kubernetes version | Latest stable (auto-upgrade channel: stable) | Best practice |
| SKU tier | Standard | [CTSO-INFRA-001 §8] |

### Node Pool Design

| Pool | VM SKU | Purpose | Min/Max Nodes | Taints |
|------|--------|---------|---------------|--------|
| `system` | Standard_D4s_v5 | System workloads (CoreDNS, metrics-server) | 3 / 5 | `CriticalAddonsOnly=true:NoSchedule` |
| `apppool` | Standard_D4s_v5 | Application microservices | 3 / 12 | None |
| `aipool` | Standard_D8s_v5 | Document Processing, ML inference | 2 / 6 | `workload=ai:NoSchedule` |

### Supplementary Compute

| Component | Platform | Justification |
|-----------|----------|---------------|
| Reporting dashboard | Azure App Service | Static reporting UI; simpler ops than AKS for standalone web app |
| Scheduled jobs (data archival, backup tests) | Azure Functions (timer-triggered) | Serverless-only workloads [CTSO-INFRA-001 §3] |

### Naming Convention

Following [CTSO-INFRA-001 §2]:

| Resource | Name |
|----------|------|
| AKS cluster (prod) | `ctso-loanapp-prod-aks-cc-001` |
| AKS cluster (DR) | `ctso-loanapp-prod-aks-ce-001` |
| ACR | `ctsoloanappprodacrcc001` |
| App Service (reporting) | `ctso-loanapp-prod-app-cc-001` |

### Required Tags

All resources tagged per [CTSO-INFRA-001 §2]:

```json
{
  "CostCenter": "CC-RETAIL-LENDING-4200",
  "Owner": "digital-banking-team@contoso.com",
  "Environment": "Production",
  "DataClassification": "Confidential",
  "ApplicationName": "LoanApplicationPlatform"
}
```

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| AKS as primary compute | CTSO-INFRA-001 | §3 |
| Private cluster mode | CTSO-INFRA-001 | §3 |
| Azure CNI overlay networking | CTSO-INFRA-001 | §3 |
| Managed identity for cluster | CTSO-INFRA-001 | §3, CTSO-IAM-001 §5 |
| Multi-zone deployment | CTSO-INFRA-001 | §6 |
| Node autoscaling enabled | CTSO-INFRA-001 | §8 |
| Contoso ACR for images | CTSO-INFRA-001 | §3, CTSO-APP-001 §4 |
| Azure Functions for serverless tasks | CTSO-INFRA-001 | §3 |
| Naming convention compliance | CTSO-INFRA-001 | §2 |
| Resource tagging | CTSO-INFRA-001 | §2 |
| Container image scanning | CTSO-SEC-001 | §5, CTSO-APP-001 §4 |
| No `latest` tag in production | CTSO-APP-001 | §4 |
| VMs not used | CTSO-INFRA-001 | §3 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Compute platform | AKS preferred for microservices | AKS recommended for microservices at scale | **Aligns** | [Microservices on AKS](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices) |
| Networking | Azure CNI overlay required | Azure CNI powered by Cilium recommended for production | **Aligns** (Cilium is a superset) | [AKS networking](https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium) |
| Private cluster | Mandatory for all AKS | Recommended for production workloads | **Aligns** | [Private AKS clusters](https://learn.microsoft.com/azure/aks/private-clusters) |
| Autoscaling | Node autoscaling required | Cluster autoscaler + HPA recommended | **Aligns** | [AKS best practices](https://learn.microsoft.com/azure/aks/best-practices-app-cluster-reliability) |
| Image management | Contoso ACR only; public Docker Hub prohibited | ACR recommended; image quarantine pattern | **Exceeds** (stricter than default) | [ACR best practices](https://learn.microsoft.com/azure/container-registry/container-registry-best-practices) |
| Container scanning | Critical/High CVEs blocked from prod | Microsoft Defender for Containers recommended | **Aligns** | [Defender for Containers](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| AI workloads may need GPU nodes for ML inference | Start with CPU-based inference; add GPU node pool if latency exceeds P95 targets | Monitor post-MVP |
| AKS cluster upgrade windows may impact availability | Use Blue-Green node pool upgrade strategy with PodDisruptionBudgets | Implement in GA |
| Budget constraint ($15K/month) may limit node pool sizing | Use Reserved Instances for base capacity; autoscale for burst | Confirm RI pricing |
| Canada Central AZ availability for specific VM SKUs | Validate D4s_v5 and D8s_v5 availability in all 3 zones | Pre-deployment check |
