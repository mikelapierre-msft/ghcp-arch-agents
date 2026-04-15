# Networking Design: Contoso Loan Application Platform

## Network Topology

The platform deploys into a **hub-spoke topology** within Contoso's existing Azure Landing Zone [CTSO-NET-001 §1]. The application workload resides in a dedicated **spoke VNet** peered to the regional hub VNet, which hosts shared services including Azure Firewall, Azure Bastion, and ExpressRoute/VPN gateways.

### VNet Design

| VNet | Purpose | Region | Address Space |
|---|---|---|---|
| Hub VNet (existing) | Shared services (Firewall, Bastion, gateways) | Canada Central | Managed by Network Engineering |
| Spoke VNet — Loan Platform | Application workloads | Canada Central | /16 (requested via IPAM portal) |
| DR Spoke VNet — Loan Platform | DR workloads | Canada East | /16 (requested via IPAM portal) |

Per [CTSO-NET-001 §2]: IP ranges are centrally managed via the IPAM portal. Self-assigned IP ranges are **prohibited**.

### Subnet Design

| Subnet | Purpose | CIDR | NSG | Notes |
|---|---|---|---|---|
| `snet-aks-system` | AKS system node pool | /26 | Yes | Min /26 for AKS per [CTSO-NET-001 §2] |
| `snet-aks-app` | AKS application node pool | /24 | Yes | Sized for auto-scale up to 8 nodes |
| `snet-aks-ml` | AKS ML inference node pool | /27 | Yes | Sized for 1-3 nodes |
| `snet-appgw` | Application Gateway (internal APIM ingress) | /27 | Yes | Regional WAF ingress |
| `snet-apim` | Azure API Management | /27 | Yes | Standard v2 VNet integration |
| `snet-privateendpoints` | Private Endpoints for PaaS services | /24 | Yes | SQL, Cosmos, Redis, Storage, Key Vault, Service Bus, Event Hubs, ACR |
| `snet-devops` | Azure DevOps / GitHub Actions agents | /27 | Yes | Self-hosted build agents |

### Network Flow Diagram

```
Internet Users
      │
      ▼
Azure Front Door (Global WAF + DDoS)
      │ Private Link
      ▼
Application Gateway (Regional WAF) ──or──► APIM (Standard v2)
      │ Internal LB                            │
      ▼                                        ▼
AKS Private Cluster (Spoke VNet)
      │ (mTLS via Istio)
      ├──► Private Endpoints ──► Azure SQL, Cosmos DB, Redis, Blob
      │
      ▼ (via Azure Firewall in Hub)
External Services (Equifax API, SendGrid, Twilio)
      │
Hub VNet ─── ExpressRoute ──► On-Premises Core Banking
```

## Ingress

### Azure Front Door (Global Layer)

- **Azure Front Door Premium** with WAF policy [CTSO-NET-001 §4], [CTSO-SEC-001 §3]
- OWASP 3.2 managed rule set in **Prevention mode**
- Custom rules for rate limiting and geo-filtering (Canada-only for customer portal)
- DDoS Protection Standard enabled [CTSO-SEC-001 §3]
- TLS 1.3 termination at the edge; re-encryption to backend [CTSO-NET-001 §4]
- Private Link connection to AKS ingress controller

### Azure API Management (API Gateway)

- **Standard v2** tier with VNet integration [CTSO-APP-001 §1]
- Internal mode (no public endpoint) [CTSO-SEC-001 §3]
- All external APIs published through APIM with:
  - OpenAPI 3.0+ specifications [CTSO-APP-001 §2]
  - Rate limiting and throttling [CTSO-APP-001 §2]
  - OAuth 2.0 token validation (Entra External ID for customers, Entra ID for advisors)
  - URL path versioning (e.g., `/api/v1/loans`) [CTSO-APP-001 §2]

### No Direct Public IPs

No public IP addresses assigned to any workload resources (VMs, AKS nodes, databases) [CTSO-NET-001 §4].

## Egress

- All outbound internet traffic routes through **Azure Firewall** in the hub VNet [CTSO-NET-001 §5]
- **Deny-all-by-default** with explicit FQDN allow lists [CTSO-NET-001 §5]
- Allowed outbound FQDNs:
  - `api.equifax.com` — Credit checks / identity verification
  - `api.sendgrid.com` — Email delivery
  - `api.twilio.com` — SMS delivery
  - `*.microsoft.com`, `*.azure.com` — Azure service dependencies
  - `mcr.microsoft.com` — Microsoft Container Registry
  - `*.docker.io` — Blocked (images re-hosted in Contoso ACR) [CTSO-SEC-001 §8]
- Azure Firewall logs sent to Log Analytics [CTSO-NET-001 §5]

## Private Connectivity

### Private Endpoints

All Azure PaaS services accessed exclusively via **Private Endpoints** [CTSO-NET-001 §6], [CTSO-SEC-001 §3]:

| Service | Private Endpoint | Private DNS Zone |
|---|---|---|
| Azure SQL Database | `pe-sql-loanplatform` | `privatelink.database.windows.net` |
| Azure Cosmos DB | `pe-cosmos-loanplatform` | `privatelink.documents.azure.com` |
| Azure Cache for Redis | `pe-redis-loanplatform` | `privatelink.redis.cache.windows.net` |
| Azure Key Vault | `pe-kv-loanplatform` | `privatelink.vaultcore.azure.net` |
| Azure Service Bus | `pe-sb-loanplatform` | `privatelink.servicebus.windows.net` |
| Azure Event Hubs | `pe-eh-loanplatform` | `privatelink.servicebus.windows.net` |
| Azure Container Registry | `pe-acr-loanplatform` | `privatelink.azurecr.io` |
| Azure Blob Storage | `pe-st-loanplatform` | `privatelink.blob.core.windows.net` |
| Azure AI Services | `pe-ai-loanplatform` | `privatelink.cognitiveservices.azure.com` |

### Private DNS Zones

- All Private DNS Zones linked to the hub VNet for centralized resolution [CTSO-NET-001 §3]
- Azure-provided DNS used; custom DNS servers prohibited [CTSO-NET-001 §3]

### On-Premises Connectivity

- **ExpressRoute** (primary) via private peering for Contoso Core Banking integration [CTSO-NET-001 §6]
- **VPN Gateway** (backup) for failover connectivity [CTSO-NET-001 §6]
- Traffic between AKS services and Core Banking traverses: AKS → Azure Firewall (hub) → ExpressRoute → On-premises

## Network Security Groups (NSGs)

- **Mandatory** on all subnets [CTSO-NET-001 §7]
- Default **deny inbound and outbound** rules [CTSO-NET-001 §7]
- **Application Security Groups (ASGs)** used over IP-based rules where possible [CTSO-NET-001 §7]
- NSG flow logs enabled with **90-day** retention [CTSO-NET-001 §7]
- Traffic Analytics enabled on NSG flow logs [CTSO-NET-001 §8]

### Key NSG Rules

| Subnet | Direction | Rule | Source | Destination | Port | Action |
|---|---|---|---|---|---|---|
| snet-aks-app | Inbound | Allow APIM | APIM subnet | AKS app subnet | 443 | Allow |
| snet-aks-app | Outbound | Allow PE | AKS app subnet | PE subnet | 443, 1433, 6380 | Allow |
| snet-aks-app | Outbound | Allow Firewall | AKS app subnet | Azure Firewall | 443 | Allow |
| snet-privateendpoints | Inbound | Allow AKS | AKS subnets | PE subnet | 443, 1433, 6380 | Allow |
| All | Inbound/Outbound | Deny All | Any | Any | Any | Deny |

## Network Monitoring

- **Azure Network Watcher** enabled in Canada Central and Canada East [CTSO-NET-001 §8]
- **Connection Monitor** configured for:
  - AKS → Azure SQL (Private Endpoint)
  - AKS → On-premises Core Banking (ExpressRoute)
  - AKS → Equifax API (Azure Firewall egress)
  - Canada Central → Canada East (cross-region DR path)
- **Traffic Analytics** on NSG flow logs [CTSO-NET-001 §8]
- Latency baselines established and monitored [CTSO-NET-001 §8]

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| Hub-spoke topology | CTSO-NET-001 | §1 |
| Centralized IPAM for IP ranges | CTSO-NET-001 | §2 |
| Azure Private DNS Zones | CTSO-NET-001 | §3 |
| Front Door with WAF for ingress | CTSO-NET-001 | §4 |
| No direct public IPs on workloads | CTSO-NET-001 | §4 |
| TLS termination + re-encryption | CTSO-NET-001 | §4 |
| All egress via Azure Firewall | CTSO-NET-001 | §5 |
| Deny-all-by-default egress | CTSO-NET-001 | §5 |
| Private Endpoints for all PaaS | CTSO-NET-001 | §6 |
| ExpressRoute primary, VPN backup | CTSO-NET-001 | §6 |
| NSG mandatory on all subnets | CTSO-NET-001 | §7 |
| NSG flow logs with 90-day retention | CTSO-NET-001 | §7 |
| Network Watcher and Traffic Analytics | CTSO-NET-001 | §8 |
| DDoS Protection Standard | CTSO-SEC-001 | §3 |
| WAF with OWASP 3.2 ruleset | CTSO-SEC-001 | §3 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| Hub-spoke topology | Required with Azure Firewall | Recommended as best practice; AKS baseline uses hub-spoke | **Aligns** | [Hub-Spoke Topology](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke) |
| Private cluster AKS | Required | Recommended for production; eliminates public API server | **Aligns** | [AKS Baseline](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/baseline-aks#network-topology) |
| Azure Front Door + WAF | Required for internet ingress | Recommended; prioritize Front Door WAF for global edge protection | **Aligns** | [Front Door to Secure AKS](https://learn.microsoft.com/azure/architecture/example-scenario/aks-front-door/aks-front-door) |
| APIM with AKS | Required for API gateway | Recommended pattern; APIM as turnkey gateway for microservices | **Aligns** | [APIM with AKS](https://learn.microsoft.com/azure/api-management/api-management-kubernetes) |
| Private Endpoints only | Service endpoints prohibited | Recommended over service endpoints for production | **Aligns** | [Private Link Deployment](https://learn.microsoft.com/azure/architecture/networking/guide/private-link-hub-spoke-network) |
| Azure Firewall for egress | Required | Recommended; AKS baseline uses Azure Firewall for egress | **Aligns** | [AKS with Azure Firewall](https://learn.microsoft.com/azure/architecture/guide/aks/aks-firewall) |
| DDoS Protection | Standard required | Recommended on all production VNets | **Aligns** | [Mission-Critical Networking](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-networking-connectivity) |
| NSG flow logs | 90-day retention | Recommended; no specific retention minimum in MS Learn | **Exceeds** — Contoso mandates 90-day retention | [NSG Flow Logs](https://learn.microsoft.com/azure/network-watcher/nsg-flow-logs-overview) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | IPAM portal request for VNet address space must be submitted early | Medium | Engage Network Engineering team during project kickoff |
| 2 | ExpressRoute circuit capacity for Core Banking integration | Low | Validate with Network Engineering; confirm bandwidth |
| 3 | Private DNS Zone conflicts if existing zones are in the hub | Low | Coordinate with Cloud Platform Team for DNS zone sharing |
| 4 | Azure Firewall cost (~$1,500/month) is significant portion of budget | Medium | Shared Firewall in hub VNet may already be provisioned by Landing Zone |
