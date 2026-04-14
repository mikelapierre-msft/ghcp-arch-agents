# Contoso Infrastructure Standards

**Document ID:** CTSO-INFRA-001  
**Version:** 2.8  
**Last Updated:** 2026-02-01  
**Owner:** Contoso Cloud Platform Team  
**Classification:** Internal

## 1. Cloud Provider & Regions

- **Primary cloud**: Microsoft Azure (exclusively). Multi-cloud is not approved.
- **Primary region**: Canada Central. **DR region**: Canada East.
- All production workloads must be deployed in **paired Canadian regions** for data residency compliance.
- Non-production workloads may use US regions (East US 2) for cost optimization.

## 2. Subscription & Resource Organization

- Landing zone model: **Azure Landing Zone (ALZ)** with management group hierarchy.
- Subscription strategy: **Workload-based** — one subscription per application per environment.
- Resource naming convention: `{company}-{workload}-{env}-{resource}-{region}-{instance}` (e.g., `ctso-payments-prod-aks-cc-001`).
- All resources must have tags: `CostCenter`, `Owner`, `Environment`, `DataClassification`, `ApplicationName`.

## 3. Compute

- **Approved compute platforms** (in order of preference):
  1. Azure Kubernetes Service (AKS)
  2. Azure Container Apps (ACA)
  3. Azure App Service
  4. Azure Functions (serverless workloads only)
- Virtual Machines are **discouraged** for new workloads. Exceptions require Architecture Review Board approval.
- AKS clusters must use **managed identity**, **Azure CNI overlay**, and **private cluster** mode.
- Container images must be hosted in **Contoso Azure Container Registry (ACR)** with geo-replication.

## 4. Infrastructure as Code

- All infrastructure must be provisioned via **IaC**. Manual Azure Portal provisioning is **prohibited** in production.
- **Approved IaC tools**: Bicep (preferred), Terraform (approved with state in Azure Storage).
- ARM templates are deprecated. CloudFormation and Pulumi are not approved.
- IaC templates must be stored in the application repository and deployed via CI/CD.

## 5. Environment Strategy

| Environment | Purpose | SLA | Region |
|---|---|---|---|
| Dev | Developer testing | None | Canada Central |
| QA | Integration testing | None | Canada Central |
| Staging | Pre-production validation | 99.5% | Canada Central |
| Production | Live workloads | 99.95% | Canada Central + Canada East |

## 6. High Availability & Disaster Recovery

- Production workloads must achieve **99.95% SLA** minimum.
- **Multi-zone** deployment required for all production compute and data services.
- DR strategy: **Active-Passive** with RPO ≤ 1 hour and RTO ≤ 4 hours.
- Automated failover testing required quarterly.
- Azure Site Recovery or native service replication for stateful workloads.

## 7. Cost Management

- All resources must have **Azure Cost Management** budgets and alerts.
- Dev/QA environments must auto-shutdown outside business hours (8 AM – 6 PM ET).
- Reserved Instances required for predictable production workloads (1-year minimum).
- Spot instances approved for batch processing and non-critical workloads.

## 8. Prohibited Practices

- Deploying resources without IaC.
- Using classic Azure resources (Cloud Services, classic VMs, classic storage).
- Deploying production workloads outside Canadian regions without CISO approval.
- Running AKS clusters without node auto-scaling.
- Using Basic SKU for any production load balancer, public IP, or storage account.
