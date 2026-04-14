---
description: "Use when discussing network architecture, hub-spoke topology, Azure Virtual WAN, VNet peering, subnets, IP addressing, DNS, Azure Firewall, NSG, ingress (Front Door, Application Gateway, WAF), egress filtering, Private Endpoints, Private Link, ExpressRoute, VPN Gateway, network security groups, or traffic routing."
---
# Contoso Networking Standards

Read the full Contoso networking standards before making recommendations:

- [Networking Standards — CTSO-NET-001](../../../standards/networking.md)

## Key Policies

- Hub-spoke topology with Azure Virtual WAN or Azure Firewall
- All inter-VNet traffic through Azure Firewall
- IP ranges centrally managed (IPAM portal); no self-assigned ranges
- Azure Private DNS Zones for internal DNS; no custom DNS servers
- Ingress via Azure Front Door (global) or Application Gateway (regional) with WAF
- Egress through Azure Firewall with FQDN filtering; direct internet prohibited
- Private Endpoints mandatory for all PaaS services; service endpoints prohibited in production
- ExpressRoute for on-prem connectivity; VPN as backup
- NSG mandatory on all subnets with default deny rules
- NSG flow logs enabled, 90-day retention

When making networking recommendations, cite sections as `[CTSO-NET-001 §N]`.
