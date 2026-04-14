# Contoso Identity & Access Management Standards

**Document ID:** CTSO-IAM-001  
**Version:** 2.6  
**Last Updated:** 2026-01-25  
**Owner:** Contoso Identity & Security Team  
**Classification:** Internal

## 1. Identity Provider

- **Microsoft Entra ID** is the sole approved enterprise identity provider.
- All user authentication must go through Entra ID — no local accounts, LDAP, or third-party IdPs unless federating into Entra ID.
- **Entra External ID** (formerly Azure AD B2C) for customer-facing identity (CIAM).
- SAML and WS-Federation are deprecated. **OIDC/OAuth 2.0** is the approved protocol.

## 2. Authentication

- **Multi-Factor Authentication (MFA)** is mandatory for:
  - All employee access (no exceptions).
  - All administrative and privileged operations.
  - Customer access to sensitive data (risk-based MFA).
- **Passwordless authentication** preferred: FIDO2 keys, Windows Hello for Business, Microsoft Authenticator.
- Password policies (where passwords are still used):
  - Minimum 14 characters, complexity required.
  - Banned password list enabled.
  - No password expiration (per NIST 800-63B) but compromised credential detection enforced.
- Session lifetime: **8 hours** for interactive sessions, **1 hour** for elevated/admin sessions.

## 3. Authorization Model

- **Role-Based Access Control (RBAC)** for Azure resource management.
- **Attribute-Based Access Control (ABAC)** for fine-grained data access (Azure Storage conditions).
- Application-level authorization: **Claims-based** using Entra ID app roles and groups.
- Principle of **least privilege** — no standing access to production resources.

## 4. Privileged Identity Management (PIM)

- All privileged roles must be **eligible**, not permanently assigned.
- PIM activation required for: Owner, Contributor, User Access Administrator, Key Vault Admin, SQL Admin.
- Maximum PIM activation duration: **8 hours**.
- PIM activation requires: MFA + justification + approval for Tier-0 roles.
- Access reviews required **quarterly** for all privileged roles.

## 5. Service Identities

- **Managed Identities** (system-assigned preferred) for Azure-to-Azure service authentication.
- User-assigned managed identities approved for shared identity scenarios (multiple resources, same workload).
- **Workload Identity Federation** for non-Azure workloads (GitHub Actions, Kubernetes external).
- Service principal secrets are **discouraged**. If required: maximum **90-day** expiry, stored in Key Vault.
- **Federated credentials** preferred over secrets for CI/CD pipelines.

## 6. Application Registration

- All applications must be registered in the **Contoso Entra ID tenant**.
- Application owners must be assigned (minimum 2 owners per app registration).
- API permissions must follow least privilege — no broad Graph API permissions without CISO approval.
- Redirect URIs must use HTTPS. `http://localhost` allowed only for local development.
- Client credentials: **certificate** (preferred) or **federated credentials**. Client secrets are last resort.

## 7. Conditional Access Policies

- **Baseline policies** (applied to all users):
  - Require MFA for all cloud apps.
  - Block legacy authentication protocols.
  - Require compliant or Entra ID-joined devices for corporate resources.
- **Elevated policies** (privileged users):
  - Require phishing-resistant MFA (FIDO2 or Windows Hello).
  - GPS/IP-based location restrictions for admin portals.
  - Session controls: sign-in frequency of 1 hour, no persistent browser sessions.

## 8. Guest & External Access

- B2B collaboration governed by **cross-tenant access settings**.
- Guest users: limited to specific Teams, SharePoint sites, and approved applications.
- Guest access reviews required **monthly**.
- External sharing of Confidential/Restricted data requires DLP policy and encryption.

## 9. Monitoring & Incident Response

- Entra ID sign-in and audit logs forwarded to **Microsoft Sentinel** in real-time.
- Risky sign-ins and risky users alerts must trigger automated investigation.
- Identity Protection policies: **Medium risk** = require MFA, **High risk** = block + reset password.
- Service principal sign-in anomalies must be monitored and alerted.

## 10. Prohibited Practices

- Using shared accounts or generic identities (e.g., `admin@contoso.com`).
- Permanently assigning Owner or Global Admin roles.
- Granting `User.ReadWrite.All` or `Directory.ReadWrite.All` Graph API permissions without CISO sign-off.
- Using client secrets for production service principals when certificates or MI are feasible.
- Disabling Conditional Access policies without Change Advisory Board approval.
- Creating app registrations in personal or non-Contoso tenants for corporate workloads.
