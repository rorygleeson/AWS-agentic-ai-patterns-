# Enterprise AI Agent — AWS AgentCore PoC

**One AI chat application. Five human users, one service account, and five end systems.**

The aim is to show how an enterprise AI agent connects to different systems **without giving everybody the same permissions**. Some calls preserve the person's identity; some use a service identity; others require separate third-party consent.

Everything is fictional and uses synthetic data. This is our **architecture design**, not a claim that the PoC is already deployed.

## 1. End systems — what does the agent connect to?

| End system | What it is | What it returns or does |
|---|---|---|
| **Knowledge Base** | Shared **RAG** repository using Amazon Bedrock. | Answers questions using approved troubleshooting guides, with citations. |
| **Network API** | Internal enterprise **REST API**. | Returns asset status, fault diagnostics and a separate aggregate health summary. |
| **Ticket API** | Internal enterprise **REST API**. | Reads existing tickets and creates new ones after confirmation. |
| **NetworkMonitor** | Simulated **third-party SaaS** with its own OAuth login. | Returns alerts belonging to the vendor user who authorized access. |
| **Platform Health API** | AWS API Gateway **REST API** secured with IAM. | Returns synthetic platform-health status. |
| **Supporting database: RDS PostgreSQL** | Private database behind the Network and Ticket APIs. | Stores assets, diagnostics, tickets and permissions. The agent **never accesses RDS directly**. |

## 2. Who can access each system?

Five people share the same AI chat app. One separate service account runs a scheduled report.

| End system | Who? | Allowed access | How enforced |
|---|---|---|---|
| **Knowledge Base (RAG)** | Sarah, James, Emma, David, Oliver | Shared approved guides | **JWT + Cedar**; KB accessed using IAM |
| **Network API** | **Sarah** | All regions: status + diagnostics | **Entra OBO** + API rules + **RLS** |
| ↳ Network API | **James** | **VIC only**: status + diagnostics | **Entra OBO** + API rules + **RLS** |
| ↳ Network API | **Emma** | All regions: **basic status only** | **Entra OBO** + API field restrictions |
| ↳ Network API | **M2M Reporter service account** | **Aggregate summary only** | **G2 IAM + OAuth 2LO** + app permission |
| **Ticket API** | **Sarah** | All regions: read/create | **Entra OBO** + API rules + **RLS** |
| ↳ Ticket API | **James** | **VIC only**: read/create | **Entra OBO** + API rules + **RLS** |
| **NetworkMonitor** | **David only** | His own vendor alerts | **OAuth 3LO** + vendor permissions |
| **Platform Health API** | **Oliver only** | Read-only platform status | **Cedar + IAM/SigV4** |
| **RDS PostgreSQL** | API backends only | Private data, never a direct agent tool | DB service role + **RLS** |

**Default: denied.** If a user isn't listed for a system, they cannot use it. Ticket creation additionally requires confirmation.

**How to read the last column:** JWT/IAM authenticates the caller; Cedar authorizes MCP tool use; OAuth access tokens carry delegated or application permissions to the downstream API; the API's own rules and RLS enforce which records and fields are returned. **OAuth by itself does not restrict James to VIC.**

### What the users ask

| User | Example request |
|---|---|
| **Sarah — National Admin** | “Investigate VIC and NSW faults and create a ticket.” |
| **James — VIC Engineer** | “Diagnose this VIC asset.” An NSW request must fail. |
| **Emma — Service Desk** | “Is this asset up?” She gets basic status only. |
| **David — External Analyst** | “Show my vendor alerts.” |
| **Oliver — Platform Analyst** | “Is the AI platform healthy?” |
| **Reporter — service account** | A scheduled health-summary request, without a human user. |
| **Any of the five humans** | “Find the approved troubleshooting procedure.” (Shared RAG.) |

## 3. AWS architecture

The diagram shows the chat app, Microsoft Entra ID, AWS API Gateway, Lambda, the two AgentCore Runtimes and Gateways, and the five downstream systems.

![AWS AgentCore architecture diagram with AWS service icons](docs/diagrams/aws-agentcore-architecture.svg)

There are **two separate inbound identity boundaries**:

- **Human:** Entra JWT → Runtime U → Gateway G1 (JWT). G1 applies tool authorization and calls the permitted end system.
- **Machine:** IAM service identity → Runtime M → Gateway G2 (IAM). G2 exposes only the reporting tool.

A Gateway has **one inbound authentication type**; individual downstream tool targets can use **different outbound authentication**. That distinction explains why we have two gateways without needing to duplicate every tool or Cedar policy.

## 4. Five main integration patterns

| Pattern | Our example | What it teaches |
|---|---|---|
| **1. OBO — preserve the person** | **James → Network API** | The agent obtains an Entra OAuth token **on behalf of James**. The Network API recognizes James and restricts him to VIC data. |
| **2. IAM service identity** | **Oliver → Platform Health API** | Cedar first authorizes Oliver to invoke the tool. The final REST API call uses the **Gateway's IAM execution role**, not Oliver's OAuth identity. |
| **3. OAuth 2LO — machine to machine** | **Network Health Reporter → Network API** | The service obtains an **app-only OAuth token**; there is no human to impersonate. Only aggregate health is permitted. |
| **4. OAuth 3LO — third-party user consent** | **David → NetworkMonitor** | David separately signs in and authorizes access at the vendor OAuth provider. Only his vendor alerts are returned. |
| **5. Shared RAG** | **Any of the five human users → Knowledge Base** | Everyone searches the same approved knowledge. We deliberately demonstrate user-specific restrictions in other systems instead. |

### A key REST API comparison

We deliberately included **two different ways to register a REST API as AgentCore MCP tools**:

| | **Native AWS API Gateway target** | **OpenAPI target** |
|---|---|---|
| Example | Oliver's Platform Health REST API | James's Network REST API |
| Gateway registration | Existing AWS API Gateway REST **stage** | **OpenAPI specification** describing REST endpoints |
| Outbound authentication | **IAM/SigV4** | **Entra OAuth OBO** |
| Final API sees | Gateway's **service role** | **James's user identity** |
| What it demonstrates | Service execution identity | End-to-end delegated user authorization |

**Both are REST APIs.** OpenAPI is the API *description*, not another running service. Our Ticket API also uses OpenAPI/OBO; the reporter's aggregate Network API uses OpenAPI with OAuth **2LO**.

## Read the deeper design

The README is intentionally simple. The supporting documents hold the technical detail:

- [Architecture baseline](docs/architecture-baseline.md) — full system flows, tools, users, Entra tokens, Cedar and PostgreSQL RLS.
- [Patterns guide](docs/architecture-patterns.md) — the five main integrations **plus supporting platform/security patterns**, with explanations and Q&A.
- [Architecture Decision Register](docs/architecture-decisions.md) — **all 21 agreed ADRs in one document**, with reasons and consequences.
- [Test and evidence plan](docs/test-and-evidence-plan.md) — success, denial and security test cases.
- [Implementation guide](docs/console-implementation-guide.md) — separate instructions for later implementation.

AWS service icons are from [AWS Architecture Icons](https://aws.amazon.com/architecture/icons/), via an [SVG catalogue](https://github.com/phergoualch/aws-icons).
