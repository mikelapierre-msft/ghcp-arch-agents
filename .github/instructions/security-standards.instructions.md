---
description: "Use when discussing security architecture, encryption, key management, TLS, WAF, DDoS protection, vulnerability scanning, SAST, DAST, SCA, secrets management, Azure Key Vault, Microsoft Defender, Sentinel, compliance (PCI-DSS, SOC2, PIPEDA), data residency, or threat modeling."
---
# Contoso Security Standards

Read the full Contoso security standards before making recommendations:

- [Security Standards — CTSO-SEC-001](../../../standards/security.md)

## Key Policies

- All secrets in Azure Key Vault — no hardcoded secrets
- 90-day key rotation cycle
- TLS 1.2+ mandatory; TLS 1.0/1.1 prohibited
- Private Endpoints only — no public endpoints in production
- mTLS required for inter-service communication
- Container images scanned for CVEs; Critical/High blocked from production
- SAST, DAST, SCA mandatory in CI/CD
- Security events to Microsoft Sentinel within 5 minutes
- Audit log retention: 365 days minimum
- Canadian data residency required (Canada Central / Canada East)

When making security recommendations, cite sections as `[CTSO-SEC-001 §N]`.
