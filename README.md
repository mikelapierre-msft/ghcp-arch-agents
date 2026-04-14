# Contoso Architecture Advisory — GHCP Demo

A demo workspace showcasing **GitHub Copilot customization** (agents, prompts, instructions, skills) as an enterprise architecture advisory system for Contoso. The system grounds AI recommendations in company standards, compares with industry best practices via MCP servers, and supports three architecture workflows.

## What This Demonstrates

| GHCP Primitive | What It Does Here |
|---|---|
| **Workspace Instructions** (`.github/copilot-instructions.md`) | Always-on Contoso architecture advisor persona — every chat interaction is grounded in company standards |
| **Instruction Files** (`.github/instructions/*.instructions.md`) | On-demand loading of domain-specific standards (security, infra, networking, data, app, identity) — only relevant standards are loaded per query |
| **Custom Agents** (`.github/agents/*.agent.md`) | Three specialized agents: Architect, Reviewer, ADR Writer — each with specific tools and workflows |
| **Prompt Files** (`.github/prompts/*.prompt.md`) | Slash-command entry points (`/design-architecture`, `/review-architecture`, `/generate-adr`) that route to the correct agent |
| **Skills** (`.github/skills/best-practices-comparison/`) | Reusable comparison workflow for evaluating Contoso standards against industry best practices |
| **MCP Integration** | `msdocs-mcp-server` for Microsoft Learn lookups; `microsoft/azure-mcp-server` for live Azure resource validation |

## Prerequisites

- **VS Code Insiders** with GitHub Copilot Chat extension
- **MCP servers configured** (user-level `mcp.json`):
  - `msdocs-mcp-server` — Microsoft Learn documentation (HTTP: `https://learn.microsoft.com/api/mcp`)
  - `microsoft/azure-mcp-server` — Azure resource management (optional, for live validation)

### Verify MCP Servers

1. Open VS Code Insiders
2. Open the Output panel → select **MCP Servers** from the dropdown
3. Confirm `msdocs-mcp-server` shows as connected

## Workspace Structure

```
ghcp/
├── .github/
│   ├── copilot-instructions.md          # Always-on persona
│   ├── agents/
│   │   ├── architect.agent.md           # Design new architectures
│   │   ├── reviewer.agent.md            # Review & compliance check
│   │   └── adr-writer.agent.md          # Generate ADRs
│   ├── instructions/
│   │   ├── security-standards.instructions.md
│   │   ├── infrastructure-standards.instructions.md
│   │   ├── networking-standards.instructions.md
│   │   ├── data-standards.instructions.md
│   │   ├── application-standards.instructions.md
│   │   └── identity-standards.instructions.md
│   ├── prompts/
│   │   ├── design-architecture.prompt.md
│   │   ├── review-architecture.prompt.md
│   │   └── generate-adr.prompt.md
│   └── skills/
│       └── best-practices-comparison/
│           ├── SKILL.md
│           └── references/sources.md
├── standards/                            # Contoso company standards
│   ├── security.md          (CTSO-SEC-001)
│   ├── infrastructure.md   (CTSO-INFRA-001)
│   ├── networking.md        (CTSO-NET-001)
│   ├── data.md              (CTSO-DATA-001)
│   ├── application.md       (CTSO-APP-001)
│   └── identity.md          (CTSO-IAM-001)
├── examples/                             # Demo walkthrough materials
│   ├── sample-requirements.md
│   ├── sample-architecture.md
│   └── sample-adr-output.md
└── README.md                             # This file
```

## Demo Walkthroughs

### Scenario 1: Design a New Architecture

**Goal**: Show how the architect agent designs a compliant architecture grounded in Contoso standards with MS Learn references.

1. Open Copilot Chat
2. Select the **architect** agent from the agent picker (or type `/design-architecture`)
3. Attach `examples/sample-requirements.md` as context
4. Prompt: *"Design an architecture for this loan application platform"*

**What to watch for**:
- Agent reads relevant Contoso standards from `standards/` folder
- Agent queries `msdocs-mcp-server` for Azure WAF guidance and reference architectures
- Output includes Standards Traceability table with `[CTSO-XXX-001 §N]` citations
- Output includes Industry Best Practices Comparison with MS Learn URLs
- Architecture respects Canadian data residency, Private Endpoints, mTLS, etc.

### Scenario 2: Review an Existing Architecture

**Goal**: Show how the reviewer agent catches compliance gaps and compares against industry practices.

1. Open Copilot Chat
2. Select the **reviewer** agent (or type `/review-architecture`)
3. Attach `examples/sample-architecture.md` as context
4. Prompt: *"Review this architecture against Contoso standards"*

**What to watch for** (intentional gaps in the sample):
- ❌ Public endpoints instead of Private Endpoints [CTSO-SEC-001 §3]
- ❌ SQL auth with hardcoded credentials [CTSO-DATA-001 §4, CTSO-SEC-001 §2]
- ❌ No Azure Key Vault — secrets in app settings [CTSO-SEC-001 §2]
- ❌ No WAF configured [CTSO-NET-001 §4]
- ❌ TLS 1.0 not disabled [CTSO-SEC-001 §1]
- ❌ Self-hosted Redis on VM instead of Azure Cache for Redis [CTSO-DATA-001 §1]
- ❌ Elasticsearch on VMs instead of Azure AI Search [CTSO-DATA-001 §1]
- ❌ No mTLS for inter-service communication [CTSO-SEC-001 §4]
- ❌ Permanent Contributor roles without PIM [CTSO-IAM-001 §4]
- ❌ No DR plan or geo-replication [CTSO-INFRA-001 §6]
- ❌ Microsoft Defender disabled [CTSO-SEC-001 §8]
- ⚠️ .NET 6 (EOL) instead of .NET 8+ [CTSO-APP-001 §3]
- ⚠️ No SAST/DAST in CI/CD [CTSO-SEC-001 §5]
- ⚠️ Single-stage deployment to production [CTSO-APP-001 §6]

### Scenario 3: Generate an ADR

**Goal**: Show how the ADR writer produces structured decision records with dual traceability.

1. Open Copilot Chat
2. Select the **adr-writer** agent (or type `/generate-adr`)
3. Prompt: *"We need to decide between AKS and Azure Container Apps for hosting the loan application microservices. The team has some Kubernetes experience. We need multi-region DR, service mesh, and compliance with all Contoso standards."*

**What to watch for**:
- Agent reads relevant standards (infrastructure, application, security, networking)
- Agent queries `msdocs-mcp-server` for AKS vs ACA comparison
- ADR includes at least 2 options with pros/cons
- Each option evaluated against Contoso standards explicitly
- Contoso Standards Compliance table at the end
- MS Learn references with URLs
- See `examples/sample-adr-output.md` for expected output format

### Bonus: Best Practices Comparison

1. Open Copilot Chat
2. Type `/best-practices-comparison`
3. Prompt: *"Compare Contoso security standards against Azure security baselines and OWASP"*

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Instruction files for standards** | On-demand discovery means only relevant domain standards load per query (not all 6). Demonstrates GHCP's contextual loading. |
| **Standards as separate files** | Raw standards in `standards/` mirror real-world doc ingestion. Instruction files reference them. Shows how existing company docs can be brought into the system. |
| **MCP-first for industry comparison** | Agents use `msdocs-mcp-server` for authoritative Microsoft Learn content rather than generic web search. More reliable, citable sources. |
| **Three specialized agents** | Each agent has a focused workflow and minimal tool surface. Architect can hand off to Reviewer for validation. |
| **Contoso standards are opinionated** | Standards deliberately exceed industry defaults (mTLS everywhere, 90-day key rotation, no public endpoints) to show comparison value. |

## Extending This Demo

- **Add MCP for Azure DevOps**: Link ADRs to work items via `microsoft/azure-devops-mcp`
- **Add GitHub MCP**: Create GitHub issues from review findings via `github-mcp-server`
- **Agent handoffs**: Architect → Reviewer for automatic "design then validate" workflows
- **Custom MCP server**: Connect to a real standards repository (Confluence, SharePoint, ServiceNow)
- **Additional standards**: Add DevOps, Observability, Cost Management domains
