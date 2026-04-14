# Sample Requirements: Contoso Loan Application Platform

## Business Context

Contoso Financial Services is building a new **digital loan application platform** that allows retail banking customers to apply for personal loans, auto loans, and mortgage pre-approvals through web and mobile channels.

## Functional Requirements

1. **Loan Application Intake**: Customers submit loan applications with personal information, employment details, financial history, and supporting documents.
2. **Identity Verification**: Integration with third-party identity verification service (Equifax) and Contoso's internal KYC system.
3. **Credit Scoring**: Automated credit scoring using an internal ML model hosted on Azure. Fallback to manual underwriting queue.
4. **Document Processing**: OCR-based extraction from uploaded documents (pay stubs, tax returns, ID scans).
5. **Decision Engine**: Rules-based + ML-assisted loan approval/decline/refer decisions.
6. **Notification Service**: Email and SMS notifications for application status updates.
7. **Advisor Portal**: Internal web application for loan advisors to review referred applications and override decisions.
8. **Reporting Dashboard**: Real-time dashboards for loan pipeline, approval rates, and SLA tracking.

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | 99.95% uptime |
| Response time (API) | P95 < 500ms for read operations, P95 < 2s for submissions |
| Concurrent users | Up to 10,000 simultaneous applicants during peak |
| Data retention | 7 years for loan applications (regulatory) |
| Data classification | **Confidential** (PII + financial data) |
| Regulatory | PIPEDA, OSFI guidelines, anti-money laundering (AML) |
| Primary region | Canada Central |
| DR requirement | RPO ≤ 30 min, RTO ≤ 2 hours |

## Integration Points

- **Equifax API**: Credit checks and identity verification (external, HTTPS)
- **Contoso Core Banking**: Loan disbursement and account creation (on-premises, via ExpressRoute)
- **Contoso KYC Service**: Existing internal service running on AKS
- **Azure AI Services**: Document Intelligence for OCR, Azure ML for credit scoring
- **SendGrid**: Email delivery (via Azure Communication Services as future migration)
- **Twilio**: SMS notifications (via Azure Communication Services as future migration)

## Stakeholders

- Product Owner: VP Retail Lending
- Architect: Senior Solution Architect, Digital Banking
- Security: Contoso CISO Office
- Compliance: Regulatory Compliance Team
- Operations: Cloud Platform Team

## Constraints

- Must comply with all Contoso architecture standards
- Must integrate with existing Contoso Azure Landing Zone
- Team has strong .NET and React experience; limited Java experience
- Budget: approximately $15K/month Azure spend for production
- Timeline: MVP in 4 months, GA in 6 months
