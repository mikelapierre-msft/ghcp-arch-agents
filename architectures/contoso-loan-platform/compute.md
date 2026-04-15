# Compute Design: Contoso Loan Application Platform

## Platform Selection

**Primary Compute:** Azure Kubernetes Service (AKS) — Private Cluster mode with Azure CNI overlay and managed identity [CTSO-INFRA-001 §3].

AKS is selected as the top-preference compute platform for this customer-facing microservices workload. The platform supports the required multi-zone availability, auto-scaling for 10,000 concurrent users, and private cluster mode to eliminate public API server exposure.

### AKS Cluster Configuration

| Parameter | Value | Standard Reference |
|---|---|---|
| Cluster Mode | Private Cluster | [CTSO-INFRA-001 §3] |
| Network Plugin | Azure CNI Overlay | [CTSO-INFRA-001 §3] |
| Identity | System-Assigned Managed Identity | [CTSO-INFRA-001 §3], [CTSO-IAM-001 §5] |
| Node Pool Scaling | Cluster Auto-scaler enabled | [CTSO-INFRA-001 §8] |
| Availability Zones | Zones 1, 2, 3 | [CTSO-INFRA-001 §6] |
| Kubernetes Version | Latest stable (1.30+) | Maintained per AKS support policy |
| Node SKU (System) | Standard_D4s_v5 (3 nodes min) | Sized for system workloads |
| Node SKU (User) | Standard_D4s_v5 (3-8 nodes, auto-scale) | Sized for application workloads |
| Container Runtime | containerd | Default for AKS |
| Workload Identity | Enabled (Microsoft Entra Workload ID) | [CTSO-IAM-001 §5] |
| Azure Policy | Enabled with deployment safeguards | [CTSO-SEC-001 §5] |
| Defender for Containers | Enabled | [CTSO-SEC-001 §5] |

### Node Pool Design

| Pool | Purpose | Min Nodes | Max Nodes | SKU | Taints |
|---|---|---|---|---|---|
| system | System pods (CoreDNS, kube-proxy) | 3 | 3 | Standard_D4s_v5 | `CriticalAddonsOnly=true:NoSchedule` |
| appworkloads | Application microservices | 3 | 8 | Standard_D4s_v5 | None |
| mlworkloads | Credit scoring ML inference | 1 | 3 | Standard_D8s_v5 | `workload=ml:NoSchedule` |

### Container Registry

- **Contoso ACR** with geo-replication (Canada Central + Canada East) [CTSO-INFRA-001 §3], [CTSO-APP-001 §4]
- Premium SKU for Private Endpoint support and geo-replication
- Contoso-approved base images (Microsoft MCR distroless / CBL-Mariner) [CTSO-APP-001 §4]
- Trivy + Microsoft Defender for Containers image scanning [CTSO-APP-001 §4]
- Immutable image tags (SHA digests) in production; `latest` tag prohibited [CTSO-APP-001 §4]
- Maximum image size: 500 MB [CTSO-APP-001 §4]

### Service Mesh

**Istio** service mesh on AKS [CTSO-APP-001 §3] for:
- **mTLS** enforcement between all services [CTSO-SEC-001 §4]
- Traffic management (canary deployments, traffic splitting)
- Observability (distributed tracing integration)
- Circuit breaking and retry policies at the mesh level

### Naming Convention

```
ctso-loanplatform-{env}-aks-cc-001
ctso-loanplatform-{env}-acr-cc-001
```
Per [CTSO-INFRA-001 §2]: `{company}-{workload}-{env}-{resource}-{region}-{instance}`

### Required Tags

| Tag | Value |
|---|---|
| CostCenter | `CC-RetailLending-4520` |
| Owner | `VP Retail Lending` |
| Environment | `prod` / `staging` / `qa` / `dev` |
| DataClassification | `Confidential` |
| ApplicationName | `LoanApplicationPlatform` |

Per [CTSO-INFRA-001 §2].

## Microservices Containerization

All services follow multi-stage Docker builds [CTSO-APP-001 §4]:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

# Runtime stage (distroless)
FROM ctso-acr.azurecr.io/base/dotnet-runtime:8.0-cbl-mariner
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
USER nonroot
ENTRYPOINT ["dotnet", "Service.dll"]
```

### Health Check Endpoints

Every service exposes [CTSO-APP-001 §5]:
- `/health/live` — Liveness probe (is the process running?)
- `/health/ready` — Readiness probe (can the service accept traffic?)

### Statelessness

All services are stateless [CTSO-APP-001 §9]. Session state stored in Azure Cache for Redis; persistent data in Azure SQL or Cosmos DB.

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| AKS as primary compute | CTSO-INFRA-001 | §3 |
| Private cluster mode | CTSO-INFRA-001 | §3 |
| Azure CNI Overlay networking | CTSO-INFRA-001 | §3 |
| Managed Identity for AKS | CTSO-INFRA-001 §3, CTSO-IAM-001 | §5 |
| Multi-zone deployment | CTSO-INFRA-001 | §6 |
| Contoso ACR with geo-replication | CTSO-INFRA-001 §3, CTSO-APP-001 | §4 |
| Approved base images only | CTSO-APP-001 | §4 |
| Image scanning (Trivy + Defender) | CTSO-APP-001 §4, CTSO-SEC-001 | §5 |
| Immutable image tags | CTSO-APP-001 | §4 |
| Multi-stage Docker builds | CTSO-APP-001 | §4 |
| Health check endpoints | CTSO-APP-001 | §5 |
| Istio service mesh | CTSO-APP-001 | §3 |
| Stateless services | CTSO-APP-001 | §9 |
| Resource naming convention | CTSO-INFRA-001 | §2 |
| Required tags | CTSO-INFRA-001 | §2 |
| Node auto-scaling enabled | CTSO-INFRA-001 | §8 |
| Reserved Instances for nodes | CTSO-INFRA-001 | §7 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| Network plugin | Azure CNI Overlay | Azure CNI powered by Cilium for production microservices | **Differs** — MS Learn recommends Cilium for eBPF-based performance; Contoso specifies CNI Overlay | [AKS Microservices Architecture](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices) |
| Private cluster | Required | Recommended for production to restrict API server access | **Aligns** | [AKS Baseline Architecture](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| Workload Identity | Managed Identities required | Workload Identity with Entra ID recommended over pod-managed identity | **Aligns** | [AKS Advanced Microservices](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices-advanced) |
| Image scanning | Trivy + Defender mandatory | Microsoft Defender for Containers + deployment safeguards recommended | **Aligns** | [AKS Advanced Microservices](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices-advanced) |
| Service mesh | Istio required for mTLS | Istio is an AKS-supported add-on; Cilium can also enforce network policies | **Aligns** | [AKS Service Mesh](https://learn.microsoft.com/azure/aks/servicemesh-about/) |
| Hub-spoke with Firewall | Required for all egress | Recommended; AKS baseline uses Azure Firewall for egress control | **Aligns** | [AKS with Azure Firewall](https://learn.microsoft.com/azure/architecture/guide/aks/aks-firewall) |
| Container image size | 500 MB max | No specific limit in MS Learn; distroless/minimal images recommended | **Exceeds** — Contoso enforces a stricter size limit | N/A |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | Azure CNI Overlay vs. Cilium — Contoso standard specifies Overlay, but MS Learn recommends Cilium for improved eBPF performance | Low | Evaluate with Cloud Platform Team whether to adopt Cilium; raise with ARB if needed |
| 2 | ML inference node pool sizing depends on credit scoring model complexity | Medium | Profile ML model during MVP; adjust D8s_v5 nodes as needed |
| 3 | Istio service mesh adds operational complexity | Low | Use AKS-managed Istio add-on to reduce management burden |
| 4 | Budget sensitivity with multi-zone + Reserved Instances | Medium | Lock 1-year RIs early; monitor with Azure Cost Management [CTSO-INFRA-001 §7] |
