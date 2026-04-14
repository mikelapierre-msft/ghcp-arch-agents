# Networking — Contoso Loan Application Platform

## Network Topology

The platform deploys into a **dedicated spoke VNet** peered to the Contoso regional hub VNet, following the hub-spoke topology [CTSO-NET-001 §1]. All traffic transits through Azure Firewall in the hub for inspection.

### Spoke VNet Design

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| VNet name | `ctso-loanapp-prod-vnet-cc-001` | [CTSO-INFRA-001 §2] |
| Address space | Allocated by IPAM team (e.g., `10.50.0.0/16`) | [CTSO-NET-001 §2] |
| Peering | Hub VNet (Canada Central) | [CTSO-NET-001 §1] |
| DDoS Protection | Standard (inherited from hub) | [CTSO-SEC-001 §3] |
| Network Watcher | Enabled | [CTSO-NET-001 §8] |

### Subnet Design

| Subnet | CIDR (example) | Purpose | NSG | Standard Reference |
|--------|---------------|---------|-----|-------------------|
| `snet-aks-nodes` | `/24` (min /26) | AKS node pool | Yes | [CTSO-NET-001 §2, §7] |
| `snet-aks-pods` | `/22` | AKS pod CIDR (CNI overlay) | Via Cilium policies | [CTSO-NET-001 §2] |
| `snet-apim` | `/27` | Azure API Management | Yes | [CTSO-NET-001 §7] |
| `snet-appgw` | `/27` | Application Gateway (regional ingress) | Yes | [CTSO-NET-001 §4] |
| `snet-private-endpoints` | `/27` | Private Endpoints for PaaS services | Yes | [CTSO-NET-001 §6] |
| `snet-devops-agents` | `/27` | Self-hosted build agents | Yes | [CTSO-NET-001 §7] |

All subnets have **NSGs with default deny** inbound and outbound rules [CTSO-NET-001 §7].

## Ingress Architecture

```
Internet → Azure Front Door (Global, WAF) → Application Gateway (Regional, WAF) → AKS Ingress Controller → Microservices
```

### Azure Front Door (Premium)

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | Premium | [CTSO-NET-001 §4] |
| WAF policy | OWASP 3.2 rule set (Prevention mode) | [CTSO-NET-001 §4], [CTSO-SEC-001 §3] |
| TLS | TLS 1.3 (minimum 1.2) | [CTSO-SEC-001 §1] |
| Custom domain | `loans.contoso.com` (customer), `advisor.contoso.com` (internal) | — |
| Origin | Application Gateway (Private Link origin) | [CTSO-NET-001 §6] |
| Rate limiting | Custom rules for API abuse prevention | [CTSO-APP-001 §2] |
| Caching | Static assets only (no PII caching) | Security |

### Application Gateway (WAF v2)

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | WAF_v2 | [CTSO-NET-001 §4] |
| Zone redundancy | Zones 1, 2, 3 | [CTSO-INFRA-001 §6] |
| WAF policy | OWASP 3.2 (Defense in depth) | [CTSO-SEC-001 §3] |
| TLS | End-to-end TLS (re-encrypt to backends) | [CTSO-NET-001 §4] |
| Backend pool | AKS internal load balancer | Private |
| Health probes | `/health/ready` endpoints | [CTSO-APP-001 §5] |

### APIM (Internal VNet Mode)

Azure API Management deployed in **internal VNet mode** behind Application Gateway:

| Feature | Configuration |
|---------|--------------|
| VNet mode | Internal | 
| Authentication | OAuth 2.0 / Entra ID token validation |
| Rate limiting | Per-subscription throttling |
| API specs | OpenAPI 3.0+ required for all APIs [CTSO-APP-001 §2] |
| Versioning | URL path versioning (`/api/v1/`) [CTSO-APP-001 §2] |

## Egress Architecture

All outbound internet traffic routes through **Azure Firewall** in the hub VNet [CTSO-NET-001 §5]:

| Destination | Protocol | Firewall Rule | Purpose |
|-------------|----------|--------------|---------|
| `api.equifax.ca` | HTTPS/443 | FQDN application rule | Credit checks |
| `api.sendgrid.com` | HTTPS/443 | FQDN application rule | Email delivery |
| `api.twilio.com` | HTTPS/443 | FQDN application rule | SMS delivery |
| `*.azurecr.io` | HTTPS/443 | FQDN application rule | Container image pulls |
| `*.microsoft.com` | HTTPS/443 | FQDN application rule | Azure SDK dependencies |
| All other outbound | Any | **Deny** | Default deny [CTSO-NET-001 §5] |

Azure Firewall logs sent to Log Analytics for audit [CTSO-NET-001 §5].

## Private Connectivity

### ExpressRoute (On-Premises)

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| Connection | Contoso ExpressRoute (private peering) | [CTSO-NET-001 §6] |
| Purpose | Core Banking integration (loan disbursement, account creation) | Requirement |
| Backup | VPN Gateway (S2S) | [CTSO-NET-001 §6] |
| Routing | UDR through hub Azure Firewall | [CTSO-NET-001 §1] |

### Private Endpoints

All PaaS services accessed via Private Endpoints [CTSO-NET-001 §6], [CTSO-SEC-001 §3]:

| Service | Private Endpoint Subnet | Private DNS Zone |
|---------|------------------------|------------------|
| Azure SQL Database | `snet-private-endpoints` | `privatelink.database.windows.net` |
| Azure Cosmos DB | `snet-private-endpoints` | `privatelink.documents.azure.com` |
| Azure Cache for Redis | `snet-private-endpoints` | `privatelink.redis.cache.windows.net` |
| Azure Service Bus | `snet-private-endpoints` | `privatelink.servicebus.windows.net` |
| Azure Key Vault | `snet-private-endpoints` | `privatelink.vaultcore.azure.net` |
| Azure Container Registry | `snet-private-endpoints` | `privatelink.azurecr.io` |
| Azure Storage (documents) | `snet-private-endpoints` | `privatelink.blob.core.windows.net` |
| Azure App Configuration | `snet-private-endpoints` | `privatelink.azconfig.io` |
| Azure Document Intelligence | `snet-private-endpoints` | `privatelink.cognitiveservices.azure.com` |

All Private DNS zones linked to the hub VNet for centralized resolution [CTSO-NET-001 §3].

## DNS

| Configuration | Value | Standard Reference |
|--------------|-------|-------------------|
| Internal DNS | Azure Private DNS Zones | [CTSO-NET-001 §3] |
| External DNS | Azure DNS (via Front Door) | [CTSO-NET-001 §3] |
| Custom DNS servers | **Prohibited** | [CTSO-NET-001 §3] |
| DNS zone linking | Linked to hub VNet | [CTSO-NET-001 §3] |

## Network Security Groups (NSGs)

NSG flow logs enabled on all subnets with **90-day retention** [CTSO-NET-001 §7]:

| NSG Rule (example: AKS subnet) | Direction | Source | Destination | Port | Action |
|--------------------------------|-----------|--------|-------------|------|--------|
| Allow-AppGw-to-AKS | Inbound | AppGw subnet | AKS subnet | 443 | Allow |
| Allow-APIM-to-AKS | Inbound | APIM subnet | AKS subnet | 443 | Allow |
| Deny-All-Inbound | Inbound | Any | Any | Any | Deny |
| Allow-AKS-to-PaaS | Outbound | AKS subnet | PE subnet | 443, 1433, 6380 | Allow |
| Allow-AKS-to-Firewall | Outbound | AKS subnet | Hub Firewall | Any | Allow |
| Deny-All-Outbound | Outbound | Any | Any | Any | Deny |

Application Security Groups (ASGs) used instead of IP-based rules where possible [CTSO-NET-001 §7].

## Network Monitoring

| Capability | Tool | Standard Reference |
|-----------|------|-------------------|
| Network Watcher | Enabled in Canada Central + Canada East | [CTSO-NET-001 §8] |
| Connection Monitor | Hub → Spoke, On-prem → AKS, Cross-region | [CTSO-NET-001 §8] |
| Traffic Analytics | Enabled on NSG flow logs | [CTSO-NET-001 §8] |
| Latency baselines | Established for Core Banking path | [CTSO-NET-001 §8] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| Hub-spoke topology with dedicated spoke VNet | CTSO-NET-001 | §1 |
| IP ranges from IPAM team | CTSO-NET-001 | §2 |
| Azure Private DNS Zones | CTSO-NET-001 | §3 |
| Front Door + App Gateway dual-layer ingress | CTSO-NET-001 | §4 |
| WAF with OWASP 3.2 rules | CTSO-NET-001 | §4, CTSO-SEC-001 §3 |
| End-to-end TLS with re-encryption | CTSO-NET-001 | §4 |
| Egress via Azure Firewall with FQDN filtering | CTSO-NET-001 | §5 |
| ExpressRoute for on-prem + VPN backup | CTSO-NET-001 | §6 |
| Private Endpoints for all PaaS services | CTSO-NET-001 | §6, CTSO-SEC-001 §3 |
| NSGs on all subnets with default deny | CTSO-NET-001 | §7 |
| NSG flow logs 90-day retention | CTSO-NET-001 | §7 |
| DDoS Protection Standard | CTSO-SEC-001 | §3 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Ingress | Front Door + App Gateway dual-layer | Front Door recommended for global; App Gateway for regional | **Exceeds** (defense-in-depth) | [Front Door + App Gateway](https://learn.microsoft.com/azure/architecture/example-scenario/gateway/firewall-application-gateway) |
| Egress | All traffic through Azure Firewall | Azure Firewall recommended for egress control | **Aligns** | [AKS egress with Firewall](https://learn.microsoft.com/azure/aks/limit-egress-traffic) |
| Private connectivity | Private Endpoints mandatory; service endpoints prohibited | Private Endpoints recommended; service endpoints acceptable | **Exceeds** | [Private Link best practices](https://learn.microsoft.com/azure/private-link/private-link-best-practices) |
| NSG flow logs | 90-day retention mandatory | Recommended for network monitoring | **Exceeds** (specific retention) | [NSG flow logs](https://learn.microsoft.com/azure/network-watcher/nsg-flow-logs-overview) |
| Hub-spoke topology | Mandatory with Azure Firewall transit | Recommended pattern for enterprise | **Aligns** | [Hub-spoke topology](https://learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke) |
| DNS | Azure Private DNS Zones only | Azure Private DNS recommended for Private Endpoints | **Aligns** | [Private DNS zones](https://learn.microsoft.com/azure/private-link/private-endpoint-dns) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| APIM internal VNet mode adds latency to API calls | Benchmark P95 latency; consider APIM stv2 platform for improved performance | Test in QA |
| Dual-layer WAF (Front Door + AppGw) may cause false positives | Tune WAF rules in Detection mode first; promote to Prevention after baseline | Pre-GA task |
| IP address space request from IPAM team may delay deployment | Submit IPAM request early; reserve /16 for future growth | **Action required** |
| ExpressRoute bandwidth for Core Banking integration | Validate current ExpressRoute circuit capacity with network team | **Action required** |
