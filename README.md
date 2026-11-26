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

## 2. Users — what can they do?

All five humans use the same chat app. They can all search the shared RAG knowledge base, but their access to the other systems is different.

| User | Example request | Allowed | Denied or restricted |
|---|---|---|---|
| **Sarah — National Admin** | “Diagnose faults across Australia and raise a ticket.” | RAG; nationwide Network API; Ticket API read/create. | Vendor alerts and Platform Health. |
| **James — VIC Engineer** | “Investigate this Victorian asset. What about NSW?” | RAG; Network and Ticket APIs for **VIC only**. | NSW/other regions, vendor alerts and Platform Health. |
| **Emma — Service Desk** | “Is this asset working?” | RAG; **basic** Network API asset status nationwide. | Diagnostics, sensitive fields, ticket actions and other tools. |
| **David — External Analyst** | “Show my vendor alerts.” | RAG; his own NetworkMonitor alerts after separate OAuth consent. | Internal diagnostics, tickets and Platform Health. |
| **Oliver — Platform Analyst** | “Is the AI platform healthy?” | RAG; **Platform Health API**. | Network diagnostics, tickets and vendor alerts. |
| **Network Health Reporter — service account** | Automatically produce a national health report. | Network API's **aggregate summary endpoint only**. | Human-only tools, raw diagnostics, tickets, RAG and impersonation. |

The key test is not just whether a permitted request succeeds. It is whether **James is denied NSW data, Emma is denied sensitive data, and each identity is restricted to its actual permissions**.

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
