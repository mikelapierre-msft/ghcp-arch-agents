# ADR-002: Container Platform Selection — AKS vs Azure Container Apps

## Status

Proposed

## Date

2026-04-14

## Context

The Contoso Loan Application Platform requires a container hosting platform for its microservices architecture. The platform processes loan applications, customer profiles, credit decisioning, and document management — all classified as **Confidential** (PII + financial data) under [CTSO-DATA-001 §2].

Key architectural requirements:
- **Microservices** hosting for the loan application domain services [CTSO-APP-001 §1]
- **Multi-region disaster recovery**: Active-Passive deployment across Canada Central (primary) and Canada East (DR) with RPO ≤ 1 hour and RTO ≤ 4 hours [CTSO-INFRA-001 §6]
- **Service mesh** with mutual TLS (mTLS) for all inter-service communication [CTSO-SEC-001 §4], with **Istio** as the approved service mesh technology [CTSO-APP-001 §3]
- **Private cluster** mode — no public endpoints in production [CTSO-SEC-001 §3]
- **99.95% SLA** minimum for production [CTSO-INFRA-001 §6]
- **Blue-green or canary deployments** [CTSO-APP-001 §6]
- **Network policy** enforcement with deny-all-by-default [CTSO-NET-001 §7]
- Egress filtering through Azure Firewall [CTSO-NET-001 §5]
- Container images from Contoso ACR with vulnerability scanning [CTSO-APP-001 §4]
- Team has **moderate Kubernetes experience** — not new to K8s, but not deep experts

## Decision Drivers

- Contoso requirement for **Istio service mesh** [CTSO-APP-001 §3] and **mTLS everywhere** [CTSO-SEC-001 §4]
- **Multi-region Active-Passive DR** with automated failover [CTSO-INFRA-001 §6]
- **Private cluster** with no public endpoints [CTSO-SEC-001 §3]
- Hub-spoke network integration with Azure Firewall egress [CTSO-NET-001 §1, §5]
- Kubernetes **network policies** for deny-all-by-default micro-segmentation [CTSO-NET-001 §7]
- Team operational readiness and Kubernetes familiarity
- Production SLA of **99.95%** [CTSO-INFRA-001 §6]

## Considered Options

1. **Azure Kubernetes Service (AKS)** — fully managed Kubernetes with Istio add-on
2. **Azure Container Apps (ACA)** — serverless container platform built on Kubernetes

## Decision

**Option 1: Azure Kubernetes Service (AKS)** is the recommended platform.

AKS is the only option that satisfies all Contoso standards simultaneously — specifically the **Istio service mesh requirement** [CTSO-APP-001 §3], **Kubernetes network policies** for deny-all-by-default micro-segmentation [CTSO-NET-001 §7], and **mature multi-region DR** patterns [CTSO-INFRA-001 §6]. Azure Container Apps does not support Istio and lacks the network segmentation controls Contoso requires.

## Options Analysis

### Option 1: Azure Kubernetes Service (AKS)

- **Pros**:
  - **Istio service mesh add-on**: AKS provides a [Microsoft-managed Istio add-on](https://learn.microsoft.com/azure/aks/istio-about) with mTLS, traffic management, observability, and fine-grained routing — directly satisfying [CTSO-APP-001 §3] and [CTSO-SEC-001 §4]
  - **Multi-region DR (proven patterns)**: Microsoft documents [active-passive](https://learn.microsoft.com/azure/aks/active-passive-solution) and [active-active](https://learn.microsoft.com/azure/aks/active-active-solution) multi-region deployment models with Azure Front Door, hub-spoke pairs, and Azure Firewall per region — aligning with [CTSO-INFRA-001 §6] and [CTSO-NET-001 §1]
  - **Private cluster mode**: Fully supports private API server with no public endpoint [CTSO-SEC-001 §3]
  - **Azure CNI overlay networking**: Required by Contoso [CTSO-INFRA-001 §3]; enables Kubernetes network policies for deny-all-by-default micro-segmentation [CTSO-NET-001 §7]
  - **Managed identity and Entra ID integration**: Native support for workload identity, managed identity for the cluster, and Entra ID RBAC [CTSO-IAM-001 §5]
  - **Availability zone support**: Nodes spread across zones for in-region HA [CTSO-INFRA-001 §6]
  - **Cluster autoscaler and node auto-scaling**: Required by Contoso [CTSO-INFRA-001 §8]
  - **Blue-green/canary via Istio traffic splitting**: Native support for advanced deployment strategies [CTSO-APP-001 §6]
  - **First in Contoso's approved compute platform list** [CTSO-INFRA-001 §3]
  - **Azure Policy and Microsoft Defender for Containers integration**: Enforces guardrails and scanning [CTSO-SEC-001 §5, §8]
  - **AKS Automatic** option available to reduce operational burden for teams with moderate K8s experience

- **Cons**:
  - **Higher operational complexity**: Cluster upgrades, node pool management, and Kubernetes API surface require ongoing operational investment
  - **Steeper learning curve**: Team's moderate K8s experience means initial ramp-up; mitigated by AKS Automatic and managed add-ons
  - **Higher base cost**: Requires dedicated VM-based node pools even at low traffic; Reserved Instances reduce this per [CTSO-INFRA-001 §7]
  - **Istio add-on limitations**: Does not yet support multi-cluster Istio mesh or Ambient mode (sidecar-less); per-cluster Istio is sufficient for active-passive DR

- **Contoso Compliance**: ✅ **Fully compliant** with all applicable standards
  - CTSO-INFRA-001 §3 (preferred compute), §6 (multi-zone, multi-region DR)
  - CTSO-APP-001 §3 (Istio service mesh), §4 (container scanning), §6 (blue-green/canary)
  - CTSO-SEC-001 §3 (private endpoints), §4 (mTLS), §5 (SAST/DAST/container scan), §8 (Defender)
  - CTSO-NET-001 §1 (hub-spoke), §4 (ingress via Front Door/AppGW), §5 (egress via Firewall), §7 (NSGs + network policies)
  - CTSO-IAM-001 §5 (managed identities)

- **Industry Alignment**: AKS is Microsoft's recommended platform for enterprise microservices requiring full Kubernetes control, service mesh, and multi-region DR. See [AKS baseline for multiregion clusters](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-multi-region/aks-multi-cluster) and [Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service).

- **Cost Estimate**: ~$2,500–$4,000/month per region for a production-grade cluster (3–5 D4s_v5 nodes with autoscaler), plus ~$1,000–$1,500/month for a scaled-down DR cluster in Canada East. Reserved Instances reduce by ~30%.

---

### Option 2: Azure Container Apps (ACA)

- **Pros**:
  - **Lower operational overhead**: Fully managed, no cluster upgrades or node pool management; Azure handles infrastructure
  - **Simpler developer experience**: Teams focus on application code, not Kubernetes internals
  - **Built-in scaling**: KEDA-based autoscaling including scale-to-zero for non-production
  - **Lower cost at low scale**: Consumption-based pricing avoids paying for idle nodes
  - **Dapr integration**: Built-in service discovery and pub/sub (though not Istio)
  - **Blue-green deployments**: Native revision management with traffic splitting [CTSO-APP-001 §6]
  - **Second in Contoso's approved compute platform list** [CTSO-INFRA-001 §3]

- **Cons**:
  - ❌ **No Istio support**: ACA uses Envoy-based internal networking and Dapr — **not Istio**. This is a **non-compliant deviation** from [CTSO-APP-001 §3] which mandates Istio for service mesh on container platforms
  - ❌ **Limited network segmentation**: ACA environments share a single security boundary with no Kubernetes network policies. Intra-environment communication is unrestricted, violating deny-all-by-default requirements [CTSO-NET-001 §7]
  - ⚠️ **Multi-region DR is custom-built**: ACA is a [single-region service](https://learn.microsoft.com/azure/reliability/reliability-container-apps#resilience-to-region-wide-failures). Multi-region requires manual deployment of separate environments + Azure Front Door routing. No documented active-passive pattern comparable to AKS. Achieving RPO ≤ 1 hour / RTO ≤ 4 hours requires significant custom engineering [CTSO-INFRA-001 §6]
  - ⚠️ **mTLS limitations**: ACA provides Envoy-based encryption in transit between apps in the same environment, but does not offer the fine-grained mTLS policy control (PeerAuthentication, AuthorizationPolicy) that Istio provides [CTSO-SEC-001 §4]
  - ⚠️ **No direct Kubernetes API access**: Cannot implement CNCF ecosystem tooling (Flux for GitOps, OPA/Gatekeeper for policy, Velero for backup)
  - ⚠️ **Fewer hardware SKU options**: Limited compute profiles compared to AKS node pool flexibility

- **Contoso Compliance**: ❌ **Non-compliant** on critical requirements
  - ❌ CTSO-APP-001 §3 — Istio not available
  - ❌ CTSO-NET-001 §7 — No Kubernetes network policies / deny-all-by-default
  - ⚠️ CTSO-INFRA-001 §6 — No native multi-region DR pattern
  - ⚠️ CTSO-SEC-001 §4 — Limited mTLS policy control

- **Industry Alignment**: Microsoft recommends ACA for teams that "don't require CNCF applications, have less experience in operations, or prefer to focus on application features" ([Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/container-service-general-considerations#conclusion)). ACA is a strong fit for simpler microservices that don't need service mesh, network policies, or multi-region HA/DR.

- **Cost Estimate**: ~$800–$2,000/month per region at comparable workload scale (consumption-based). Lower than AKS but does not include the cost of custom DR engineering.

## Consequences

### Positive
- Full compliance with all six Contoso standards domains — no waivers or ARB exceptions required
- Istio service mesh delivers mTLS, traffic management, fault injection, and canary routing out of the box
- Proven multi-region active-passive architecture with Azure Front Door and hub-spoke topology
- AKS Automatic reduces operational burden while preserving Kubernetes flexibility
- Team's existing Kubernetes experience is directly leveraged and deepened

### Negative
- Higher operational responsibility vs. ACA: cluster upgrades, node OS patching (mitigated by auto-upgrade channels), Istio revision management
- Higher baseline infrastructure cost (~2–3× ACA at equivalent scale)
- Team will need to invest in Kubernetes operational skills (Istio configuration, network policies, GitOps with Flux)

### Neutral
- Both AKS and ACA integrate with the same Azure ecosystem (Key Vault, ACR, Entra ID, Azure Monitor, Defender)
- If Contoso's standards evolve to accept Dapr as an alternative to Istio, ACA could become viable for future workloads

## Contoso Standards Compliance

| Standard | Section | Status | Notes |
|----------|---------|--------|-------|
| CTSO-INFRA-001 | §3 Compute | ✅ Compliant | AKS is first-preference compute platform |
| CTSO-INFRA-001 | §6 HA/DR | ✅ Compliant | Multi-zone + Active-Passive multi-region with RPO ≤ 1h / RTO ≤ 4h |
| CTSO-INFRA-001 | §8 Prohibited | ✅ Compliant | Node auto-scaling enabled via cluster autoscaler |
| CTSO-APP-001 | §1 Architecture | ✅ Compliant | Microservices with DDD boundaries |
| CTSO-APP-001 | §3 Service Mesh | ✅ Compliant | Istio via AKS managed add-on |
| CTSO-APP-001 | §4 Containerization | ✅ Compliant | ACR + Trivy + Defender scanning; immutable tags |
| CTSO-APP-001 | §6 CI/CD | ✅ Compliant | Blue-green/canary via Istio traffic splitting |
| CTSO-SEC-001 | §1 Encryption | ✅ Compliant | TLS 1.2+ in transit; AES-256 at rest |
| CTSO-SEC-001 | §3 Network Security | ✅ Compliant | Private cluster; no public endpoints |
| CTSO-SEC-001 | §4 mTLS | ✅ Compliant | Istio strict mTLS for all inter-service communication |
| CTSO-SEC-001 | §5 App Security | ✅ Compliant | Container scanning + SAST/DAST in pipeline |
| CTSO-SEC-001 | §8 Defender | ✅ Compliant | Microsoft Defender for Containers enabled |
| CTSO-NET-001 | §1 Topology | ✅ Compliant | Hub-spoke with Azure Firewall per region |
| CTSO-NET-001 | §4 Ingress | ✅ Compliant | Azure Front Door + Application Gateway + WAF |
| CTSO-NET-001 | §5 Egress | ✅ Compliant | All egress through Azure Firewall with FQDN filtering |
| CTSO-NET-001 | §6 Private Endpoints | ✅ Compliant | All PaaS accessed via Private Endpoints |
| CTSO-NET-001 | §7 NSGs | ✅ Compliant | NSGs on all subnets + Kubernetes network policies |
| CTSO-IAM-001 | §5 Service Identity | ✅ Compliant | Managed identities + workload identity federation |

## Industry Best Practices References

| Source | Recommendation | Alignment | URL |
|--------|---------------|-----------|-----|
| Azure Architecture Center | Choose an Azure container service — AKS for teams needing full K8s control, service mesh, network policies | ✅ Aligned | https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service |
| Azure Architecture Center | AKS baseline for multiregion clusters — active-passive with Front Door and hub-spoke | ✅ Aligned | https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-multi-region/aks-multi-cluster |
| AKS Documentation | Active-passive DR solution with Azure Front Door, paired regions, geo-replicated ACR | ✅ Aligned | https://learn.microsoft.com/azure/aks/active-passive-solution |
| AKS Documentation | Multi-region deployment models — active-active, active-passive, passive-cold | ✅ Aligned | https://learn.microsoft.com/azure/aks/reliability-multi-region-deployment-models |
| AKS Documentation | Istio-based service mesh add-on — managed mTLS, traffic management, observability | ✅ Aligned | https://learn.microsoft.com/azure/aks/istio-about |
| Azure WAF | Mission-critical application platform — AKS as first-choice for container orchestration | ✅ Aligned | https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-application-platform |
| Azure Reliability | Container Apps is single-region; custom multi-region required | Confirms ACA gap | https://learn.microsoft.com/azure/reliability/reliability-container-apps |

## Related Decisions

- ADR-001: Database Platform Selection for Loan Application Platform
- ADR-003 (pending): Ingress Architecture — Azure Front Door vs Application Gateway
- ADR-004 (pending): GitOps Strategy — Flux vs ArgoCD for AKS
