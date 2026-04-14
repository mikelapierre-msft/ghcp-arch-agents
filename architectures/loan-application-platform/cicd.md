# CI/CD — Contoso Loan Application Platform

## Pipeline Strategy

All services use **GitHub Actions** for CI/CD pipelines with the stage gates defined in [CTSO-APP-001 §6]. Production deployments use **blue-green** strategy.

## Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         GitHub Actions Pipeline                         │
├─────────┬──────────┬───────────┬───────────┬──────────┬────────────────┤
│  Build  │   Test   │ Security  │ Container │  Deploy  │   Release      │
│         │          │  Scans    │  Build    │  (Env)   │                │
├─────────┼──────────┼───────────┼───────────┼──────────┼────────────────┤
│ dotnet  │ Unit     │ SAST      │ Docker    │ Dev      │ Staging        │
│ build   │ Tests    │ (CodeQL)  │ Build     │ (auto)   │ (smoke tests)  │
│         │          │           │ Multi-    │          │                │
│ dotnet  │ Code     │ SCA       │ stage     │ QA       │ Production     │
│ restore │ Coverage │(Dependabot│           │ (auto)   │ (blue-green)   │
│         │ >80%     │ + Snyk)   │ Push to   │          │                │
│ React   │          │           │ ACR       │ DAST     │ Manual Approval│
│ build   │          │ Container │ (SHA tag) │ (ZAP)    │ (2 approvers)  │
│         │          │ Scan      │           │          │                │
│         │          │(Trivy +   │           │          │                │
│         │          │ Defender) │           │          │                │
└─────────┴──────────┴───────────┴───────────┴──────────┴────────────────┘
```

## Pipeline Stages

### Stage 1: Build

| Step | Tool | Criteria | Standard Reference |
|------|------|----------|-------------------|
| Restore dependencies | `dotnet restore` / `npm install` | Lock file integrity check | Best practice |
| Compile backend | `dotnet build` | No build errors | — |
| Build frontend | `npm run build` (React) | No build errors | — |
| Generate OpenAPI spec | `dotnet swagger tofile` | OpenAPI 3.0+ valid | [CTSO-APP-001 §2] |

### Stage 2: Test

| Step | Tool | Criteria | Standard Reference |
|------|------|----------|-------------------|
| Unit tests | `dotnet test` / `jest` | All tests pass | [CTSO-APP-001 §6] |
| Code coverage | Coverlet / Istanbul | ≥80% line coverage | Best practice |
| Integration tests | Testcontainers | All tests pass | Best practice |

### Stage 3: Security Scans

| Step | Tool | Blocking Criteria | Standard Reference |
|------|------|-------------------|-------------------|
| SAST | CodeQL (GitHub Advanced Security) | No Critical/High findings | [CTSO-SEC-001 §5] |
| SCA | Dependabot + Snyk | No known Critical/High CVEs | [CTSO-SEC-001 §5] |
| Secrets detection | GitHub Secret Scanning | No exposed secrets | [CTSO-SEC-001 §2] |

### Stage 4: Container Build

| Step | Configuration | Standard Reference |
|------|--------------|-------------------|
| Dockerfile | Multi-stage build | [CTSO-APP-001 §4] |
| Base image | Contoso-approved from internal ACR (CBL-Mariner based) | [CTSO-APP-001 §4] |
| Image tag | Git SHA digest (immutable) | [CTSO-APP-001 §4] |
| Max image size | 500 MB | [CTSO-APP-001 §4] |
| Container scan | Trivy + Microsoft Defender for Containers | No Critical/High CVEs | [CTSO-SEC-001 §5] |
| Push to | Contoso ACR (geo-replicated) | [CTSO-SEC-001 §8] |

**`latest` tag is prohibited in production manifests** [CTSO-APP-001 §4].

### Stage 5: Deploy to Environments

| Environment | Trigger | Approval | Tests After Deploy | Standard Reference |
|-------------|---------|----------|-------------------|-------------------|
| Dev | Auto (on PR merge to `develop`) | None | Smoke tests | [CTSO-INFRA-001 §5] |
| QA | Auto (after Dev success) | None | Integration + DAST | [CTSO-INFRA-001 §5] |
| Staging | Auto (after QA success) | 1 approver | Smoke tests + performance | [CTSO-INFRA-001 §5] |
| Production | Manual trigger | **2 approvers minimum** | Canary validation | [CTSO-APP-001 §6] |

### Stage 6: Production Release (Blue-Green)

```
1. Deploy new version to "green" slot (inactive)
2. Run smoke tests against green endpoint
3. Validate health checks (/health/live, /health/ready)
4. Switch traffic: 10% canary → 50% → 100%
5. Monitor error rates and latency for 15 minutes at each step
6. If healthy: decommission "blue" (previous version)
7. If unhealthy: instant rollback to "blue"
```

Blue-green deployment avoids big-bang releases [CTSO-APP-001 §6].

## Authentication for Pipelines

| Connection | Method | Standard Reference |
|-----------|--------|-------------------|
| GitHub Actions → Azure | Workload Identity Federation (OIDC) | [CTSO-IAM-001 §5] |
| GitHub Actions → AKS | Workload Identity Federation | [CTSO-IAM-001 §5] |
| GitHub Actions → ACR | Managed Identity (via Azure Login) | [CTSO-IAM-001 §5] |

**No service principal secrets** stored in GitHub — all authentication via federated credentials [CTSO-IAM-001 §5].

## Feature Flags

All new features deployed behind feature flags via **Azure App Configuration** [CTSO-APP-001 §8]:

| Flag | Purpose | Default (Prod) |
|------|---------|----------------|
| `EnableMLCreditScoring` | ML-based vs rules-only scoring | Enabled |
| `EnableDocumentOCR` | Automated OCR processing | Enabled |
| `EnableSMSNotifications` | SMS via Twilio | Dark launch |
| `EnableACSmigration` | Azure Communication Services migration | Disabled |
| `EnableNewDecisionRules` | Updated underwriting rules | Canary (10%) |

Feature flags enable **dark launches** before GA [CTSO-APP-001 §8].

## Infrastructure as Code

| Aspect | Configuration | Standard Reference |
|--------|--------------|-------------------|
| IaC tool | Bicep | [CTSO-INFRA-001 §4] |
| State management | N/A (Bicep is declarative/idempotent) | [CTSO-INFRA-001 §4] |
| Storage | Application repository (`/infra` folder) | [CTSO-INFRA-001 §4] |
| Deployment | CI/CD pipeline (GitHub Actions) | [CTSO-INFRA-001 §4] |
| Validation | `az deployment group validate` + What-If | Best practice |
| Portal provisioning | **Prohibited** in production | [CTSO-INFRA-001 §4] |

## Repository Structure

```
loan-application-platform/
├── .github/
│   └── workflows/
│       ├── ci-backend.yml        # Backend service CI
│       ├── ci-frontend.yml       # Frontend CI
│       ├── cd-dev.yml            # Deploy to Dev
│       ├── cd-qa.yml             # Deploy to QA + DAST
│       ├── cd-staging.yml        # Deploy to Staging
│       └── cd-production.yml     # Deploy to Production (blue-green)
├── src/
│   ├── services/
│   │   ├── ApplicationIntake/
│   │   ├── IdentityVerification/
│   │   ├── CreditScoring/
│   │   ├── DocumentProcessing/
│   │   ├── DecisionEngine/
│   │   ├── NotificationService/
│   │   ├── AdvisorPortalBff/
│   │   └── ReportingService/
│   └── frontend/
│       ├── customer-portal/      # React/Next.js
│       └── advisor-portal/       # React
├── infra/
│   ├── main.bicep
│   ├── modules/
│   │   ├── aks.bicep
│   │   ├── sql.bicep
│   │   ├── cosmosdb.bicep
│   │   ├── networking.bicep
│   │   ├── keyvault.bicep
│   │   └── monitoring.bicep
│   └── parameters/
│       ├── dev.bicepparam
│       ├── qa.bicepparam
│       ├── staging.bicepparam
│       └── prod.bicepparam
├── k8s/
│   ├── base/                     # Kustomize base manifests
│   └── overlays/
│       ├── dev/
│       ├── qa/
│       ├── staging/
│       └── prod/
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

## Standards Traceability

| Decision | Contoso Standard | Section |
|----------|-----------------|---------|
| GitHub Actions for CI/CD | CTSO-APP-001 | §6 |
| Build → Test → SAST/SCA → Container → Deploy stages | CTSO-APP-001 | §6 |
| SAST mandatory (CodeQL) | CTSO-SEC-001 | §5 |
| SCA mandatory (Dependabot) | CTSO-SEC-001 | §5 |
| Container scanning (Trivy + Defender) | CTSO-SEC-001 | §5, CTSO-APP-001 §4 |
| Blue-green deployment for production | CTSO-APP-001 | §6 |
| 2 approvers for production | CTSO-APP-001 | §6 |
| No Critical/High CVEs in production | CTSO-SEC-001 | §5 |
| SHA digest tags (no `latest`) | CTSO-APP-001 | §4 |
| Multi-stage Docker builds | CTSO-APP-001 | §4 |
| Contoso ACR base images only | CTSO-APP-001 | §4, CTSO-SEC-001 §8 |
| Feature flags via App Configuration | CTSO-APP-001 | §8 |
| Bicep for IaC | CTSO-INFRA-001 | §4 |
| No portal provisioning in prod | CTSO-INFRA-001 | §4 |
| Workload Identity Federation for pipelines | CTSO-IAM-001 | §5 |

## Industry Best Practices Comparison

| Area | Contoso Requirement | MS Learn Recommendation | Alignment | Reference |
|------|--------------------|-----------------------|-----------|-----------|
| CI/CD pipeline stages | 6 mandatory stages with security gates | Recommended DevSecOps pipeline | **Aligns** | [CI/CD for microservices](https://learn.microsoft.com/azure/architecture/microservices/ci-cd) |
| Blue-green deployment | Mandatory for production | Recommended for zero-downtime releases | **Aligns** | [Deployment strategies](https://learn.microsoft.com/azure/architecture/framework/devops/release-engineering-cd) |
| Container scanning | Mandatory with Critical/High blocking | Microsoft Defender for Containers recommended | **Aligns** | [Defender for Containers](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction) |
| IaC | Bicep mandatory; portal prohibited | Bicep recommended for Azure-native IaC | **Aligns** | [Bicep documentation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) |
| Feature flags | Azure App Configuration mandatory | Recommended for progressive exposure | **Aligns** | [App Configuration feature flags](https://learn.microsoft.com/azure/azure-app-configuration/concept-feature-management) |
| Pipeline auth | Workload Identity Federation (no secrets) | Recommended for GitHub Actions | **Aligns** | [OIDC for GitHub Actions](https://learn.microsoft.com/entra/workload-id/workload-identity-federation-create-trust-github) |
| 2-approver gate | Mandatory for production | Recommended for production environments | **Aligns** | [Release gates](https://learn.microsoft.com/azure/devops/pipelines/release/approvals/gates) |
| Image tagging | SHA digest mandatory | Recommended for reproducibility | **Exceeds** (strict enforcement) | [ACR best practices](https://learn.microsoft.com/azure/container-registry/container-registry-best-practices) |

## Risks & Open Items

| Risk | Mitigation | Status |
|------|-----------|--------|
| Pipeline execution time may exceed 30 min with all security scans | Parallelize SAST/SCA/container scan stages; cache dependencies | Optimize in sprint 2 |
| DAST (ZAP) may produce false positives blocking QA → Staging promotion | Maintain ZAP baseline exceptions file; review weekly | Ongoing |
| Blue-green requires double AKS capacity during deployment | Use Kubernetes rolling deployment with maxSurge for non-critical services; true blue-green for critical path only | Design decision |
| Self-hosted build agents needed for private AKS access | Deploy build agents in `snet-devops-agents` subnet within spoke VNet | Infrastructure setup |
