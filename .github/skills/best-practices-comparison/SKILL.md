---
name: best-practices-comparison
description: "Compare architecture decisions against industry best practices. Use when comparing Contoso standards with Azure Well-Architected Framework, Microsoft Learn guidance, OWASP, CIS Benchmarks, or NIST frameworks. Produces a structured comparison table showing alignment, gaps, and references."
---

# Best Practices Comparison

## When to Use

- Comparing a Contoso architecture decision with industry recommendations
- Validating whether Contoso standards align with or exceed cloud provider guidance
- Researching what Microsoft recommends for a specific architecture pattern
- Preparing justification for why Contoso requires stricter controls than industry defaults

## Procedure

### Step 1: Identify the Architecture Domain

Determine which domains are relevant to the comparison:
- Security → Azure Security Baseline, OWASP, CIS
- Infrastructure → Azure WAF Reliability & Performance pillars
- Networking → Azure networking best practices, hub-spoke reference architecture
- Data → Azure data service best practices, encryption guidance
- Application → Azure WAF Operational Excellence, microservices guidance
- Identity → Microsoft Entra ID best practices, Zero Trust

### Step 2: Query Microsoft Learn via MCP

Use `#tool:mcp_msdocs-mcp-se_microsoft_docs_search` to search for:
- `"Azure Well-Architected Framework {pillar}"` for WAF guidance
- `"Azure {service name} best practices"` for service-specific guidance
- `"Azure {service name} security baseline"` for security baselines
- `"Azure reference architecture {pattern}"` for reference architectures

Follow up with `#tool:mcp_msdocs-mcp-se_microsoft_docs_fetch` on high-value pages to get complete guidance.

### Step 3: Query Non-Microsoft Sources (if needed)

For areas not covered by Microsoft Learn, use web search for:
- **OWASP**: Application security (Top 10, ASVS, API Security)
- **CIS Benchmarks**: Azure CIS benchmark, Kubernetes CIS benchmark
- **NIST**: 800-53 (security controls), 800-63B (digital identity)
- **Cloud Security Alliance (CSA)**: Cloud Controls Matrix

See [curated sources](./references/sources.md) for specific URLs.

### Step 4: Load Contoso Standards

Read the relevant Contoso standard(s) from the `standards/` folder.

### Step 5: Produce Comparison Table

For each comparison area, document:

```markdown
## Best Practices Comparison: {Domain}

| Area | Contoso Requirement | Industry Recommendation | Source | Alignment | Impact |
|------|--------------------|-----------------------|--------|-----------|--------|
| {topic} | {what Contoso requires} | {what industry recommends} | {MS Learn URL or standard} | Exceeds / Aligns / Below | {business impact of gap or surplus} |
```

### Alignment Categories

- **Exceeds**: Contoso is stricter than industry. Document the business justification (regulatory, risk appetite).
- **Aligns**: Contoso matches industry. No action needed.
- **Below**: Contoso is less strict than industry. Flag as a potential risk for ARB review.
- **Differs**: Contoso takes a different approach. Document rationale and trade-offs.

## Output Format

```markdown
# Industry Best Practices Comparison

## Summary
- {N} areas compared
- {X} areas where Contoso exceeds industry
- {Y} areas aligned with industry
- {Z} areas where Contoso differs or falls below

## Comparison Table
{See table format above}

## Notable Differences
### Where Contoso Exceeds Industry
{Explanation and justification}

### Where Contoso Differs from Industry
{Explanation and trade-off analysis}

## References
{List of all MS Learn URLs and external sources consulted}
```
