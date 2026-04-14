# Curated Best Practices Sources

## Microsoft Learn — Azure Well-Architected Framework

| Pillar | URL |
|--------|-----|
| Reliability | https://learn.microsoft.com/azure/well-architected/reliability/ |
| Security | https://learn.microsoft.com/azure/well-architected/security/ |
| Cost Optimization | https://learn.microsoft.com/azure/well-architected/cost-optimization/ |
| Operational Excellence | https://learn.microsoft.com/azure/well-architected/operational-excellence/ |
| Performance Efficiency | https://learn.microsoft.com/azure/well-architected/performance-efficiency/ |

## Microsoft Learn — Reference Architectures

| Pattern | Search Query |
|---------|-------------|
| Microservices on AKS | `"microservices architecture Azure Kubernetes Service"` |
| Web app (multi-region) | `"multi-region web application Azure"` |
| Event-driven | `"event-driven architecture Azure"` |
| Hub-spoke networking | `"hub-spoke network topology Azure"` |
| Zero Trust | `"zero trust architecture Azure"` |
| API Management | `"API Management architecture Azure"` |

## Microsoft Learn — Security Baselines

| Service | Search Query |
|---------|-------------|
| AKS | `"Azure Kubernetes Service security baseline"` |
| Azure SQL | `"Azure SQL Database security baseline"` |
| Azure Storage | `"Azure Storage security baseline"` |
| Key Vault | `"Azure Key Vault security baseline"` |
| App Service | `"Azure App Service security baseline"` |

## External — Industry Standards

| Framework | Focus | URL |
|-----------|-------|-----|
| OWASP Top 10 | Web application security | https://owasp.org/www-project-top-ten/ |
| OWASP API Security | API security risks | https://owasp.org/www-project-api-security/ |
| OWASP ASVS | Application security verification | https://owasp.org/www-project-application-security-verification-standard/ |
| CIS Azure Benchmark | Azure configuration security | https://www.cisecurity.org/benchmark/azure |
| CIS Kubernetes Benchmark | Kubernetes hardening | https://www.cisecurity.org/benchmark/kubernetes |
| NIST 800-53 | Security and privacy controls | https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final |
| NIST 800-63B | Digital identity guidelines | https://pages.nist.gov/800-63-3/sp800-63b.html |
| CSA CCM | Cloud controls matrix | https://cloudsecurityalliance.org/research/cloud-controls-matrix/ |

## How to Use

1. For **Microsoft content**: Use `#tool:mcp_msdocs-mcp-se_microsoft_docs_search` with the search queries above, then `#tool:mcp_msdocs-mcp-se_microsoft_docs_fetch` for full content.
2. For **external content**: Use web search or `#tool:fetch_webpage` to retrieve guidance from the URLs listed.
3. Always **cite the specific URL** in your comparison output.
