---
description: "Use when discussing identity, authentication, authorization, Microsoft Entra ID, Azure AD, OAuth, OIDC, MFA, passwordless authentication, RBAC, ABAC, PIM, Privileged Identity Management, managed identities, workload identity federation, service principals, Conditional Access, B2B collaboration, guest access, app registrations, CIAM, Entra External ID, or zero trust identity."
---
# Contoso Identity & Access Management Standards

Read the full Contoso IAM standards before making recommendations:

- [Identity Standards — CTSO-IAM-001](../../../standards/identity.md)

## Key Policies

- Microsoft Entra ID is the sole identity provider; OIDC/OAuth 2.0 only
- MFA mandatory for all employees; passwordless preferred (FIDO2, WHfB)
- RBAC for Azure resources; ABAC for fine-grained data access
- All privileged roles via PIM — no permanent Owner or Global Admin assignments
- PIM activation: max 8 hours, requires MFA + justification
- Managed Identities for Azure service auth; workload identity federation for external
- Service principal secrets discouraged; certificates or federated credentials preferred
- Conditional Access baseline: require MFA, block legacy auth, require compliant devices
- App registrations: minimum 2 owners, least-privilege API permissions, HTTPS redirect URIs
- Entra ID logs and sign-in events to Microsoft Sentinel in real-time
- Access reviews quarterly for privileged roles, monthly for guest users

When making IAM recommendations, cite sections as `[CTSO-IAM-001 §N]`.
