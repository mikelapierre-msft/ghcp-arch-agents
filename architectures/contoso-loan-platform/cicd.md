# CI/CD Design: Contoso Loan Application Platform

## Pipeline Platform

**GitHub Actions** is selected for CI/CD pipelines, leveraging the team's existing .NET and React expertise. GitHub Actions is approved per Contoso CI/CD standards [CTSO-APP-001 §6], with **Workload Identity Federation** for Azure authentication [CTSO-IAM-001 §5].

## Pipeline Stages

Per [CTSO-APP-001 §6], the pipeline follows a gated, multi-stage promotion model:

```
┌──────────┐   ┌───────────┐   ┌──────────┐   ┌─────────────┐   ┌──────────────┐
│  Build   │──►│ Unit Test │──►│SAST / SCA│──►│Container    │──►│ Container    │
│          │   │           │   │          │   │  Build      │   │   Scan       │
└──────────┘   └───────────┘   └──────────┘   └─────────────┘   └──────┬───────┘
                                                                       │
                                                                       ▼
┌──────────┐   ┌───────────┐   ┌──────────┐   ┌─────────────┐   ┌─────┴────────┐
│Deploy to │◄──│Integration│◄──│Deploy to │◄──│  Gate:      │◄──│ Push to ACR  │
│   QA     │   │   Test    │   │   Dev    │   │ No Crit/High│   │              │
└────┬─────┘   └───────────┘   └──────────┘   └─────────────┘   └──────────────┘
     │
     ▼
┌──────────┐   ┌───────────┐   ┌──────────────┐   ┌────────────────────────────┐
│   DAST   │──►│Deploy to  │──►│ Smoke Test   │──►│ Deploy to Prod             │
│          │   │ Staging   │   │              │   │ (Blue-Green, 2 approvers)  │
└──────────┘   └───────────┘   └──────────────┘   └────────────────────────────┘
```

### Stage Details

| Stage | Tools | Gate Criteria | Standard |
|---|---|---|---|
| **Build** | `dotnet build`, `npm run build` | Compilation succeeds | [CTSO-APP-001 §6] |
| **Unit Test** | xUnit (.NET), Jest (React) | All tests pass; coverage ≥ 80% | [CTSO-APP-001 §6] |
| **SAST** | CodeQL (GitHub Advanced Security) | No Critical/High findings | [CTSO-SEC-001 §5] |
| **SCA** | Dependabot, NuGet Audit, npm audit | No Critical/High CVEs | [CTSO-SEC-001 §5] |
| **Container Build** | Multi-stage Dockerfile | Image size ≤ 500 MB | [CTSO-APP-001 §4] |
| **Container Scan** | Trivy + Microsoft Defender for Containers | No Critical/High CVEs | [CTSO-SEC-001 §5] |
| **Push to ACR** | Docker push to Contoso ACR | SHA digest tag (immutable) | [CTSO-APP-001 §4] |
| **Deploy to Dev** | Helm + kubectl (via GitHub Actions) | Auto-deploy on main branch | [CTSO-INFRA-001 §5] |
| **Integration Test** | .NET integration test suite | All tests pass | [CTSO-APP-001 §6] |
| **Deploy to QA** | Helm + kubectl | Auto-promote from Dev | [CTSO-INFRA-001 §5] |
| **DAST** | OWASP ZAP | No Critical/High findings | [CTSO-SEC-001 §5] |
| **Deploy to Staging** | Helm + kubectl | All DAST results reviewed | [CTSO-INFRA-001 §5] |
| **Smoke Test** | Automated API smoke tests | Health checks pass; key flows succeed | [CTSO-APP-001 §6] |
| **Deploy to Prod** | Blue-green deployment via Istio traffic shifting | **Manual approval (2 approvers)** | [CTSO-APP-001 §6] |

## Deployment Strategy

### Blue-Green Deployments

Per [CTSO-APP-001 §6], big-bang deployments are **prohibited** in production. The platform uses **blue-green deployments** implemented via Istio traffic shifting:

1. Deploy new version to "green" deployment alongside existing "blue"
2. Run automated smoke tests against green
3. Shift 10% traffic to green (canary phase)
4. Monitor for errors/latency regressions (15-minute soak)
5. Shift 100% traffic to green
6. Keep blue running for 1 hour as rollback target
7. Scale down blue

### Rollback

- Instant rollback by shifting Istio traffic back to previous version
- No database rollback needed (forward-only migrations with backward compatibility)
- Maximum rollback time: < 5 minutes

## Infrastructure as Code

### Bicep (Preferred)

Per [CTSO-INFRA-001 §4]:

| Configuration | Value | Standard |
|---|---|---|
| IaC Tool | Bicep | [CTSO-INFRA-001 §4] |
| State Management | Native (Azure Resource Manager) | N/A for Bicep |
| Repository | Co-located in application repo (`infra/` folder) | [CTSO-INFRA-001 §4] |
| Deployment | Via CI/CD pipeline (GitHub Actions) | [CTSO-INFRA-001 §4] |
| Manual provisioning | **Prohibited** in production | [CTSO-INFRA-001 §4] |

### Repository Structure

```
contoso-loan-platform/
├── src/
│   ├── LoanIntakeService/
│   ├── IdentityVerificationService/
│   ├── CreditScoringService/
│   ├── DocumentProcessingService/
│   ├── DecisionEngineService/
│   ├── NotificationService/
│   ├── AdvisorPortalBff/
│   ├── ReportingService/
│   ├── customer-portal/          (React/Next.js)
│   └── advisor-portal/           (React)
├── infra/
│   ├── main.bicep
│   ├── modules/
│   │   ├── aks.bicep
│   │   ├── sql.bicep
│   │   ├── cosmos.bicep
│   │   ├── redis.bicep
│   │   ├── servicebus.bicep
│   │   ├── keyvault.bicep
│   │   ├── apim.bicep
│   │   ├── frontdoor.bicep
│   │   └── monitoring.bicep
│   └── parameters/
│       ├── dev.bicepparam
│       ├── qa.bicepparam
│       ├── staging.bicepparam
│       └── prod.bicepparam
├── charts/                        (Helm charts)
│   ├── loan-intake/
│   ├── identity-verification/
│   └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   └── smoke/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd-dev.yml
│       ├── cd-qa.yml
│       ├── cd-staging.yml
│       └── cd-prod.yml
└── docs/
```

## Authentication for CI/CD

### Workload Identity Federation

Per [CTSO-IAM-001 §5]:

```yaml
# GitHub Actions workflow - OIDC authentication
permissions:
  id-token: write
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

- **No secrets** stored in GitHub; uses OIDC federated credentials [CTSO-IAM-001 §5]
- Separate service principals per environment with least-privilege RBAC
- Service principal secrets are **discouraged** [CTSO-IAM-001 §5]

## Configuration Management

### Azure App Configuration

Per [CTSO-APP-001 §8]:

| Configuration | Implementation |
|---|---|
| App Configuration | Centralized config store per environment |
| Feature Flags | All new features behind feature flags (dark launch) |
| Secrets | Referenced from Azure Key Vault (not stored in App Config) |
| Environment-specific config | Via App Configuration labels; NOT baked into images |

### Feature Flags

Per [CTSO-APP-001 §8], all new features must use **dark launch** via feature flags:

| Feature Flag | Purpose | Default (MVP) |
|---|---|---|
| `EnableAutoDecision` | Automated loan decisions vs. all-manual | Off |
| `EnableOcrProcessing` | OCR document extraction | On |
| `EnableSmsNotifications` | SMS via Twilio | Off (email-only MVP) |
| `EnableMortgagePreApproval` | Mortgage pre-approval flow | Off |

## Environment Strategy

Per [CTSO-INFRA-001 §5]:

| Environment | Auto-Deploy | Approval | Auto-Shutdown | Region |
|---|---|---|---|---|
| Dev | On push to `main` | None | Yes (8AM–6PM ET) | Canada Central |
| QA | On successful Dev deploy | None | Yes (8AM–6PM ET) | Canada Central |
| Staging | On successful QA + DAST | 1 approver | No | Canada Central |
| Production | Manual trigger | 2 approvers | No | Canada Central + Canada East |

Dev/QA environments auto-shutdown outside business hours per [CTSO-INFRA-001 §7].

## Standards Traceability

| Decision | Contoso Standard | Section |
|---|---|---|
| Multi-stage CI/CD pipeline | CTSO-APP-001 | §6 |
| SAST / SCA / DAST in pipeline | CTSO-SEC-001 | §5 |
| Container scan before promotion | CTSO-SEC-001 | §5 |
| Blue-green deployments (no big-bang) | CTSO-APP-001 | §6 |
| 2-approver gate for production | CTSO-APP-001 | §6 |
| Bicep IaC; no manual provisioning | CTSO-INFRA-001 | §4 |
| IaC in application repository | CTSO-INFRA-001 | §4 |
| Workload Identity Federation for CI/CD | CTSO-IAM-001 | §5 |
| Feature flags for all new features | CTSO-APP-001 | §8 |
| Config via Azure App Configuration | CTSO-APP-001 | §8 |
| Secrets via Key Vault references | CTSO-APP-001 §8, CTSO-SEC-001 | §2 |
| No config baked into images | CTSO-APP-001 | §8 |
| Dev/QA auto-shutdown | CTSO-INFRA-001 | §7 |
| Immutable image tags in production | CTSO-APP-001 | §4 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|---|---|---|---|---|
| Pipeline stages | Build → Test → Scan → Deploy (gated) | Similar multi-stage pipeline recommended | **Aligns** | [AKS Baseline DevOps](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| Blue-green / canary | Required for production | Recommended; avoid big-bang deployments | **Aligns** | [Deployment Strategies](https://learn.microsoft.com/azure/architecture/guide/aks/blue-green-deployment-for-aks) |
| OIDC Federation for CI/CD | Required; secrets discouraged | Recommended for GitHub Actions → Azure | **Aligns** | [Workload Identity Federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation) |
| Feature flags | Required for all new features | Recommended via Azure App Configuration | **Aligns** | [Azure App Configuration Feature Flags](https://learn.microsoft.com/azure/azure-app-configuration/concept-feature-management) |
| IaC (Bicep) | Required; no manual provisioning | Bicep recommended as primary Azure IaC | **Aligns** | [Bicep Overview](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) |
| SAST/SCA/DAST | All three mandatory | Defender for DevOps provides integrated scanning | **Aligns** | [Defender for DevOps](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-devops-introduction) |
| Container image scanning | Mandatory before deployment | Recommended via Defender for Containers | **Aligns** | [Container Image Security](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction) |
| Dev/QA auto-shutdown | Required outside business hours | Recommended for cost optimization | **Aligns** | [Cost Management Best Practices](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices) |

## Risks & Open Items

| # | Risk / Open Item | Severity | Mitigation / Action |
|---|---|---|---|
| 1 | GitHub Actions self-hosted runners needed for Private Endpoint access | Medium | Deploy runners in the spoke VNet (snet-devops subnet) |
| 2 | Blue-green deployment requires database schema backward compatibility | Medium | Use expand/contract migration pattern; validate during integration tests |
| 3 | DAST (OWASP ZAP) scan duration may slow QA → Staging promotion | Low | Run DAST asynchronously; gate on results before staging deploy |
| 4 | Feature flag management requires process governance | Low | Define feature flag lifecycle SOP with Product Owner |
| 5 | Helm chart versioning must align with service versions | Low | Use semver for chart versions; automate version bumps in CI |
