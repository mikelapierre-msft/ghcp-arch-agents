# Contoso Security Standards

**Document ID:** CTSO-SEC-001  
**Version:** 3.2  
**Last Updated:** 2026-01-15  
**Owner:** Contoso CISO Office  
**Classification:** Internal

## 1. Encryption

- **Data at rest**: AES-256 encryption required for all data stores. Azure Storage must use Microsoft-managed keys (MMK) at minimum; customer-managed keys (CMK) via Azure Key Vault required for PII and financial data.
- **Data in transit**: TLS 1.2 minimum; TLS 1.3 preferred. TLS 1.0 and 1.1 are **prohibited**.
- **Certificate management**: All certificates must be stored in Azure Key Vault. Self-signed certificates are **prohibited** in production.

## 2. Key & Secret Management

- All secrets, keys, and connection strings must be stored in **Azure Key Vault**. Hardcoded secrets are **prohibited**.
- Key rotation: **90-day maximum** rotation cycle for all cryptographic keys.
- Access to Key Vault must use **RBAC** (Key Vault Crypto Officer / Secrets User). Access policies mode is deprecated.

## 3. Network Security

- All production workloads must use **Private Endpoints**. Public endpoints are **prohibited** in production.
- Azure Firewall or third-party NVA required for egress filtering.
- Web applications must be fronted by **Azure WAF** (Application Gateway or Front Door).
- DDoS Protection Standard must be enabled on all production VNets.

## 4. Authentication & Authorization

- All service-to-service communication must use **Managed Identities**. Service principal secrets are discouraged; certificates preferred when MI is not possible.
- **Mutual TLS (mTLS)** is required for all inter-service communication within the platform.
- All user-facing applications must enforce **Multi-Factor Authentication (MFA)** via Microsoft Entra ID.

## 5. Application Security

- All container images must be scanned for vulnerabilities before deployment. Images with **Critical** or **High** CVEs are blocked from production.
- OWASP Top 10 mitigations must be validated in security review.
- Static Application Security Testing (SAST) and Dynamic Application Security Testing (DAST) are mandatory in CI/CD pipelines.
- Software Composition Analysis (SCA) required for all third-party dependencies.

## 6. Logging & Monitoring

- Security events must be forwarded to **Microsoft Sentinel** within 5 minutes.
- Failed authentication attempts must trigger alerts after **5 consecutive failures**.
- Audit logs must be retained for a minimum of **365 days**.
- Diagnostic settings must be enabled on all Azure resources.

## 7. Compliance Requirements

- PCI-DSS compliance required for payment processing workloads.
- SOC 2 Type II certification must be maintained.
- PIPEDA compliance mandatory for all Canadian customer data.
- Data residency: Canadian customer data must remain in **Canada Central** or **Canada East** Azure regions.

## 8. Prohibited Practices

- Storing secrets in source code, environment variables, or app settings without Key Vault references.
- Using shared accounts or generic service accounts.
- Disabling Azure Defender / Microsoft Defender for Cloud on any subscription.
- Granting `Owner` or `Contributor` roles at the subscription level without PIM activation.
- Using public Docker Hub images directly without re-hosting in Contoso Azure Container Registry.
