---
description: "Use when discussing cloud infrastructure, Azure regions, compute platforms (AKS, ACA, App Service, Functions), virtual machines, subscription design, landing zones, resource naming, tagging, IaC (Bicep, Terraform), environment strategy, high availability, disaster recovery, SLA targets, cost management, reserved instances, or Azure resource provisioning."
---
# Contoso Infrastructure Standards

Read the full Contoso infrastructure standards before making recommendations:

- [Infrastructure Standards — CTSO-INFRA-001](../../../standards/infrastructure.md)

## Key Policies

- Azure-only cloud (no multi-cloud)
- Primary region: Canada Central; DR region: Canada East
- Compute preference: AKS > ACA > App Service > Functions; VMs discouraged
- AKS must use managed identity, Azure CNI overlay, private cluster mode
- All infra via IaC (Bicep preferred, Terraform approved); no portal provisioning in production
- Naming: `{company}-{workload}-{env}-{resource}-{region}-{instance}`
- Required tags: CostCenter, Owner, Environment, DataClassification, ApplicationName
- Production SLA: 99.95% minimum, multi-zone deployment
- DR: Active-Passive, RPO ≤ 1h, RTO ≤ 4h
- Dev/QA auto-shutdown outside business hours

When making infrastructure recommendations, cite sections as `[CTSO-INFRA-001 §N]`.
