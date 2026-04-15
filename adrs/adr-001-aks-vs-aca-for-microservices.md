# ADR-001: AKS vs Azure Container Apps for Loan Application Microservices

## Status
Accepted

## Date
2026-04-15

## Context

The Contoso Loan Application Platform is a customer-facing digital lending system enabling retail banking customers to apply for personal loans, auto loans, and mortgage pre-approvals. The platform follows a microservices architecture [CTSO-APP-001 §1] with services including loan intake, identity verification, credit scoring (ML-based), document processing, decision engine, notifications, advisor portal, and reporting.

We need to select the container compute platform for hosting these microservices. The decision is constrained by several factors:

- **Regulatory compliance**: The platform handles Confidential data (PII + financial data) subject to PIPEDA, OSFI, and AML regulations. All data must reside in Canadian Azure regions [CTSO-DATA-001 §3], [CTSO-SEC-001 §7].
- **Multi-region DR**: Production must achieve 99.95% SLA with active-passive DR across Canada Central and Canada East. RPO ≤ 30 min, RTO ≤ 2 hours [CTSO-INFRA-001 §6].
- **Service mesh requirement**: Mutual TLS (mTLS) is mandatory for all inter-service communication [CTSO-SEC-001 §4]. Istio is the approved service mesh [CTSO-APP-001 §3].
- **Network isolation**: All production workloads must use Private Endpoints with no public endpoints [CTSO-SEC-001 §3], [CTSO-NET-001 §6]. Traffic must transit through Azure Firewall [CTSO-NET-001 §1].
- **Scale**: Must support 10,000 concurrent users with auto-scaling.
- **Team capability**: The team has some Kubernetes experience but is not deeply specialized.
- **Budget**: ~$15K/month Azure production spend target.

## Decision Drivers

- **Contoso compute preference**: AKS is the #1 approved compute platform, followed by ACA [CTSO-INFRA-001 §3]
- **Service mesh requirement**: Istio is the mandated service mesh technology [CTSO-APP-001 §3]
- **mTLS enforcement**: All inter-service communication requires mutual TLS [CTSO-SEC-001 §4]
- **Multi-region DR**: Active-passive deployment with RPO ≤ 30 min, RTO ≤ 2 hours [CTSO-INFRA-001 §6]
- **Private cluster / private networking**: No public endpoints in production [CTSO-SEC-001 §3]
- **Hub-spoke network integration**: Traffic must transit Azure Firewall; spoke VNet per workload [CTSO-NET-001 §1]
- **IaC deployment**: All infrastructure via Bicep (preferred) or Terraform [CTSO-INFRA-001 §4]
- **Blue-green / canary deployments**: Big-bang deployments are prohibited [CTSO-APP-001 §6]
- **Observability**: OpenTelemetry SDK, Azure Monitor / Application Insights export [CTSO-APP-001 §5]
- **Team Kubernetes experience**: Moderate — some experience but not deep specialization
- **Cost**: Production budget ~$15K/month

## Considered Options

1. **Azure Kubernetes Service (AKS)** — Full managed Kubernetes with Istio service mesh add-on, private cluster mode, and multi-region active-passive DR
2. **Azure Container Apps (ACA)** — Serverless container platform with built-in Envoy-based networking, managed scaling, and simplified operations
3. **AKS Automatic** — AKS with automated cluster management (node provisioning, scaling, security configuration) for reduced operational burden

## Decision

**Option 1: Azure Kubernetes Service (AKS)** with the Istio-based service mesh add-on is selected as the compute platform for the Contoso Loan Application Platform.

### Rationale

AKS is chosen because it is the only option that fully satisfies all Contoso standards without exceptions or workarounds:

1. **Istio service mesh**: AKS is the only platform that supports the Istio service mesh, which is Contoso's mandated service mesh technology [CTSO-APP-001 §3]. ACA offers only built-in Envoy-based mTLS, which does not provide the full Istio feature set (authorization policies, fine-grained traffic management, fault injection, observability hooks).
2. **Multi-region active-passive DR**: AKS has a well-documented, Microsoft-recommended multi-region active-passive architecture using Azure Front Door with priority-based routing, hub-spoke networking per region, and Azure Firewall Manager for cross-region firewall policy management. ACA multi-region is achievable but requires manual redeployment patterns with less prescriptive Microsoft guidance.
3. **Private cluster mode**: AKS private cluster mode ensures the Kubernetes API server is not exposed publicly [CTSO-INFRA-001 §3]. ACA supports Private Endpoints for ingress but does not offer the same depth of API server isolation.
4. **Network integration**: AKS with Azure CNI Overlay integrates directly into Contoso's hub-spoke topology with subnet sizing of /26 for node pools [CTSO-NET-001 §2]. Traffic routing through Azure Firewall and NSG enforcement on subnets are natively supported.
5. **Deployment strategies**: AKS with Istio enables canary deployments and traffic splitting natively via VirtualService and DestinationRule CRDs [CTSO-APP-001 §6].
6. **Team readiness**: While the team has only moderate Kubernetes experience, the managed Istio add-on and AKS managed features (auto-upgrade, node auto-scaling, Workload Identity) significantly reduce operational complexity. The Contoso platform team can provide guardrails through Azure Policy and deployment safeguards.

## Options Analysis

### Option 1: Azure Kubernetes Service (AKS)

**Description**: Deploy microservices on AKS private clusters in Canada Central (primary) and Canada East (DR) with the managed Istio service mesh add-on, Azure CNI Overlay networking, Workload Identity, and Microsoft Defender for Containers.

- **Pros**:
  - #1 approved Contoso compute platform [CTSO-INFRA-001 §3]
  - Full Istio service mesh support via managed add-on — satisfies mTLS, traffic management, and observability requirements
  - Private cluster mode eliminates public API server exposure
  - Well-documented multi-region active-passive DR architecture with Azure Front Door, Application Gateway, and hub-spoke per region
  - Azure CNI Overlay integrates directly with Contoso hub-spoke VNet topology
  - Full Kubernetes API access for advanced workload scenarios (ML inference node pools, CronJobs, DaemonSets)
  - Supports canary/blue-green deployments via Istio traffic splitting
  - Workload Identity Federation for managed identity integration with Entra ID
  - Mature ecosystem of security tooling (Defender for Containers, Azure Policy for AKS, OPA Gatekeeper)
  - Node auto-scaling and cluster auto-upgrade reduce operational burden

- **Cons**:
  - Higher operational complexity compared to ACA — requires Kubernetes knowledge for cluster management, upgrades, and troubleshooting
  - Dedicated compute costs (VMs always running) vs. ACA consumption-based billing
  - Team has moderate Kubernetes experience, which may slow initial velocity
  - Requires managing node pool sizing, OS patching cadence, and Kubernetes version upgrades
  - More IaC complexity (cluster config, node pools, Istio mesh config, network policies)

- **Contoso Compliance**: ✅ Fully compliant with all standards
  - [CTSO-INFRA-001 §3]: AKS is #1 approved platform; private cluster + Azure CNI Overlay + managed identity ✅
  - [CTSO-APP-001 §3]: Istio service mesh is the approved technology ✅
  - [CTSO-SEC-001 §4]: mTLS enforced via Istio PeerAuthentication ✅
  - [CTSO-NET-001 §1]: Hub-spoke with Azure Firewall transit ✅
  - [CTSO-NET-001 §2]: /26 subnets for AKS node pools ✅
  - [CTSO-INFRA-001 §6]: Multi-region active-passive DR supported ✅

- **Industry Alignment**: Aligns with Microsoft's recommended AKS baseline architecture and multi-region active-passive DR pattern. See [AKS active-passive disaster recovery](https://learn.microsoft.com/azure/aks/active-passive-solution) and [Choose an Azure compute option for microservices](https://learn.microsoft.com/azure/architecture/microservices/design/compute-options).

- **Cost Estimate**: ~$8K–12K/month for compute (2 clusters: primary 3-8 nodes + DR 3 nodes standby, Standard_D4s_v5) plus supporting services. Fits within the $15K/month production budget.

### Option 2: Azure Container Apps (ACA)

**Description**: Deploy microservices on ACA workload profile environments in Canada Central and Canada East with built-in Envoy-based mTLS, managed scaling, and Private Endpoints.

- **Pros**:
  - Simplified operations — no Kubernetes cluster management, patching, or version upgrades
  - Built-in auto-scaling including scale-to-zero for non-critical services, reducing costs
  - Built-in Envoy-based mTLS for inter-service communication within an environment
  - Supports Private Endpoints (workload profile environments) and VNet integration
  - Traffic splitting for revision-based canary deployments
  - Lower barrier to entry for the team given moderate Kubernetes experience
  - Consumption-based billing can be cost-effective for variable workloads
  - Managed Dapr integration for service-to-service invocation and pub/sub

- **Cons**:
  - **Does not support Istio** — only built-in Envoy-based mTLS. This violates [CTSO-APP-001 §3] which mandates Istio as the approved service mesh
  - Limited traffic management compared to Istio (no VirtualService, DestinationRule, fault injection, circuit breaking at mesh level)
  - mTLS scope is limited to within a single Container Apps environment — no cross-environment mTLS
  - Multi-region DR requires deploying separate environments with manual configuration; less prescriptive Microsoft guidance than AKS
  - Single security boundary per environment — no namespace-level isolation for multi-tenant scenarios
  - No direct Kubernetes API access, limiting advanced workload patterns (DaemonSets, custom operators, ML node pools with GPU taints)
  - Environment-level logging (single Log Analytics workspace) limits per-service isolation
  - Less mature ecosystem for enterprise security controls compared to AKS

- **Contoso Compliance**: ⚠️ Non-compliant on service mesh requirement
  - [CTSO-INFRA-001 §3]: ACA is #2 approved platform ✅
  - [CTSO-APP-001 §3]: **Istio is not supported** — built-in Envoy mTLS only ❌
  - [CTSO-SEC-001 §4]: mTLS available (built-in Envoy) but not via approved Istio mesh ⚠️
  - [CTSO-NET-001 §1]: VNet integration supported but less granular than AKS ⚠️
  - [CTSO-NET-001 §6]: Private Endpoints supported ✅
  - [CTSO-INFRA-001 §6]: Multi-region DR achievable but less prescriptive ⚠️

- **Industry Alignment**: Microsoft recommends ACA for teams that "don't require direct access to Kubernetes APIs" and prefer operational simplicity. However, Microsoft documentation notes that AKS is preferred when service mesh and full Kubernetes API control are needed. See [Comparing AKS with other container options](https://learn.microsoft.com/azure/aks/compare-container-options-with-aks) and [Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service).

- **Cost Estimate**: ~$5K–8K/month for compute (consumption + dedicated workload profiles, 2 regions). Lower than AKS but the service mesh non-compliance requires an ARB exception that introduces governance risk.

### Option 3: AKS Automatic

**Description**: Deploy on AKS Automatic, which automates node provisioning, scaling, security configuration, and upgrades while still providing access to Kubernetes APIs and Istio support.

- **Pros**:
  - Reduces operational burden vs. standard AKS — automated node management, OS patching, and scaling
  - Retains full Kubernetes API access and Istio service mesh add-on support
  - Better fit for teams with moderate Kubernetes experience — "managed AKS" experience
  - Supports all the same security and networking features as standard AKS (private cluster, Azure CNI, Workload Identity)
  - Deployment safeguards and Azure Policy enforced by default

- **Cons**:
  - **Newer offering** — less production track record and community knowledge compared to standard AKS
  - Less granular control over node pool configuration (automated node provisioning may not align with specific sizing requirements for ML workloads)
  - Custom node pool taints and dedicated GPU node pools may be limited
  - The team must still understand Kubernetes concepts for troubleshooting and application deployment
  - Multi-region DR patterns are the same as standard AKS, requiring the same operational setup

- **Contoso Compliance**: ✅ Fully compliant with all standards (same as standard AKS)
  - [CTSO-INFRA-001 §3]: AKS is #1 approved platform; private cluster mode supported ✅
  - [CTSO-APP-001 §3]: Istio service mesh add-on supported ✅
  - [CTSO-SEC-001 §4]: mTLS via Istio ✅
  - [CTSO-NET-001 §1]: Hub-spoke with Azure Firewall ✅
  - [CTSO-INFRA-001 §6]: Multi-region active-passive DR ✅

- **Industry Alignment**: AKS Automatic is Microsoft's recommended path for teams seeking reduced operational overhead while retaining Kubernetes capabilities. See [AKS Automatic overview](https://learn.microsoft.com/azure/aks/intro-aks-automatic).

- **Cost Estimate**: ~$8K–12K/month (similar to standard AKS; node provisioning automation may optimize slightly through right-sizing).

## Consequences

### Positive
- Full compliance with all Contoso standards — no ARB exceptions required
- Istio service mesh provides enterprise-grade mTLS, traffic management, and observability across all microservices
- Well-documented multi-region active-passive DR pattern with Azure Front Door ensures business continuity
- AKS private cluster with hub-spoke integration provides defense-in-depth network security
- Kubernetes ecosystem provides flexibility for future workload patterns (ML inference, batch processing, CronJobs)
- Workload Identity Federation eliminates credential management for service-to-Azure-service authentication

### Negative
- Higher operational complexity than ACA — the team needs to build Kubernetes proficiency
- Dedicated VM costs for DR cluster even when passive (nodes running at minimum scale)
- Cluster upgrades require planning and testing (Kubernetes version + Istio version coordination)
- IaC templates for AKS are more complex than ACA (cluster config, node pools, mesh config, network policies, RBAC)
- Initial onboarding period may be longer; recommend allocating 2-4 weeks for team training on AKS + Istio operations

### Neutral
- The team's moderate Kubernetes experience is sufficient for AKS with managed add-ons, but investment in Kubernetes training is advisable
- AKS Automatic (Option 3) may be reconsidered in the future as it matures, potentially reducing operational overhead further
- If Contoso standards evolve to accept alternative service mesh technologies (e.g., Envoy-based mesh in ACA), ACA could be revisited

## Contoso Standards Compliance

| Standard | Section | Status | Notes |
|----------|---------|--------|-------|
| CTSO-INFRA-001 | §3 - Compute | ✅ Compliant | AKS is #1 approved platform; private cluster, Azure CNI Overlay, managed identity all configured |
| CTSO-INFRA-001 | §4 - IaC | ✅ Compliant | Bicep templates for cluster provisioning; stored in repo, deployed via CI/CD |
| CTSO-INFRA-001 | §6 - HA/DR | ✅ Compliant | Multi-zone in Canada Central + active-passive DR to Canada East; RPO ≤ 30 min, RTO ≤ 2 hours |
| CTSO-INFRA-001 | §8 - Prohibited | ✅ Compliant | Cluster auto-scaler enabled on all node pools |
| CTSO-APP-001 | §1 - Architecture | ✅ Compliant | Microservices with DDD, event-driven via Service Bus/Event Hubs |
| CTSO-APP-001 | §3 - Tech Stack | ✅ Compliant | **Istio service mesh on AKS** is the approved service mesh |
| CTSO-APP-001 | §4 - Containerization | ✅ Compliant | Contoso ACR base images, Trivy + Defender scanning, immutable SHA tags |
| CTSO-APP-001 | §5 - Observability | ✅ Compliant | OpenTelemetry SDK with OTLP export to Azure Monitor; Istio telemetry integration |
| CTSO-APP-001 | §6 - CI/CD | ✅ Compliant | Blue-green/canary via Istio traffic splitting; pipeline gates enforced |
| CTSO-SEC-001 | §3 - Network Security | ✅ Compliant | Private Endpoints, Azure Firewall egress, WAF on ingress, DDoS Protection |
| CTSO-SEC-001 | §4 - AuthN/AuthZ | ✅ Compliant | mTLS via Istio; Managed Identities for service-to-service; Workload Identity |
| CTSO-SEC-001 | §5 - App Security | ✅ Compliant | Defender for Containers, SAST/DAST/SCA in pipeline, no Critical/High CVEs |
| CTSO-SEC-001 | §7 - Compliance | ✅ Compliant | Canadian regions only; PIPEDA, PCI-DSS, SOC 2 alignment |
| CTSO-NET-001 | §1 - Topology | ✅ Compliant | Hub-spoke with Azure Firewall; dedicated spoke VNet per workload |
| CTSO-NET-001 | §2 - IPAM | ✅ Compliant | /26 subnets for AKS node pools via centralized IPAM |
| CTSO-NET-001 | §4 - Ingress | ✅ Compliant | Azure Front Door (global) + Application Gateway (regional) with WAF |
| CTSO-NET-001 | §5 - Egress | ✅ Compliant | All egress through Azure Firewall with FQDN filtering |
| CTSO-NET-001 | §7 - NSGs | ✅ Compliant | NSGs on all subnets; NSG flow logs enabled (90-day retention) |
| CTSO-IAM-001 | §5 - Service Identities | ✅ Compliant | System-assigned Managed Identities; Workload Identity Federation for pods |
| CTSO-DATA-001 | §3 - Data Residency | ✅ Compliant | All compute in Canada Central + Canada East |
| CTSO-DATA-001 | §4 - DB Security | ✅ Compliant | Entra ID auth via Managed Identity; Private Endpoints for all data services |

## Industry Best Practices References

| Source | Recommendation | Alignment | URL |
|--------|---------------|-----------|-----|
| Azure Architecture Center | Use AKS for microservices requiring direct Kubernetes API access and service mesh | ✅ Aligned — AKS selected for Istio and Kubernetes API needs | https://learn.microsoft.com/azure/architecture/microservices/design/compute-options |
| Azure Architecture Center | Choose AKS when service mesh, multi-tenant isolation, or advanced traffic management is needed | ✅ Aligned — Istio provides all required capabilities | https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service |
| AKS Documentation | Active-passive DR with Azure Front Door, hub-spoke per region, Azure Firewall Manager | ✅ Aligned — matches Contoso DR and networking requirements | https://learn.microsoft.com/azure/aks/active-passive-solution |
| AKS Documentation | Multi-region deployment models (active-active, active-passive, passive-cold) | ✅ Aligned — active-passive selected per Contoso DR strategy | https://learn.microsoft.com/azure/aks/reliability-multi-region-deployment-models |
| AKS Documentation | Istio-based service mesh add-on for mTLS, traffic management, observability | ✅ Aligned — managed Istio add-on reduces operational burden | https://learn.microsoft.com/azure/aks/istio-about |
| Azure Architecture Center | AKS provides most configurability but requires more operational effort; ACA suits teams focused on feature development | ✅ Acknowledged — team training investment planned | https://learn.microsoft.com/azure/architecture/guide/container-service-general-considerations |
| Azure WAF | Reliability: use availability zones and Standard/Premium tier for AKS clusters | ✅ Aligned — multi-zone deployment in Canada Central | https://learn.microsoft.com/azure/well-architected/reliability/checklist |
| Azure WAF | Security: AKS security baseline architecture for securing the compute platform | ✅ Aligned — Defender for Containers, Azure Policy, private cluster | https://learn.microsoft.com/security/benchmark/azure/baselines/azure-kubernetes-service-aks-security-baseline |
| Contoso Standards | AKS is #1 preferred compute; Istio is approved service mesh; ACA is #2 but lacks Istio | ✅ Exceeds industry defaults — Contoso requires Istio specifically, not just any mTLS | Standards: CTSO-INFRA-001, CTSO-APP-001 |

## Related Decisions

- **Networking topology**: Hub-spoke with Azure Firewall (documented in [architectures/contoso-loan-platform/networking.md](../architectures/contoso-loan-platform/networking.md))
- **Data platform**: Azure SQL + Cosmos DB + Redis with Private Endpoints (see [architectures/contoso-loan-platform/data.md](../architectures/contoso-loan-platform/data.md))
- **Identity**: Entra ID + Workload Identity Federation (see [architectures/contoso-loan-platform/identity.md](../architectures/contoso-loan-platform/identity.md))
- **Observability**: OpenTelemetry + Azure Monitor + Application Insights (see [architectures/contoso-loan-platform/observability.md](../architectures/contoso-loan-platform/observability.md))
- **CI/CD**: Pipeline design with canary deployments via Istio (see [architectures/contoso-loan-platform/cicd.md](../architectures/contoso-loan-platform/cicd.md))
