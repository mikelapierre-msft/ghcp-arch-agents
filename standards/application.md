# Contoso Application & Microservices Standards

**Document ID:** CTSO-APP-001  
**Version:** 3.0  
**Last Updated:** 2026-01-05  
**Owner:** Contoso Application Architecture Team  
**Classification:** Internal

## 1. Architecture Patterns

- **Microservices** architecture for new customer-facing applications. Monoliths require ARB exception.
- Domain-Driven Design (DDD) for service boundary identification.
- Event-driven architecture using **Azure Event Hubs** or **Azure Service Bus** for asynchronous communication.
- Synchronous inter-service calls via **gRPC** (preferred) or REST. Direct database sharing between services is **prohibited**.
- API Gateway pattern using **Azure API Management (APIM)** for all external-facing APIs.

## 2. API Standards

- REST APIs must follow **OpenAPI 3.0+** specification. All APIs must have a published OpenAPI document.
- API versioning: **URL path versioning** (e.g., `/api/v1/resource`). Header versioning is not approved.
- Rate limiting and throttling must be enforced at the APIM layer.
- All APIs must return standard error responses following **RFC 9457** (Problem Details for HTTP APIs).
- API response pagination required for list endpoints returning >100 items.

## 3. Technology Stack

| Layer | Approved Technologies |
|---|---|
| Backend services | .NET 8+ (C#), Java 21+ (Spring Boot), Python 3.12+ (FastAPI) |
| Frontend | React 18+, Next.js 14+ |
| Mobile | React Native, .NET MAUI |
| API Gateway | Azure API Management |
| Service Mesh | Istio (on AKS) |
| Messaging | Azure Service Bus, Azure Event Hubs |
| Orchestration | Azure Durable Functions, Temporal (on AKS) |

- **Unapproved**: Node.js for backend services (frontend tooling ok), PHP, Ruby, Go (under evaluation).

## 4. Containerization

- All services must be containerized using **Docker** with multi-stage builds.
- Base images: Contoso-approved base images from internal ACR (based on Microsoft MCR distroless/CBL-Mariner).
- Container images must be **scanned** (Trivy + Microsoft Defender for Containers) before promotion.
- Image tags must be **immutable** — use SHA digests in production manifests. `latest` tag is **prohibited** in production.
- Maximum container image size: **500 MB** (excluding debug images).

## 5. Observability

- All services must implement the **three pillars**: logs, metrics, and distributed traces.
- **OpenTelemetry** SDK for instrumentation (vendor-neutral).
- Export to **Azure Monitor / Application Insights** using the OTLP exporter.
- Structured logging (JSON) with correlation IDs. Unstructured `Console.WriteLine` / `print()` is **prohibited** in production.
- Health check endpoints: `/health/live` (liveness) and `/health/ready` (readiness) are mandatory.

## 6. CI/CD

- All services must have **CI/CD pipelines** in Azure DevOps or GitHub Actions.
- Pipeline stages: Build → Unit Test → SAST/SCA → Container Build → Container Scan → Deploy to Dev → Integration Test → Deploy to QA → DAST → Deploy to Staging → Smoke Test → Deploy to Prod.
- **Blue-green** or **canary** deployments for production. Big-bang deployments are **prohibited**.
- Deployment must be gated by:
  - All tests passing
  - No Critical/High CVEs in container scan
  - Manual approval for production (2 approvers minimum)

## 7. Resiliency Patterns

- **Retry with exponential backoff** for all external calls (Polly for .NET, Resilience4j for Java).
- **Circuit breaker** pattern for downstream service calls.
- **Bulkhead isolation** to prevent cascading failures.
- **Timeout policies**: 30-second maximum for synchronous API calls.
- **Graceful degradation**: services must return partial results rather than failing entirely.

## 8. Configuration Management

- Application configuration via **Azure App Configuration** with feature flags.
- Secrets via **Azure Key Vault** (referenced from App Configuration).
- Environment-specific config must NOT be baked into container images.
- Feature flags for all new features — dark launch before GA.

## 9. Prohibited Practices

- Sharing databases between microservices.
- Direct service-to-service calls without circuit breakers.
- Deploying without health check endpoints.
- Using `latest` tag for container images in production.
- Synchronous chains of more than 3 services deep.
- Storing state in service instances (all services must be stateless).
