# Security — Contoso Loan Application Platform

## Threat Model Summary

The Loan Application Platform processes **Confidential** data (PII + financial information) and is subject to PIPEDA, OSFI, and AML regulations. The primary threat categories include:

| Threat Category | Key Concerns | Mitigations |
|----------------|-------------|-------------|
| Data exfiltration | PII leakage, unauthorized data access | CMK encryption, Private Endpoints, DLP policies |
| Identity attacks | Credential theft, session hijacking | MFA, Conditional Access, token binding |
| API abuse | Injection, rate limiting bypass | WAF, APIM throttling, input validation |
| Supply chain | Compromised dependencies, container images | SCA, image scanning, Contoso ACR |
| Insider threat | Unauthorized advisor access, data manipulation | PIM, audit logging, RBAC |

## Encryption

### Data at Rest

| Asset | Encryption | Key Management | Standard Reference |
|-------|-----------|----------------|-------------------|
| Azure SQL Database | TDE with CMK | Azure Key Vault (RSA 2048) | [CTSO-SEC-001 §1] |
| Azure Cosmos DB | Service-side encryption with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| Azure Blob Storage (documents) | AES-256 with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |
| Azure Cache for Redis | At-rest encryption (Enterprise tier) | Service-managed | [CTSO-SEC-001 §1] |
| Azure Service Bus | At-rest encryption with CMK | Azure Key Vault | [CTSO-SEC-001 §1] |

CMK is required for all stores containing Confidential data [CTSO-SEC-001 §1], [CTSO-DATA-001 §2].

### Data in Transit

| Path | Protocol | Standard Reference |
|------|----------|-------------------|
| Client → Front Door | TLS 1.3 | [CTSO-SEC-001 §1] |
| Front Door → Application Gateway | TLS 1.2+ | [CTSO-SEC-001 §1] |
| App Gateway → AKS | TLS 1.2+ (end-to-end) | [CTSO-NET-001 §4] |
| Inter-service (within AKS) | mTLS via Istio | [CTSO-SEC-001 §4] |
| AKS → PaaS services | TLS 1.2+ over Private Endpoint | [CTSO-SEC-001 §1] |
| AKS → Equifax/SendGrid/Twilio | TLS 1.2+ via Azure Firewall | [CTSO-SEC-001 §1] |
| On-prem → Azure | Encrypted ExpressRoute (MACsec) | [CTSO-NET-001 §6] |

**TLS 1.0 and 1.1 are prohibited** across all endpoints [CTSO-SEC-001 §1].

### Certificate Management

| Certificate | Storage | Rotation | Standard Reference |
|-------------|---------|----------|-------------------|
| Front Door TLS (customer domain) | Azure Key Vault | Auto-renew (Front Door managed) | [CTSO-SEC-001 §1] |
| App Gateway TLS | Azure Key Vault | 90-day rotation | [CTSO-SEC-001 §1, §2] |
| Istio mTLS (inter-service) | Istio CA (cert-manager + Key Vault) | Auto-rotate (24h) | [CTSO-SEC-001 §4] |
| Database CMK keys | Azure Key Vault | 90-day rotation | [CTSO-SEC-001 §2] |

No self-signed certificates in production [CTSO-SEC-001 §1].

## Key & Secret Management

### Azure Key Vault Configuration

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| SKU | Premium (HSM-backed) | Financial services requirement |
| Access model | RBAC (Key Vault Crypto Officer / Secrets User) | [CTSO-SEC-001 §2] |
| Soft delete | Enabled (90-day retention) | Best practice |
| Purge protection | Enabled | Compliance requirement |
| Network | Private Endpoint only | [CTSO-SEC-001 §3] |
| Diagnostic logs | Sent to Microsoft Sentinel | [CTSO-SEC-001 §6] |

**Secrets stored in Key Vault:**

| Secret Category | Examples | Access Pattern |
|----------------|---------|----------------|
| External API keys | Equifax API key, SendGrid API key, Twilio credentials | Managed Identity → Key Vault reference |
| CMK encryption keys | SQL TDE key, Cosmos DB key, Storage key | Azure-managed auto-rotation |
| TLS certificates | App Gateway, Istio root CA | cert-manager integration |

**Prohibited**: Hardcoded secrets, secrets in environment variables, secrets in appsettings [CTSO-SEC-001 §2], [CTSO-DATA-001 §8].

## Network Security

| Control | Implementation | Standard Reference |
|---------|---------------|-------------------|
| Private Endpoints | All PaaS services (see [networking.md](networking.md)) | [CTSO-SEC-001 §3] |
| WAF | Azure Front Door + Application Gateway (OWASP 3.2) | [CTSO-SEC-001 §3] |
| DDoS Protection | Standard tier on production VNets | [CTSO-SEC-001 §3] |
| Azure Firewall | Hub-based egress filtering with FQDN rules | [CTSO-SEC-001 §3] |
| NSGs | Default deny on all subnets | [CTSO-NET-001 §7] |
| mTLS | Istio service mesh for all inter-service calls | [CTSO-SEC-001 §4] |

## Application Security

### CI/CD Security Pipeline

```
Code Commit → SAST (CodeQL) → Unit Tests → SCA (Dependabot/Snyk) → 
Container Build → Container Scan (Trivy + Defender) → Deploy to Dev → 
Integration Tests → Deploy to QA → DAST (OWASP ZAP) → Deploy to Staging → 
Smoke Tests → Manual Approval (2 approvers) → Deploy to Prod
```

| Security Gate | Tool | Blocking Criteria | Standard Reference |
|--------------|------|-------------------|-------------------|
| SAST | CodeQL (GitHub Advanced Security) | Critical/High findings | [CTSO-SEC-001 §5] |
| SCA | Dependabot + Snyk | Known CVEs in dependencies | [CTSO-SEC-001 §5] |
| Container scan | Trivy + Microsoft Defender for Containers | Critical/High CVEs | [CTSO-SEC-001 §5], [CTSO-APP-001 §4] |
| DAST | OWASP ZAP | OWASP Top 10 violations | [CTSO-SEC-001 §5] |
| Image provenance | Contoso ACR only | Public Docker Hub images blocked | [CTSO-SEC-001 §8] |

### OWASP Top 10 Mitigations

| OWASP Risk | Mitigation |
|-----------|-----------|
| A01: Broken Access Control | Entra ID RBAC + claims-based authorization; API-level policy enforcement at APIM |
| A02: Cryptographic Failures | TLS 1.2+, CMK encryption, Key Vault for all secrets |
| A03: Injection | Parameterized queries (EF Core), input validation, WAF rules |
| A04: Insecure Design | Threat modeling, security reviews, DDD bounded contexts |
| A05: Security Misconfiguration | IaC (Bicep), Azure Policy, Defender for Cloud compliance |
| A06: Vulnerable Components | SCA scanning, Dependabot, Contoso-approved base images |
| A07: Auth Failures | Entra ID MFA, Conditional Access, token validation at APIM |
| A08: Data Integrity Failures | Signed container images (SHA digests), pipeline integrity |
| A09: Logging Failures | Structured logging, Sentinel integration, 365-day retention |
| A10: SSRF | Egress filtering via Azure Firewall, no direct internet from services |

## Logging & Monitoring (Security)

| Log Source | Destination | Retention | Alert Threshold | Standard Reference |
|-----------|-------------|-----------|----------------|-------------------|
| Entra ID sign-in logs | Microsoft Sentinel | 365 days | Risky sign-in detected | [CTSO-SEC-001 §6] |
| Azure Firewall logs | Log Analytics / Sentinel | 365 days | Denied egress attempts | [CTSO-SEC-001 §6] |
| Key Vault access logs | Microsoft Sentinel | 365 days | Unauthorized access attempt | [CTSO-SEC-001 §6] |
| NSG flow logs | Log Analytics | 90 days | Anomalous traffic patterns | [CTSO-NET-001 §7] |
| AKS audit logs | Microsoft Sentinel | 365 days | Privilege escalation attempt | [CTSO-SEC-001 §6] |
| Application auth failures | Application Insights / Sentinel | 365 days | 5 consecutive failures | [CTSO-SEC-001 §6] |
| WAF logs | Log Analytics / Sentinel | 365 days | Attack pattern detected | [CTSO-SEC-001 §6] |
| Microsoft Defender alerts | Microsoft Sentinel | 365 days | Any alert | [CTSO-SEC-001 §6] |

All security events forwarded to Sentinel **within 5 minutes** [CTSO-SEC-001 §6].

## Microsoft Defender Coverage

| Defender Plan | Scope | Standard Reference |
|-------------|-------|-------------------|
| Defender for Cloud | All subscriptions | [CTSO-SEC-001 §8] |
| Defender for Containers | AKS clusters + ACR | [CTSO-SEC-001 §5] |
| Defender for SQL | Azure SQL databases | [CTSO-DATA-001 §4] |
| Defender for Key Vault | Key Vault instances | Best practice |
| Defender for Storage | Blob storage (documents) | Best practice |
| Defender for App Service | Reporting dashboard | Best practice |

Disabling Defender is **prohibited** [CTSO-SEC-001 §8].

## Compliance

| Regulation | Requirement | Implementation |
|-----------|-------------|----------------|
| PIPEDA | Privacy of Canadian personal data | Data residency in Canada; consent management; right-to-erasure |
| OSFI | Operational resilience for financial institutions | HA/DR architecture; quarterly failover tests |
| AML | Anti-money laundering screening | Integration with KYC service; audit trail retention |
| SOC 2 Type II | Security controls attestation | Continuous compliance via Defender for Cloud |
| PCI-DSS | (Future) If payment processing added | Architecture designed for PCI readiness |

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| CMK encryption for Confidential data | CTSO-SEC-001 | §1 |
| TLS 1.2+ mandatory, TLS 1.0/1.1 prohibited | CTSO-SEC-001 | §1 |
| Certificates in Key Vault | CTSO-SEC-001 | §1 |
| Key Vault with RBAC, 90-day rotation | CTSO-SEC-001 | §2 |
| Private Endpoints, WAF, DDoS Protection | CTSO-SEC-001 | §3 |
| mTLS for inter-service communication | CTSO-SEC-001 | §4 |
| Managed Identities for service auth | CTSO-SEC-001 | §4 |
| Container image scanning (Critical/High block) | CTSO-SEC-001 | §5 |
| SAST, DAST, SCA in CI/CD | CTSO-SEC-001 | §5 |
| Sentinel integration within 5 minutes | CTSO-SEC-001 | §6 |
| 365-day audit log retention | CTSO-SEC-001 | §6 |
| No secrets in code or env vars | CTSO-SEC-001 | §8 |
| Defender enabled on all subscriptions | CTSO-SEC-001 | §8 |
| Contoso ACR only (no public Docker Hub) | CTSO-SEC-001 | §8 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Encryption at rest | CMK for PII/financial data | Service-managed keys default; CMK recommended for sensitive data | **Exceeds** (mandatory CMK) | [CMK for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/transparent-data-encryption-byok-overview) |
| mTLS | Required for all inter-service calls | Recommended for zero-trust architectures | **Aligns** | [Zero Trust networking](https://learn.microsoft.com/azure/architecture/example-scenario/gateway/zero-trust-network) |
| Key rotation | 90-day maximum | Annual rotation typical; shorter cycles recommended | **Exceeds** | [Key management best practices](https://learn.microsoft.com/azure/key-vault/keys/about-keys) |
| Security scanning | SAST + DAST + SCA mandatory | Recommended as part of DevSecOps | **Aligns** | [DevSecOps on AKS](https://learn.microsoft.com/azure/architecture/guide/devsecops/devsecops-on-aks) |
| Container images | Contoso ACR only; public images prohibited | ACR recommended; image quarantine supported | **Exceeds** | [ACR security](https://learn.microsoft.com/azure/container-registry/container-registry-best-practices) |
| Audit retention | 365 days minimum | 90 days default in Log Analytics | **Exceeds** | [Log Analytics data retention](https://learn.microsoft.com/azure/azure-monitor/logs/data-retention-configure) |
| Sentinel integration | All security events within 5 min | Real-time recommended for critical workloads | **Aligns** | [Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/overview) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| mTLS via Istio adds certificate management complexity | Use cert-manager with Key Vault integration; auto-rotation policy | Design decision |
| 365-day Log Analytics retention increases cost | Use Basic logs tier for high-volume, low-query data; archive to Storage | Estimate cost |
| WAF false positives may block legitimate loan applications | Tune rules in Detection mode before switching to Prevention | Pre-GA task |
| Equifax/SendGrid/Twilio API key management | Store in Key Vault with 90-day rotation; monitor for compromise | Implement in MVP |
| PIPEDA compliance requires formal Privacy Impact Assessment | Engage Contoso Privacy Office for PIA before GA launch | **Action required** |
