# Contoso Networking Standards

**Document ID:** CTSO-NET-001  
**Version:** 2.5  
**Last Updated:** 2026-01-20  
**Owner:** Contoso Network Engineering  
**Classification:** Internal

## 1. Network Topology

- **Hub-spoke** topology using Azure Virtual WAN (preferred) or traditional hub-spoke with Azure Firewall.
- Each workload gets a dedicated **spoke VNet** peered to the regional hub.
- Hub VNet hosts shared services: Azure Firewall, DNS, VPN/ExpressRoute gateways.
- All inter-VNet traffic must transit through **Azure Firewall** for inspection.

## 2. IP Address Management

- IPAM is centrally managed by the Network Engineering team. Self-assigned IP ranges are **prohibited**.
- VNet address spaces must be requested via the IPAM portal (minimum /24, maximum /16).
- RFC 1918 ranges only. Overlapping address spaces are **prohibited** across all VNets.
- Subnet sizing: minimum /27 for application subnets, /26 for AKS node pools.

## 3. DNS

- **Azure Private DNS Zones** for all internal name resolution.
- Private DNS zones must be linked to the hub VNet for centralized resolution.
- Custom DNS servers are **prohibited**. Use Azure-provided DNS with Private DNS Zones.
- External DNS managed via **Azure DNS** or **Azure Front Door** for global load balancing.

## 4. Ingress

- Internet-facing traffic must flow through **Azure Front Door** (global) or **Azure Application Gateway** (regional).
- Azure WAF policy required on all ingress points. OWASP 3.2 rule set minimum.
- No direct public IP on workload resources (VMs, AKS nodes, databases).
- TLS termination at the ingress controller; re-encryption to backends required (end-to-end TLS).

## 5. Egress

- All outbound internet traffic must route through **Azure Firewall** with FQDN filtering.
- Direct internet access from workloads is **prohibited**.
- Egress rules must follow **deny-all-by-default** with explicit allow lists.
- Azure Firewall logs must be sent to Log Analytics for audit.

## 6. Private Connectivity

- **ExpressRoute** (primary) and **VPN Gateway** (backup) for on-premises connectivity.
- ExpressRoute circuits must use **private peering** only. Microsoft peering requires approval.
- All Azure PaaS services must be accessed via **Private Endpoints** — public service endpoints are **prohibited** in production.
- Private Link DNS integration must use centralized Private DNS Zones in the hub.

## 7. Network Security Groups (NSGs)

- NSGs are **mandatory** on all subnets (except those managed by Azure, e.g., AzureFirewallSubnet).
- Default deny inbound and outbound rules must be present.
- NSG flow logs must be enabled and stored for **90 days** minimum.
- Application Security Groups (ASGs) preferred over IP-based rules.

## 8. Network Monitoring

- **Azure Network Watcher** must be enabled in all regions.
- Connection Monitor for critical paths (on-prem to Azure, cross-region).
- Traffic Analytics must be enabled on NSG flow logs.
- Latency baselines must be established and monitored for SLA compliance.

## 9. Prohibited Practices

- Assigning public IPs directly to VMs or AKS nodes.
- Using service endpoints instead of Private Endpoints in production.
- Peering VNets without going through the hub (direct spoke-to-spoke peering).
- Opening broad port ranges (e.g., 0-65535) in NSG rules.
- Using UDRs that bypass Azure Firewall for internet-bound traffic.
