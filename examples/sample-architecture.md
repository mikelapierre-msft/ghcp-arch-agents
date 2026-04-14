# Sample Architecture: Contoso Customer Portal (with intentional gaps)

> **Note for demo purposes**: This architecture document contains **intentional deviations** from Contoso standards that the reviewer agent should catch.

## Overview

The Contoso Customer Portal is a web application that allows customers to view their account information, manage preferences, and submit support tickets. It was designed 18 months ago and is due for an architecture review.

## Current Architecture

### Compute

- **Frontend**: React 17 application hosted on **Azure App Service** (Standard S1 tier)
- **Backend API**: .NET 6 Web API hosted on **Azure App Service** (Standard S2 tier)
- **Background Jobs**: .NET 6 Worker Service on a single **Azure Virtual Machine** (Standard D4s v3)
- **Batch Processing**: Python 3.9 scripts running on Azure VM via cron

### Data

- **Primary Database**: Azure SQL Database (Standard S3, single region, single zone)
  - Authentication: SQL authentication (sa account with password stored in appsettings.json)
- **Caching**: Self-hosted Redis on Azure VM (Ubuntu 22.04)
- **Document Storage**: Azure Blob Storage (LRS, public access enabled for customer documents)
- **Search**: Elasticsearch cluster on 3 Azure VMs

### Networking

- App Service has a **public endpoint** (no Private Endpoint)
- Azure SQL Database has a **service endpoint** (not Private Endpoint)
- Backend communicates with VMs over the public internet
- No Azure Firewall — traffic routes directly
- NSG on VM subnet allows inbound SSH (port 22) from `0.0.0.0/0`
- No WAF configured on the App Service

### Security

- TLS 1.2 enabled but **TLS 1.0 not explicitly disabled**
- API keys stored in **environment variables** on App Service
- No container image scanning (not containerized)
- No SAST/DAST in the CI/CD pipeline
- Azure Defender/Microsoft Defender for Cloud **disabled** on the subscription
- No Azure Key Vault — secrets in app settings and appsettings.json

### Identity

- End users authenticate via Entra ID with MFA
- Backend API uses a **service principal with client secret** (18-month expiry) to access Azure SQL
- Background worker VM uses a **local admin account** to run jobs
- No PIM configured — two developers have permanent **Contributor** role on the production subscription
- No Conditional Access policies applied

### Observability

- Application Insights enabled on the App Service (basic)
- No structured logging — uses `Console.WriteLine` for debug output
- No health check endpoints
- No distributed tracing
- VM monitoring via Azure Monitor agent (basic metrics only)

### CI/CD

- GitHub Actions pipeline: Build → Deploy to App Service (single stage)
- No separate environments — deploys directly to production
- No test stage, no security scanning
- Deployment uses `latest` tag on container images
- Single approver for production deployments

### Disaster Recovery

- No DR plan documented
- Azure SQL has no geo-replication
- No backup verification process
- Storage account uses LRS only (no geo-redundancy)
- Estimated RTO: unknown, RPO: unknown

## Known Issues

- Occasional timeout errors when the background VM is under load
- Redis cache on the VM crashed twice in the last quarter
- Customers reported slow page loads during peak hours (no CDN)
