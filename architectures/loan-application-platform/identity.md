# Identity — Contoso Loan Application Platform

## Identity Architecture Overview

The platform uses **Microsoft Entra ID** as the sole identity provider [CTSO-IAM-001 §1], with distinct identity flows for three user populations:

| User Population | Identity Provider | Auth Protocol | MFA | Standard Reference |
|----------------|-------------------|---------------|-----|-------------------|
| Banking customers | Entra External ID (CIAM) | OIDC / OAuth 2.0 | Risk-based MFA | [CTSO-IAM-001 §1, §2] |
| Loan advisors (employees) | Entra ID (corporate tenant) | OIDC / OAuth 2.0 | MFA mandatory (phishing-resistant preferred) | [CTSO-IAM-001 §2, §7] |
| Platform operators | Entra ID (corporate tenant) | OIDC / OAuth 2.0 | Phishing-resistant MFA (FIDO2) | [CTSO-IAM-001 §2, §7] |

## Customer Identity (Entra External ID)

### Configuration

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| Platform | Entra External ID (CIAM) | [CTSO-IAM-001 §1] |
| Sign-up/Sign-in | Email + password, Social providers (optional) | [CTSO-IAM-001 §1] |
| MFA | Risk-based — triggered for sensitive operations (loan submission, document upload) | [CTSO-IAM-001 §2] |
| Password policy | Min 14 characters, complexity required, banned password list | [CTSO-IAM-001 §2] |
| Session lifetime | 8 hours interactive | [CTSO-IAM-001 §2] |
| Token format | OAuth 2.0 access tokens (JWT) | [CTSO-IAM-001 §1] |

### Customer Authorization

| Operation | Required Claims | Enforcement |
|----------|----------------|-------------|
| View own application | `user_id` claim matches application owner | Application-level |
| Submit new application | Authenticated + MFA step-up | APIM policy + application |
| Upload documents | Authenticated + MFA step-up | APIM policy + application |
| View decision | `user_id` claim matches application owner | Application-level |

## Employee Identity (Entra ID — Corporate)

### Advisor Portal Authentication

| Parameter | Value | Standard Reference |
|-----------|-------|-------------------|
| Identity provider | Entra ID (corporate tenant) | [CTSO-IAM-001 §1] |
| Protocol | OIDC / OAuth 2.0 | [CTSO-IAM-001 §1] |
| MFA | Mandatory for all employees | [CTSO-IAM-001 §2] |
| Preferred auth | Passwordless (FIDO2, Windows Hello) | [CTSO-IAM-001 §2] |
| Session lifetime | 8 hours interactive | [CTSO-IAM-001 §2] |
| Device compliance | Required (Conditional Access) | [CTSO-IAM-001 §7] |

### Advisor Authorization (App Roles)

| App Role | Permissions | Assigned Via |
|----------|-----------|-------------|
| `LoanAdvisor` | View assigned applications, add notes, approve/decline referred loans | Entra ID group |
| `SeniorAdvisor` | All LoanAdvisor + override decision engine, escalate | Entra ID group |
| `LoanManager` | All SeniorAdvisor + reassign cases, view team dashboard | Entra ID group |
| `ComplianceOfficer` | Read-only access to all applications + audit trails | Entra ID group |
| `Admin` | Platform configuration, feature flags | Entra ID group + PIM activation |

Authorization enforced via **claims-based access control** using Entra ID app roles and group memberships [CTSO-IAM-001 §3].

### Conditional Access Policies

| Policy | Target | Controls | Standard Reference |
|--------|--------|----------|-------------------|
| Require MFA for Advisor Portal | All employees accessing `advisor.contoso.com` | MFA required | [CTSO-IAM-001 §7] |
| Block legacy auth | All cloud apps | Block | [CTSO-IAM-001 §7] |
| Require compliant device | Corporate resources | Compliant / Entra ID-joined device | [CTSO-IAM-001 §7] |
| Admin sign-in restrictions | Platform Admin role | Phishing-resistant MFA + GPS/IP restriction + 1-hour session | [CTSO-IAM-001 §7] |
| Risk-based for advisors | LoanAdvisor, SeniorAdvisor | Medium risk → MFA, High risk → Block | [CTSO-IAM-001 §9] |

## Service Identities

### Managed Identities

All service-to-Azure authentication uses **Managed Identities** [CTSO-IAM-001 §5], [CTSO-SEC-001 §4]:

| Service | Identity Type | Accesses | Standard Reference |
|---------|--------------|---------|-------------------|
| AKS cluster | System-assigned MI | ACR pull, Key Vault, network resources | [CTSO-IAM-001 §5] |
| Application Intake Service | User-assigned MI (Workload Identity) | Azure SQL, Service Bus, App Configuration | [CTSO-IAM-001 §5] |
| Credit Scoring Service | User-assigned MI (Workload Identity) | Azure ML, Azure SQL, Service Bus | [CTSO-IAM-001 §5] |
| Document Processing Service | User-assigned MI (Workload Identity) | Blob Storage, Cosmos DB, Document Intelligence | [CTSO-IAM-001 §5] |
| Decision Engine Service | User-assigned MI (Workload Identity) | Azure SQL, Service Bus, Redis | [CTSO-IAM-001 §5] |
| Notification Service | User-assigned MI (Workload Identity) | Service Bus, Key Vault (external API keys) | [CTSO-IAM-001 §5] |
| Advisor Portal BFF | User-assigned MI (Workload Identity) | Azure SQL, Redis, Service Bus | [CTSO-IAM-001 §5] |
| Reporting Service | User-assigned MI (Workload Identity) | Azure SQL (read replica), Redis | [CTSO-IAM-001 §5] |

Each microservice gets a **unique Workload Identity** for least-privilege RBAC [CTSO-IAM-001 §5].

### CI/CD Identity

| Pipeline | Identity | Auth Method | Standard Reference |
|----------|----------|-------------|-------------------|
| GitHub Actions → Azure | Workload Identity Federation | Federated credentials (no secrets) | [CTSO-IAM-001 §5] |
| GitHub Actions → AKS | Workload Identity Federation | OIDC token exchange | [CTSO-IAM-001 §5] |

No service principal secrets for CI/CD pipelines [CTSO-IAM-001 §5].

## Privileged Identity Management (PIM)

| Role | Scope | PIM Config | Standard Reference |
|------|-------|-----------|-------------------|
| AKS Cluster Admin | AKS resource group | Eligible only, 8h max, MFA + justification | [CTSO-IAM-001 §4] |
| SQL Server Admin | Azure SQL instance | Eligible only, 8h max, MFA + justification | [CTSO-IAM-001 §4] |
| Key Vault Admin | Key Vault instances | Eligible only, 8h max, MFA + justification + approval | [CTSO-IAM-001 §4] |
| Subscription Contributor | Workload subscription | Eligible only, 8h max, MFA + justification + approval | [CTSO-IAM-001 §4] |

No permanent Owner or Global Admin assignments [CTSO-IAM-001 §4]. Quarterly access reviews required [CTSO-IAM-001 §4].

## Application Registrations

| App Registration | Purpose | Owners | API Permissions | Standard Reference |
|-----------------|---------|--------|----------------|-------------------|
| Loan Application Customer Portal | Customer-facing SPA | 2 owners minimum | User.Read (delegated) | [CTSO-IAM-001 §6] |
| Advisor Portal | Internal web app | 2 owners minimum | User.Read, GroupMember.Read.All (delegated) | [CTSO-IAM-001 §6] |
| Loan Application API | Backend API surface | 2 owners minimum | Custom scopes only | [CTSO-IAM-001 §6] |

All redirect URIs use HTTPS [CTSO-IAM-001 §6]. Client credentials use **certificates** or **federated credentials** — client secrets are last resort [CTSO-IAM-001 §6].

## Identity Monitoring

| Event | Destination | Alert | Standard Reference |
|-------|-------------|-------|-------------------|
| Entra ID sign-in logs | Microsoft Sentinel (real-time) | Risky sign-ins | [CTSO-IAM-001 §9] |
| Entra ID audit logs | Microsoft Sentinel (real-time) | Privilege changes | [CTSO-IAM-001 §9] |
| Service principal sign-ins | Microsoft Sentinel | Anomalous patterns | [CTSO-IAM-001 §9] |
| Identity Protection | Automated | Medium risk → MFA, High risk → Block + reset | [CTSO-IAM-001 §9] |
| PIM activations | Microsoft Sentinel | All activations logged | [CTSO-IAM-001 §4] |

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| Entra ID as sole identity provider | CTSO-IAM-001 | §1 |
| Entra External ID for customer CIAM | CTSO-IAM-001 | §1 |
| OIDC/OAuth 2.0 only | CTSO-IAM-001 | §1 |
| MFA mandatory for employees | CTSO-IAM-001 | §2 |
| Risk-based MFA for customers | CTSO-IAM-001 | §2 |
| Passwordless preferred | CTSO-IAM-001 | §2 |
| RBAC for Azure resources | CTSO-IAM-001 | §3 |
| Claims-based app authorization | CTSO-IAM-001 | §3 |
| PIM for all privileged roles | CTSO-IAM-001 | §4 |
| Managed Identities for services | CTSO-IAM-001 | §5 |
| Workload Identity Federation for CI/CD | CTSO-IAM-001 | §5 |
| App registrations with 2 owners | CTSO-IAM-001 | §6 |
| Conditional Access baseline | CTSO-IAM-001 | §7 |
| Identity logs to Sentinel | CTSO-IAM-001 | §9 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| Identity provider | Entra ID only (no LDAP, no third-party IdP) | Entra ID recommended; federation supported | **Exceeds** (stricter) | [Entra ID overview](https://learn.microsoft.com/entra/fundamentals/whatis) |
| MFA | Mandatory for all employees, risk-based for customers | MFA recommended; risk-based optional | **Exceeds** | [Entra ID MFA](https://learn.microsoft.com/entra/identity/authentication/concept-mfa-howitworks) |
| Passwordless | Preferred (FIDO2, Windows Hello) | Recommended as stronger auth factor | **Aligns** | [Passwordless auth](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-passwordless) |
| PIM | All privileged roles eligible-only | PIM recommended for production | **Aligns** | [PIM overview](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure) |
| Managed Identities | Required for all Azure-to-Azure auth | Recommended over credentials | **Aligns** | [Managed identities](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) |
| Workload Identity Federation | Required for CI/CD (no secrets) | Recommended for GitHub Actions | **Aligns** | [Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation) |
| Conditional Access | Mandatory baseline policies | Recommended security defaults | **Exceeds** (more granular) | [Conditional Access](https://learn.microsoft.com/entra/identity/conditional-access/overview) |
| CIAM | Entra External ID | Entra External ID recommended for B2C scenarios | **Aligns** | [Entra External ID](https://learn.microsoft.com/entra/external-id/external-identities-overview) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| Entra External ID pricing for 10K concurrent customer users | Estimate MAU-based pricing; assess Free tier limits | **Estimate needed** |
| Risk-based MFA may cause friction for legitimate customers | Tune risk detection policies; provide clear MFA enrollment UX | Pre-GA UX review |
| Workload Identity per microservice increases identity management overhead | Automate via Bicep; use naming convention for discoverability | Design decision |
| Quarterly PIM access reviews require process ownership | Assign to Contoso CISO Office as reviewers | **Action required** |
