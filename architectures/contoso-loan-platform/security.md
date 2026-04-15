# Security Design: Contoso Loan Application Platform

## Security Classification

This platform handles **Confidential** data (PII + financial data) and is subject to **PIPEDA**, **OSFI**, and **AML** regulations. The security design implements defense-in-depth across all layers.

## Encryption

### Data at Rest

| Layer | Encryption | Key Management | Standard |
|---|---|---|---|
| Azure SQL Database | AES-256 TDE with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| Azure Cosmos DB | AES-256 with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| Azure Cache for Redis | AES-256 at rest | Service-managed + CMK | [CTSO-SEC-001 §1] |
| Azure Blob Storage | AES-256 with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| Azure Service Bus | AES-256 with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| AKS etcd | Encrypted by default | Azure-managed | [CTSO-SEC-001 §1] |

CMK is required for all data stores handling PII and financial data per [CTSO-SEC-001 §1] and [CTSO-DATA-001 §2].

### Data in Transit

| Path | Protocol | Standard |
|---|---|---|
| Client → Front Door | TLS 1.3 | [CTSO-SEC-001 §1] |
| Front Door → APIM | TLS 1.2+ (Private Link) | [CTSO-SEC-001 §1] |
| APIM → AKS Ingress | TLS 1.2+ (end-to-end) | [CTSO-NET-001 §4] |
| Service ↔ Service (AKS) | mTLS via Istio | [CTSO-SEC-001 §4] |
| AKS → PaaS Services | TLS 1.2+ (Private Endpoints) | [CTSO-SEC-001 §1] |
| AKS → On-Premises | TLS 1.2+ over ExpressRoute | [CTSO-SEC-001 §1] |
| AKS → External APIs | TLS 1.2+ via Azure Firewall | [CTSO-SEC-001 §1] |

- TLS 1.0 and 1.1 are **prohibited** [CTSO-SEC-001 §1]
- Self-signed certificates **prohibited** in production [CTSO-SEC-001 §1]
- All certificates stored in Azure Key Vault [CTSO-SEC-001 §1]

### Field-Level Encryption

- **Always Encrypted** with secure enclaves for:
  - Social Insurance Number (SIN)
  - Financial account numbers
  - Credit card details (if applicable)
- Ensures data remains encrypted even in database memory

## Key & Secret Management

### Azure Key Vault

| Configuration | Value | Standard |
|---|---|---|
| SKU | Premium (HSM-backed) | Required for financial services |
| Access Model | RBAC (Key Vault Crypto Officer / Secrets User) | [CTSO-SEC-001 §2] |
| Key Rotation | 90-day maximum cycle | [CTSO-SEC-001 §2] |
| Soft Delete | Enabled (90-day retention) | Default |
| Purge Protection | Enabled | Required for CMK scenarios |
| Network Access | Private Endpoint only | [CTSO-SEC-001 §3] |
| Diagnostic Settings | Enabled → Log Analytics | [CTSO-SEC-001 §6] |

### Secrets Management

- All secrets, keys, and connection strings stored in **Azure Key Vault** [CTSO-SEC-001 §2]
- Hardcoded secrets **prohibited** [CTSO-SEC-001 §2]
- AKS accesses Key Vault via **Secrets Store CSI Driver** with Workload Identity
- Application configuration references Key Vault via **Azure App Configuration** [CTSO-APP-001 §8]

## Network Security

Covered in detail in [networking.md](networking.md). Key security controls:

- Private Endpoints for all PaaS services [CTSO-SEC-001 §3]
- Public endpoints **prohibited** in production [CTSO-SEC-001 §3]
- Azure Firewall with FQDN filtering for egress [CTSO-SEC-001 §3]
- Azure WAF (Front Door) with OWASP 3.2 ruleset [CTSO-SEC-001 §3]
- DDoS Protection Standard on all production VNets [CTSO-SEC-001 §3]

## Authentication & Authorization

- All service-to-service communication via **Managed Identities** [CTSO-SEC-001 §4]
- **mTLS** required for all inter-service communication via Istio [CTSO-SEC-001 §4]
- User-facing applications enforce **MFA** via Microsoft Entra ID [CTSO-SEC-001 §4]
- No service principal secrets; certificates preferred when MI is not possible [CTSO-SEC-001 §4]

See [identity.md](identity.md) for detailed identity design.

## Application Security

### Container Image Security

| Control | Implementation | Standard |
|---|---|---|
| Base images | Contoso-approved from internal ACR | [CTSO-APP-001 §4] |
| Image scanning | Trivy + Microsoft Defender for Containers | [CTSO-SEC-001 §5] |
| CVE policy | Critical/High CVEs blocked from production | [CTSO-SEC-001 §5] |
| Image tags | Immutable (SHA digests); `latest` prohibited | [CTSO-APP-001 §4] |
| Public Docker Hub | **Prohibited**; re-host in Contoso ACR | [CTSO-SEC-001 §8] |

### OWASP Top 10 Mitigations

| OWASP Category | Mitigation |
|---|---|
| A01: Broken Access Control | Claims-based authorization via Entra ID; RBAC at API layer |
| A02: Cryptographic Failures | TLS 1.2+; AES-256 CMK at rest; Always Encrypted for sensitive fields |
| A03: Injection | Parameterized queries (Entity Framework); input validation; WAF SQL injection rules |
| A04: Insecure Design | Threat modeling before development; security review gates |
| A05: Security Misconfiguration | IaC-enforced configuration; Azure Policy; deployment safeguards |
| A06: Vulnerable Components | SCA scanning (NuGet Audit, npm audit); automatic dependency updates |
| A07: Authentication Failures | Entra ID with MFA; passwordless preferred; no local accounts |
| A08: Data Integrity Failures | SAST/DAST in CI/CD; signed container images; immutable tags |
| A09: Logging & Monitoring Failures | Structured logging; Sentinel SIEM; 365-day audit retention |
| A10: Server-Side Request Forgery | Egress filtering via Azure Firewall; deny-all-by-default |

Per [CTSO-SEC-001 §5].

### CI/CD Security Controls

| Stage | Security Control | Standard |
|---|---|---|
| Build | SAST (CodeQL / SonarQube) | [CTSO-SEC-001 §5] |
| Build | SCA (NuGet Audit, npm audit, Dependabot) | [CTSO-SEC-001 §5] |
| Container Build | Trivy + Defender image scan | [CTSO-SEC-001 §5] |
| Pre-Deploy | Block on Critical/High CVEs | [CTSO-SEC-001 §5] |
| QA | DAST (OWASP ZAP) | [CTSO-SEC-001 §5] |
| Production | 2-approver gate | [CTSO-APP-001 §6] |

## Logging & Monitoring (Security)

| Requirement | Implementation | Standard |
|---|---|---|
| SIEM | Microsoft Sentinel | [CTSO-SEC-001 §6] |
| Event forwarding | Security events to Sentinel within 5 minutes | [CTSO-SEC-001 §6] |
| Auth failure alerts | Alert after 5 consecutive failures | [CTSO-SEC-001 §6] |
| Audit log retention | 365 days minimum | [CTSO-SEC-001 §6] |
| Diagnostic settings | Enabled on all Azure resources | [CTSO-SEC-001 §6] |
| Defender for Cloud | Enabled on all subscriptions | [CTSO-SEC-001 §8] |
| Defender for Containers | Enabled on AKS cluster | [CTSO-SEC-001 §5] |
| Defender for SQL | Advanced Threat Protection enabled | [CTSO-DATA-001 §4] |

## Compliance

| Requirement | Implementation | Standard |
|---|---|---|
| PIPEDA | Data residency in Canada; right-to-erasure procedures; consent management | [CTSO-SEC-001 §7] |
| OSFI | Audit logging; access controls; encryption; DR testing | [CTSO-SEC-001 §7] |
| AML | Transaction monitoring via Decision Engine; suspicious activity reporting | Regulatory |
| SOC 2 Type II | Continuous compliance monitoring via Defender for Cloud | [CTSO-SEC-001 §7] |
| Data Residency | Canada Central + Canada East only | [CTSO-SEC-001 §7] |

## Prohibited Practices Checklist

| Prohibited Practice | Status | Standard |
|---|---|---|
| Secrets in source code / env vars / appsettings | ✅ Prevented — Key Vault only | [CTSO-SEC-001 §8] |
| Shared accounts / generic service accounts | ✅ Prevented — individual identities + MI | [CTSO-SEC-001 §8] |
| Disabled Defender for Cloud | ✅ Prevented — enabled on all subscriptions | [CTSO-SEC-001 §8] |
| Owner/Contributor without PIM | ✅ Prevented — PIM activation required | [CTSO-SEC-001 §8] |
| Public Docker Hub images | ✅ Prevented — Contoso ACR only | [CTSO-SEC-001 §8] |
| SQL authentication in production | ✅ Prevented — Entra ID only | [CTSO-DATA-001 §8] |
| Public endpoints on databases | ✅ Prevented — Private Endpoints only | [CTSO-DATA-001 §8] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| AES-256 + CMK for all data stores | CTSO-SEC-001 | §1 |
| TLS 1.2+ mandatory; 1.0/1.1 prohibited | CTSO-SEC-001 | §1 |
| Certificates in Key Vault; no self-signed | CTSO-SEC-001 | §1 |
| All secrets in Key Vault | CTSO-SEC-001 | §2 |
| 90-day key rotation | CTSO-SEC-001 | §2 |
| RBAC access to Key Vault | CTSO-SEC-001 | §2 |
| Private Endpoints; no public endpoints | CTSO-SEC-001 | §3 |
| WAF with OWASP 3.2 ruleset | CTSO-SEC-001 | §3 |
| DDoS Protection Standard | CTSO-SEC-001 | §3 |
| Managed Identities for service-to-service | CTSO-SEC-001 | §4 |
| mTLS for inter-service communication | CTSO-SEC-001 | §4 |
| MFA for user-facing apps | CTSO-SEC-001 | §4 |
| Container image scanning | CTSO-SEC-001 | §5 |
| SAST, DAST, SCA in CI/CD | CTSO-SEC-001 | §5 |
| OWASP Top 10 mitigations | CTSO-SEC-001 | §5 |
| Sentinel SIEM within 5 minutes | CTSO-SEC-001 | §6 |
| 365-day audit log retention | CTSO-SEC-001 | §6 |
| PIPEDA, SOC 2 compliance | CTSO-SEC-001 | §7 |
| Canadian data residency | CTSO-SEC-001 | §7 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| CMK for data at rest | Required for Confidential data | Recommended for compliance workloads; CMK with Key Vault | **Aligns** | [Azure SQL Security](https://learn.microsoft.com/azure/well-architected/service-guides/azure-sql-database#security) |
| TLS 1.2+ enforcement | TLS 1.2 minimum, 1.3 preferred | TLS 1.2 recommended minimum; enforce via server settings | **Aligns** | [Azure SQL TLS](https://learn.microsoft.com/azure/azure-sql/database/security-best-practice?view=azuresql#network-security) |
| mTLS between services | Required | Recommended via service mesh (Istio) for zero-trust | **Exceeds** — Many orgs rely on network isolation only | [AKS Advanced Microservices](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices-advanced) |
| Image scanning | Trivy + Defender mandatory | Defender for Containers recommended; deployment safeguards | **Aligns** | [AKS Advanced Security](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices-advanced#considerations) |
| SAST/DAST/SCA | All mandatory in CI/CD | Defender for DevOps + agentless code scanning recommended | **Aligns** | [Defender for DevOps](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-devops-introduction) |
| Audit log retention | 365 days minimum | No specific minimum in MS Learn; Sentinel supports long retention | **Exceeds** — Contoso mandates 365 days | [Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/overview) |
| Key rotation | 90-day maximum | Recommended; automated rotation with Key Vault | **Exceeds** — Some MS Learn guidance suggests 180-365 days | [Key Vault Best Practices](https://learn.microsoft.com/azure/key-vault/general/best-practices) |
| WAF mode | Prevention mode required | Prevention mode recommended; Detection for performance-sensitive scenarios | **Aligns** | [WAF Best Practices](https://learn.microsoft.com/azure/well-architected/mission-critical/mission-critical-networking-connectivity#application-delivery-services) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | Threat modeling must be completed before development begins | High | Schedule STRIDE threat modeling session with CISO Office |
| 2 | Always Encrypted with secure enclaves requires Azure SQL Business Critical tier | Medium | Already selected; validate enclave support in Canada Central |
| 3 | Key rotation automation for 90-day cycle requires Key Vault event triggers | Low | Implement via Event Grid + Azure Functions for rotation notifications |
| 4 | PCI-DSS compliance may be needed if payment processing is added later | Medium | Currently out of scope; flag for ARB if payment features are added |
| 5 | AML transaction monitoring rules must be defined with Compliance team | High | Engage Regulatory Compliance Team during MVP planning |
