---
description: "Review an existing architecture against Contoso standards and industry best practices. Produces a compliance report with gap analysis."
agent: "reviewer"
argument-hint: "Provide the architecture document, description, or file reference to review."
---
Review the provided architecture against all applicable Contoso company standards from the `standards/` folder.

Use the msdocs-mcp-server to compare with Azure Well-Architected Framework guidance and Microsoft Learn best practices.

Produce a structured compliance report including:
- Executive summary with overall compliance posture
- Compliance / gap analysis table with status (✅ ⚠️ ❌) for each area
- Detailed findings grouped by severity (Critical, High, Medium)
- Industry best practices comparison with MS Learn URLs
- Prioritized remediation recommendations

Check all six standard domains: Security, Infrastructure, Networking, Data, Application, and Identity.
