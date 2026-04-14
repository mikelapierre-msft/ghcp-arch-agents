---
description: "Use when discussing application architecture, microservices, API design, REST, gRPC, OpenAPI, API Management, containerization, Docker, Kubernetes, CI/CD pipelines, observability, OpenTelemetry, Application Insights, health checks, resiliency patterns (retry, circuit breaker, bulkhead), feature flags, App Configuration, technology stack (.NET, Java, Python, React), service mesh, or deployment strategies (blue-green, canary)."
---
# Contoso Application & Microservices Standards

Read the full Contoso application standards before making recommendations:

- [Application Standards — CTSO-APP-001](../../../standards/application.md)

## Key Policies

- Microservices for customer-facing apps; DDD for service boundaries
- Event-driven with Event Hubs / Service Bus; gRPC preferred for sync calls
- API Management (APIM) for all external APIs; OpenAPI 3.0+ specs required
- Approved backend: .NET 8+, Java 21+ (Spring Boot), Python 3.12+ (FastAPI)
- All services containerized with multi-stage Docker builds; Contoso ACR base images
- Image scanning mandatory; `latest` tag prohibited in production
- OpenTelemetry instrumentation; export to Application Insights via OTLP
- Health endpoints mandatory: `/health/live` and `/health/ready`
- CI/CD: Build → Test → SAST/SCA → Container Scan → Deploy; blue-green/canary for production
- Resiliency: retry with exponential backoff, circuit breakers, bulkhead isolation
- Configuration via Azure App Configuration + Key Vault; no config baked into images

When making application architecture recommendations, cite sections as `[CTSO-APP-001 §N]`.
