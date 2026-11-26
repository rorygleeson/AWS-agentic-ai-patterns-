# Enterprise GenAI Platform on AWS

**A reference architecture and hands-on proof of concept for secure agent access to enterprise systems.**

This repository documents a deliberately small fictional network-operations enterprise. It illustrates *who can access which end system*, then explains the AWS AgentCore, MCP, OAuth, IAM, Cedar, REST and database patterns that make those permissions enforceable. The design is generic and is not associated with a particular organisation.

> **Documentation status:** Architecture baseline and patterns are design specifications. The AWS/Entra environment is not yet deployed. Examples and results are synthetic; no operational health metrics are claimed.

![AWS AgentCore reference architecture showing the human and machine flows](docs/diagrams/aws-agentcore-architecture.svg)

The numbered callouts are explained in [Architecture patterns](docs/architecture-patterns.md); this diagram distinguishes the inbound JWT/IAM boundaries from each target's outbound authentication.

## 1. Meet the end systems first

| End system | What it is | What it returns or changes |
|---|---|---|
| **Shared Knowledge Repository** | Bedrock Knowledge Base with approved fictional operational guides (RAG). | Cited troubleshooting guidance and procedures; all five humans may search it. |
| **Network Operations API** | Internal OAuth-protected REST API backed by RDS PostgreSQL. | `get_asset_status` (basic status), `get_diagnostics` (restricted faults), and a restricted aggregate health report. |
| **Ticket Management API** | Separate internal OAuth-protected REST API backed by RDS. | `get_ticket` and `create_ticket`, with explicit confirmation and idempotency. |
| **NetworkMonitor** | Fictional external SaaS with its own Cognito OAuth authorization server. | `search_vendor_alerts`: alerts visible to the separately authorized vendor user. |
| **Platform Health API** | Small AWS API Gateway REST API protected by IAM/SigV4. | `get_platform_health`: synthetic status of platform services. |
| **PostgreSQL** | Private RDS database behind the Network and Ticket APIs; *not* a direct agent tool. | Assets, diagnostics, tickets, principals and entitlements; RLS constrains records. |

The two REST-to-MCP approaches are intentional: the Network/Ticket APIs use **OpenAPI targets + OAuth OBO**; the Platform Health API uses a **native API Gateway stage target + IAM**.

## 2. Meet the callers

| Caller | Business task | Allowed | Denied or limited |
|---|---|---|---|
| **Sarah — National Admin** | Investigate nationwide faults and manage tickets. | Shared RAG; national asset status, diagnostics, authorized tickets. | David's separately authorized vendor account; Oliver-only platform health. |
| **James — VIC Engineer** | Diagnose a Victorian network fault and raise a ticket. | Shared RAG; VIC status, diagnostics and VIC ticket actions. | NSW restricted records, Oliver-only platform health. |
| **Emma — Service Desk** | Answer routine outage enquiries. | Shared RAG; nationwide **basic** asset-status fields. | Diagnostics, ticket creation, sensitive fields, Oliver-only platform health. |
| **David — External Analyst** | Search his NetworkMonitor alerts after separate consent. | Shared RAG; his external vendor alerts via 3LO. | Privileged internal assets, diagnostics and ticket actions. |
| **Oliver — Platform Analyst** | Check platform health. | Shared RAG and `get_platform_health`. | Internal network diagnostics/tickets and other users' vendor data. |
| **Network Health Reporter — service** | Produce a scheduled aggregate health summary. | Read-only aggregate via IAM Runtime/Gateway and OAuth 2LO. | Human delegated tokens, diagnostics and writes. |

A successful login grants **chat access**, not universal tool or data access.

## 3. How the design is organized

- **Architecture Baseline:** [docs/architecture-baseline.md](docs/architecture-baseline.md) — systems, personas, topology, trust boundaries, contracts and requirements.
- **Patterns Guide:** [docs/architecture-patterns.md](docs/architecture-patterns.md) — the 14 agreed patterns, comparison tables and practical Q&A.
- **Architecture Decision Register:** [docs/architecture-decisions.md](docs/architecture-decisions.md) — the accepted decisions, reasons and consequences in a **single document**.
- **Implementation Guide:** [docs/console-implementation-guide.md](docs/console-implementation-guide.md) — a **separate**, console-first build sequence; no IaC required.
- **Test and Evidence Plan:** [docs/test-and-evidence-plan.md](docs/test-and-evidence-plan.md) — positive, negative and bypass scenarios.

## 4. Key design decisions in one glance

**Two AgentCore Runtimes:** Runtime U uses Entra JWT inbound; Runtime M uses AWS IAM inbound.

**Two AgentCore Gateways:** G1 uses `CUSTOM_JWT` inbound for humans; G2 uses `AWS_IAM` inbound for the reporter. **Each Gateway has exactly one inbound authorizer type; each target independently configures outbound credentials.**

**Two API integration methods:**
1. Oliver → G1/Cedar → **native API Gateway stage target** → Gateway IAM/SigV4 → Platform Health REST API. The backend sees the Gateway role, *not Oliver*.
2. James → G1/Cedar → **OpenAPI target** → Entra OBO → Network REST API → PostgreSQL RLS. The backend sees **James's delegated identity**.

**Three OAuth patterns:** OBO for user delegation, 2LO for a workload, 3LO for David's independent SaaS account consent.

**No silent privilege translation:** The human JWT never becomes a claimed human identity merely because Lambda or Gateway possesses an IAM role.

## 5. Architecture at a glance

The interactive path is **Web Chat → Entra login → API Gateway JWT authorizer → proxy Lambda → JWT Runtime U → JWT Gateway G1 → MCP tools → protected systems**.

The automated path is **EventBridge Scheduler → Reporter Lambda → IAM Runtime M → IAM Gateway G2 → restricted aggregate API using service 2LO**.

G1 exposes shared RAG, internal Network/Ticket tools, external NetworkMonitor and Oliver's platform health tool; G2 exposes only the reporter's aggregate operation. This deliberate separation avoids copying human Cedar policies into G2.

See the baseline for the detailed trust boundaries and end-to-end token contracts. **No ASKnbn or employer-specific assumptions are needed.**

## 6. What the repository demonstrates

- Entra authentication versus authorization; JWT audiences/scopes/app roles.
- User-token forwarding, OBO, client credentials 2LO and independent vendor 3LO.
- AgentCore Runtime, Gateway, Identity, MCP and OpenAPI target contracts.
- Gateway tool policies (Cedar) plus downstream API entitlements and PostgreSQL RLS.
- IAM service authorization versus end-to-end delegated user authorization.
- Shared RAG, safe ticket writes, session isolation and audit correlation.
- Denial scenarios designed alongside successful business use cases.

**Repository scope:** Reference architecture, documentation and future sample code. Real customer data, real network access and production-readiness claims are explicitly excluded.
