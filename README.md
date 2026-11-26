# AWS Agentic AI Patterns

An enterprise GenAI proof of concept built around realistic **business users, the systems they need, and the security rules that control access**. All operational data and external services are synthetic.

## 1. The end systems

Before looking at AWS components, these are the systems the assistant must work with.

| End system | What it contains or does |
|---|---|
| **Network Operations** | Network assets, status and fault diagnostics |
| **Ticket Management** | Read tickets and create authorised follow-up tickets |
| **Knowledge Repository** | Approved procedures and troubleshooting articles with citations |
| **NetworkMonitor SaaS** | Vendor alerts available through a separately authorised account |
| **Platform Health** | Operational health information about the platform |
| **Network Health Summary** | Read-only, aggregated health reporting for automation |

Network Operations and Ticket Management use a protected **PostgreSQL database** behind their REST APIs. The database is not exposed directly as an AI tool.

## 2. Who can access what?

We use five named people and one automated service. Each has a specific business purpose and permission boundary.

| Persona | Accessible end systems | Restrictions |
|---|---|---|
| **Emma — Service Desk** | Network basic status; Knowledge Repository | No diagnostics or ticket actions |
| **James — VIC Engineer** | Network diagnostics; Ticket Management; Knowledge Repository | VIC records only; tickets require confirmation |
| **Sarah — National Admin** | Network diagnostics; Ticket Management; Knowledge Repository | National permissions; ticket creation requires confirmation |
| **David — External Analyst** | NetworkMonitor SaaS | Must separately authorise his vendor account; no privileged internal access |
| **Oliver — Platform Operations** | Platform Health | No broader Network/Ticket privileges |
| **Network Health Reporter — automation** | Network Health Summary | Aggregate read-only data; no human identity or writes |

### User-to-system overview

```mermaid
flowchart TB
  Emma["Emma · Service Desk"] -->|"Basic status"| Network["Network Operations"]
  Emma --> Knowledge["Knowledge Repository"]
  James["James · VIC Engineer"] -->|"VIC diagnostics"| Network
  James -->|"VIC tickets"| Tickets["Ticket Management"]
  James --> Knowledge
  Sarah["Sarah · National Admin"] -->|"National diagnostics"| Network
  Sarah -->|"National tickets"| Tickets
  Sarah --> Knowledge
  David["David · External Analyst"] -->|"Separate consent"| Vendor["NetworkMonitor SaaS"]
  Oliver["Oliver · Platform Ops"] --> Platform["Platform Health"]
  Reporter["Network Health Reporter"] -->|"Aggregate only"| Summary["Network Health Summary"]
  Network --> RDS[("PostgreSQL")]
  Tickets --> RDS
```

These arrows describe **permitted business access**, not unrestricted direct connections. Identity, tool policies and each protected API must enforce the restrictions.

## 3. Business use cases

| ID | Journey | Desired result |
|---|---|---|
| **UC01** | Emma checks an asset and asks for an outage procedure | Basic status and cited guidance |
| **UC02** | James investigates a VIC network fault | VIC diagnostics and an authorised escalation |
| **UC03** | Sarah reviews national faults and creates a ticket | National visibility and a confirmed, authorised ticket |
| **UC04** | David connects NetworkMonitor | Alerts from his separately authorised vendor account |
| **UC05** | Oliver checks platform health | Platform health information only |
| **UC06** | Reporter produces the daily health summary | Scheduled, read-only aggregate results |

## 4. Architecture patterns

The seven agreed technical patterns explain **how** we deliver and secure those journeys.

| ID | Pattern | Demonstration |
|---|---|---|
| **P01** | Corporate SSO + JWT | Microsoft Entra sign-in and validated API access |
| **P02** | OAuth On-Behalf-Of (OBO) | James's delegated identity reaches the Network API |
| **P03** | IAM + OAuth 2LO | Machine reporter obtains scoped application access |
| **P04** | Third-party OAuth 3LO | David separately authorises NetworkMonitor via Cognito |
| **P05** | MCP Gateway + Cedar | Fine-grained tool and parameter authorisation |
| **P06** | REST APIs + PostgreSQL RLS | VIC versus national data isolation |
| **P07** | Shared RAG retrieval | Approved Knowledge Base excerpts and citations |

## 5. AWS implementation

The design uses **two AgentCore Runtimes and two AgentCore Gateways**: one lane for human JWT callers and another for IAM-authenticated automation. The Gateways expose **eight MCP tools** across the end systems above, with independently configured downstream credentials.

The full AWS architecture, authentication flows, controls and component diagram belong in the [technical architecture](docs/two-gateway-design.md). The reason for the dual inbound-authentication boundaries is recorded in [ADR-001](docs/architecture-decisions.md#adr-001--separate-human-jwt-and-automation-iam-gateways).

### Technical reference

- [Users and permissions](docs/identities-and-permissions.md)
- [Eight MCP tools and downstream API mappings](docs/tool-and-api-inventory.md)
- [2LO, 3LO and OBO architecture notes](docs/oauth-token-flows.md)
- [Architecture decisions](docs/architecture-decisions.md)

**Status:** Design in progress; no infrastructure has been deployed and identity flows are not yet verified.
