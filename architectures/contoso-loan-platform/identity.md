# Identity Design: Contoso Loan Application Platform

## Identity Architecture Overview

The platform serves two distinct user populations with different identity requirements:

| User Type | Identity Provider | Protocol | MFA | Standard |
|---|---|---|---|---|
| **Retail Banking Customers** | Microsoft Entra External ID (CIAM) | OIDC / OAuth 2.0 | Risk-based MFA | [CTSO-IAM-001 §1] |
| **Loan Advisors / Internal Staff** | Microsoft Entra ID (Workforce) | OIDC / OAuth 2.0 | Mandatory MFA (FIDO2/WHfB) | [CTSO-IAM-001 §2] |
| **Platform Services** | Managed Identities | OAuth 2.0 (client credentials) | N/A | [CTSO-IAM-001 §5] |
| **CI/CD Pipelines** | Workload Identity Federation | OIDC Federation | N/A | [CTSO-IAM-001 §5] |

## Customer Identity (CIAM)

### Microsoft Entra External ID

> **Note:** Azure AD B2C reached end-of-sale in May 2025. Microsoft Entra External ID is the next-generation CIAM solution per [CTSO-IAM-001 §1].

| Configuration | Value | Rationale |
|---|---|---|
| Tenant Type | External Tenant | Separate from Contoso workforce tenant |
| Protocols | OIDC / OAuth 2.0 | [CTSO-IAM-001 §1]; SAML/WS-Fed deprecated |
| Self-Service Sign-Up | Enabled | Customer self-registration for loan applications |
| MFA | Risk-based | Triggered for sensitive operations (loan submission, document upload) per [CTSO-IAM-001 §2] |
| Social Identity Providers | None initially | Financial services typically use email/password or government ID |
| Branding | Custom branded sign-in pages | White-label experience matching Contoso Banking brand |
| Data Residency | Canada | User profile data stored in Canadian region |

### Customer Authentication Flow

```
Customer (Browser/Mobile)
      │
      ▼
Customer Portal (React/Next.js)
      │ Redirect to Entra External ID
      ▼
Microsoft Entra External ID
      │ OIDC Authorization Code + PKCE
      │ Risk-based MFA (if triggered)
      ▼
ID Token + Access Token
      │
      ▼
Azure Front Door → APIM (validates JWT)
      │
      ▼
AKS Microservices (claims-based authorization)
```

### Customer Access Controls

- **Claims-based authorization** using Entra External ID app roles [CTSO-IAM-001 §3]
- Customers can only access their own loan applications (row-level filtering in services)
- Access tokens validated at the APIM layer before reaching backend services
- Token lifetime: standard OIDC defaults with refresh token rotation

## Internal Staff Identity

### Microsoft Entra ID (Workforce)

| Configuration | Value | Standard |
|---|---|---|
| Identity Provider | Contoso Entra ID tenant | [CTSO-IAM-001 §1] |
| Authentication | OIDC / OAuth 2.0 | [CTSO-IAM-001 §1] |
| MFA | Mandatory for all access | [CTSO-IAM-001 §2] |
| Preferred MFA Method | Passwordless (FIDO2, WHfB) | [CTSO-IAM-001 §2] |
| Session Lifetime | 8 hours interactive; 1 hour for elevated | [CTSO-IAM-001 §2] |
| Conditional Access | Baseline + elevated policies | [CTSO-IAM-001 §7] |

### Advisor Portal App Roles

| Role | Access | Entra ID Group |
|---|---|---|
| `LoanAdvisor` | View referred applications; submit override decisions | `SG-LoanPlatform-Advisors` |
| `SeniorUnderwriter` | All advisor access + override credit scoring decisions | `SG-LoanPlatform-SeniorUnderwriters` |
| `LoanManager` | All access + approve/reject advisor overrides; view reports | `SG-LoanPlatform-Managers` |
| `PlatformAdmin` | Service configuration; feature flags; operational dashboards | `SG-LoanPlatform-PlatformAdmins` |

### Conditional Access Policies

**Baseline Policies** (all users) [CTSO-IAM-001 §7]:
- Require MFA for all cloud apps
- Block legacy authentication protocols
- Require compliant or Entra ID-joined devices

**Elevated Policies** (privileged users — PlatformAdmin, SeniorUnderwriter) [CTSO-IAM-001 §7]:
- Require phishing-resistant MFA (FIDO2 or Windows Hello)
- GPS/IP-based location restrictions for admin operations
- Session controls: 1-hour sign-in frequency, no persistent browser sessions

## Service Identities

### Managed Identities

All Azure-to-Azure service authentication uses **Managed Identities** [CTSO-IAM-001 §5].

| Service | Identity Type | Target Resources | Standard |
|---|---|---|---|
| AKS Cluster | System-Assigned MI | ACR, VNet, Load Balancer | [CTSO-IAM-001 §5] |
| AKS Pods (each service) | Workload Identity (Entra ID) | SQL, Cosmos, Redis, Key Vault, Service Bus, Event Hubs, Blob Storage, AI Services | [CTSO-IAM-001 §5] |
| Azure Functions (if any) | System-Assigned MI | Key Vault, Service Bus | [CTSO-IAM-001 §5] |
| API Management | System-Assigned MI | Key Vault (certificates), AKS backend | [CTSO-IAM-001 §5] |

### Workload Identity Federation

For non-Azure workloads [CTSO-IAM-001 §5]:

| Workload | Federation Type | Rationale |
|---|---|---|
| GitHub Actions CI/CD | OIDC Federated Credentials | No secrets required; [CTSO-IAM-001 §5] federated credentials preferred |
| On-Premises Core Banking Integration | Certificate-based Service Principal | MI not available for on-prem; certificate preferred over secrets [CTSO-IAM-001 §5] |

### Service Principal Policy

- Service principal secrets are **discouraged** [CTSO-IAM-001 §5]
- If required: 90-day maximum expiry, stored in Key Vault [CTSO-IAM-001 §5]
- **Federated credentials** preferred for CI/CD pipelines [CTSO-IAM-001 §5]

## Application Registration

### App Registrations

| Application | Redirect URIs | Client Credentials | Owners | Standard |
|---|---|---|---|---|
| Loan Platform - Customer Portal | `https://loans.contoso.com/auth/callback` | N/A (public client, PKCE) | 2 owners min | [CTSO-IAM-001 §6] |
| Loan Platform - Advisor Portal | `https://advisor.loans.contoso.com/auth/callback` | N/A (public client, PKCE) | 2 owners min | [CTSO-IAM-001 §6] |
| Loan Platform - API | N/A (API scope definition) | Certificate (preferred) | 2 owners min | [CTSO-IAM-001 §6] |

- All redirect URIs use **HTTPS** [CTSO-IAM-001 §6]
- API permissions follow **least privilege** [CTSO-IAM-001 §6]
- No broad Graph API permissions without CISO approval [CTSO-IAM-001 §6]
- Registered in the **Contoso Entra ID tenant** [CTSO-IAM-001 §6]

## Privileged Identity Management (PIM)

| Role | Assignment | Activation Duration | Approval | Standard |
|---|---|---|---|---|
| AKS Cluster Admin | Eligible via PIM | 8 hours max | MFA + justification | [CTSO-IAM-001 §4] |
| SQL Server Admin | Eligible via PIM | 8 hours max | MFA + justification | [CTSO-IAM-001 §4] |
| Key Vault Admin | Eligible via PIM | 8 hours max | MFA + justification + approval | [CTSO-IAM-001 §4] |
| Subscription Contributor | Eligible via PIM | 8 hours max | MFA + justification + approval | [CTSO-IAM-001 §4] |

- No **permanent** Owner or Global Admin assignments [CTSO-IAM-001 §4]
- Access reviews **quarterly** for all privileged roles [CTSO-IAM-001 §4]
- Direct production database access requires PIM-activated Just-In-Time roles [CTSO-DATA-001 §8]

## Identity Monitoring

| Requirement | Implementation | Standard |
|---|---|---|
| Sign-in and audit logs | Forwarded to Microsoft Sentinel in real-time | [CTSO-IAM-001 §9] |
| Risky sign-ins | Automated investigation triggered | [CTSO-IAM-001 §9] |
| Identity Protection | Medium risk = require MFA; High risk = block + reset | [CTSO-IAM-001 §9] |
| Service principal anomalies | Monitored and alerted | [CTSO-IAM-001 §9] |
| Guest access reviews | Monthly (if any B2B access added) | [CTSO-IAM-001 §8] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| Entra External ID for customer CIAM | CTSO-IAM-001 | §1 |
| Entra ID as sole enterprise IdP | CTSO-IAM-001 | §1 |
| OIDC / OAuth 2.0 only | CTSO-IAM-001 | §1 |
| Mandatory MFA for all employees | CTSO-IAM-001 | §2 |
| Passwordless preferred (FIDO2, WHfB) | CTSO-IAM-001 | §2 |
| Risk-based MFA for customers | CTSO-IAM-001 | §2 |
| 8-hour session lifetime | CTSO-IAM-001 | §2 |
| RBAC for Azure resources | CTSO-IAM-001 | §3 |
| Claims-based app authorization | CTSO-IAM-001 | §3 |
| PIM for all privileged roles | CTSO-IAM-001 | §4 |
| Quarterly access reviews | CTSO-IAM-001 | §4 |
| Managed Identities for service auth | CTSO-IAM-001 | §5 |
| Workload Identity Federation for CI/CD | CTSO-IAM-001 | §5 |
| App registrations with 2+ owners | CTSO-IAM-001 | §6 |
| Least-privilege API permissions | CTSO-IAM-001 | §6 |
| Conditional Access baseline policies | CTSO-IAM-001 | §7 |
| Identity logs to Sentinel | CTSO-IAM-001 | §9 |
| Identity Protection policies | CTSO-IAM-001 | §9 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| CIAM solution | Entra External ID | Entra External ID recommended for new apps (B2C end-of-sale May 2025) | **Aligns** | [Entra External ID Overview](https://learn.microsoft.com/entra/external-id/customers/overview-customers-ciam) |
| Passwordless MFA | Preferred (FIDO2, WHfB) | Phishing-resistant MFA recommended for privileged users | **Aligns** | [Entra ID MFA](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-methods) |
| Workload Identity | Required for AKS pods | Recommended; replaces pod-managed identity | **Aligns** | [AKS Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-overview) |
| Managed Identities | System-assigned preferred | Recommended to eliminate credentials in code | **Aligns** | [Managed Identities Best Practices](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) |
| Federated credentials for CI/CD | Required; secrets discouraged | Recommended for GitHub Actions OIDC federation | **Aligns** | [Workload Identity Federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation) |
| PIM for privileged access | All privileged roles eligible; no standing access | Recommended; JIT access reduces attack surface | **Aligns** | [PIM Best Practices](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure) |
| Conditional Access | Baseline: MFA + compliant devices + block legacy auth | Recommended as zero-trust foundation | **Aligns** | [Conditional Access](https://learn.microsoft.com/entra/identity/conditional-access/overview) |
| Service principal secrets | Discouraged; 90-day max if required | MS Learn suggests 6-month max; Contoso is stricter | **Exceeds** | [App Registration Best Practices](https://learn.microsoft.com/entra/identity-platform/security-best-practices-for-app-registration) |
| Session lifetime | 8h interactive, 1h elevated | Configurable; shorter sessions for high-risk scenarios recommended | **Aligns** | [Session Lifetime Policies](https://learn.microsoft.com/entra/identity/conditional-access/howto-conditional-access-session-lifetime) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | Entra External ID tenant provisioning requires CISO approval for external tenant | Medium | Engage Identity & Security Team early |
| 2 | Customer password policy alignment with PIPEDA requirements | Low | NIST 800-63B-aligned defaults (no expiration, banned passwords) meet PIPEDA |
| 3 | Integration with Equifax for identity verification requires API key management | Medium | Store Equifax API key in Key Vault; rotate on 90-day cycle |
| 4 | On-premises Core Banking service principal certificate management | Medium | Certificate stored in Key Vault; automated rotation alerts |
| 5 | Risk-based MFA tuning for customer portal to balance security and UX | Low | Start with default risk policies; tune based on sign-in analytics post-launch |
