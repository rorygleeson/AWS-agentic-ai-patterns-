# Enterprise GenAI Platform on AWS

A reference proof of concept showing how an AWS AgentCore AI agent can access enterprise systems securely—sometimes **on behalf of a person**, sometimes using **a service identity**, and sometimes with **separate third-party authorization**.

All users, business data and APIs are fictional. This repository currently describes the **agreed design**, not a deployed implementation.

## 1. End systems

First, the systems our agent will interact with:

| End system | What it is | What it does / returns |
|---|---|---|
| **Knowledge Repository** | Shared **RAG** repository using Amazon Bedrock Knowledge Bases. | Answers questions using approved operational procedures and troubleshooting guides, with citations. |
| **Network Operations API** | Internal enterprise **REST API** backed by PostgreSQL. | Returns asset status and fault diagnostics. Also provides a separate read-only network health summary for automation. |
| **Ticket Management API** | Internal enterprise **REST API** backed by PostgreSQL. | Retrieves existing tickets and creates new tickets, with authorization and user confirmation. |
| **NetworkMonitor** | Simulated external **third-party SaaS** with its own OAuth authorization server. | Returns alerts belonging to the user's independently authorized vendor account. |
| **Platform Health API** | Small AWS API Gateway **REST API** protected by IAM. | Returns a synthetic, read-only health summary of the AI platform itself. |
| **Amazon RDS PostgreSQL** | Private database **behind** Network Operations and Ticket Management. | Stores assets, diagnostics, tickets and access entitlements. Row-level security restricts data. The agent never connects to RDS directly. |

Each end system has a distinct purpose. The APIs are fictional working examples, not connections to real corporate systems.

## 2. Users, use cases and access

We have **five human users** signing into the same AI chat application and **one non-human service account**.

| User | What they want to do | Systems they can access | What they cannot access |
|---|---|---|---|
| **Sarah — National Admin** | Investigate nationwide network problems and manage support tickets. | **RAG**; Network Operations status and diagnostics across all regions; Ticket Management read/create. | NetworkMonitor's vendor account; Oliver's platform health tool. |
| **James — VIC Engineer** | Diagnose a VIC network fault and raise a ticket. | **RAG**; Network Operations and Ticket Management for **VIC only**. | NSW restricted records, other regions, vendor alerts and platform health. |
| **Emma — Service Desk** | Answer a caller asking whether an asset is working. | **RAG**; **basic** asset status for all regions. | Detailed diagnostics, sensitive fields, ticket changes, vendor alerts and platform health. |
| **David — External Analyst** | Review alerts from his third-party monitoring provider. | **RAG**; his own **NetworkMonitor** alerts after separate authorization. | Internal privileged diagnostics, tickets and platform health. |
| **Oliver — Platform Analyst** | Check whether the AI platform is healthy. | **RAG**; **Platform Health API** only. | Internal diagnostics/tickets and other people's vendor alerts. |
| **Network Health Reporter — service account** | Automatically generate a national network health report. | **Network Operations** aggregate-summary endpoint only. | RAG, individual asset diagnostics, tickets, third-party alerts and user impersonation. |

### Example use cases

| ID | Use case | Expected behaviour |
|---|---|---|
| **UC01** | Any human asks for an approved troubleshooting guide. | Shared RAG returns cited guidance. |
| **UC02** | Sarah investigates VIC and NSW faults and raises a ticket. | National access; ticket created only after confirmation. |
| **UC03** | James investigates a VIC asset, then asks for NSW diagnostics. | VIC allowed; NSW **denied**. |
| **UC04** | Emma checks asset status. | Basic status returned, without sensitive diagnostic information. |
| **UC05** | David asks for vendor alerts. | Requires independent vendor authorization; only his alerts are returned. |
| **UC06** | Oliver checks platform health. | Read-only platform status; the final API is accessed using AWS IAM. |
| **UC07** | Scheduled reporter generates a health summary. | App-only identity; aggregate report, no human data access. |

The main security lesson is simple: **logging into the AI chat app does not mean every user can invoke every tool or see every record.**

## 3. AWS architecture

This diagram shows the agreed components and identity boundaries: the human flow uses Entra ID and a JWT-authenticated AgentCore Runtime/Gateway; the reporter uses a separate IAM-authenticated Runtime/Gateway. The end systems use different outbound authentication methods.

![AWS AgentCore architecture with AWS service icons](docs/diagrams/aws-agentcore-architecture.svg)

The diagram is deliberately a **high-level overview**. The [architecture baseline](docs/architecture-baseline.md) contains the detailed token, Gateway, Cedar and RLS contracts.

## 4. Architecture patterns demonstrated

**A use case describes what someone does. A pattern describes how the technology makes it work.** Multiple use cases can demonstrate the same reusable pattern.

| Pattern | What we demonstrate | Example |
|---|---|---|
| **P01 — Corporate SSO and JWT** | Humans authenticate to the chat app using Microsoft Entra ID. | All five human users. |
| **P02 — JWT identity propagation** | API Gateway and Lambda handle a request without silently replacing its authenticated user with Lambda's IAM role. | James asks for diagnostics. |
| **P03 — Entra OBO** | An OAuth token for a downstream API preserves the **person's delegated identity**. | Network/Ticket API sees James and restricts him to VIC. |
| **P04 — OAuth 2LO** | Client credentials authenticate a workload as an **application**, not a human. | Scheduled Network Health Reporter. |
| **P05 — OAuth 3LO** | A user separately signs in/consents at an **external provider** to access only their own data. | David's NetworkMonitor alerts. |
| **P06 — Native REST API stage target** | AgentCore Gateway registers an AWS API Gateway REST stage directly and accesses it using **IAM/SigV4**. | Oliver's Platform Health API. |
| **P07 — OpenAPI REST target** | AgentCore Gateway uses an **OpenAPI specification** to expose REST operations as MCP tools and supports the appropriate OAuth outbound flow. | James's Network API OBO; reporter's Network API 2LO. |
| **P08 — Human Gateway authorization** | JWT-inbound G1 uses Cedar to control human tool access. | Emma cannot invoke diagnostics. |
| **P09 — Machine Gateway authorization** | Separate IAM-inbound G2 permits the service runtime's narrowly scoped tool. | Reporter can invoke only health summary. |
| **P10 — API authorization and RLS** | Downstream API authorizes the authenticated principal and PostgreSQL restricts rows; API also restricts sensitive fields. | James sees VIC; Emma sees basic fields. |
| **P11 — Shared RAG** | A single approved knowledge repository is available to all five human users. | Troubleshooting guide search. |
| **P12 — Secured AgentCore Runtime ingress** | Human JWT and machine IAM invoke separate secured runtimes. | Lambda → Runtime U; reporter → Runtime M. |
| **P13 — Safe writes** | Authorization, confirmation and idempotency protect state-changing operations. | Sarah or James creates a ticket. |
| **P14 — Audit and least privilege** | Trace the actual caller, Gateway decision, downstream identity and result without logging secrets. | Every tool call. |

### Two REST integration patterns worth comparing

**Both are REST APIs.** The difference is how the API becomes an AgentCore MCP target and how the final request is authenticated.

| | **Native AWS API Gateway REST stage** | **OpenAPI specification target** |
|---|---|---|
| Our example | Oliver → Platform Health | James → Network Diagnostics |
| AgentCore configuration | Register the AWS API Gateway stage | Register an OpenAPI document describing REST operations |
| Outbound authentication | Gateway **IAM/SigV4** | Entra **OAuth OBO** |
| Downstream identity | **Gateway execution role** | **James's delegated user identity** |
| Main lesson | Service identity for the final call | End-to-end human authorization |

The same human Gateway **G1** supports both types of target, because its **inbound JWT authentication** is configured separately from each target's **outbound credentials**. **G2** is separate because its inbound authentication is **AWS IAM** for the machine runtime.

The [full patterns guide](docs/architecture-patterns.md) explains OAuth 2LO/3LO/OBO, Cedar, RLS, tools, trust boundaries and examples in detail.

## Further documentation

- [Architecture baseline](docs/architecture-baseline.md) — full design and tool inventory.
- [Architecture patterns](docs/architecture-patterns.md) — detailed reusable patterns and Q&A.
- [Architecture decision register](docs/architecture-decisions.md) — decisions and rationale in one standalone document.
- [Test and evidence plan](docs/test-and-evidence-plan.md) — planned positive and negative tests.
- [Implementation guide](docs/console-implementation-guide.md) — separate document for later hands-on work.

AWS service icons in the diagram are from [AWS Architecture Icons](https://aws.amazon.com/architecture/icons/), via an [SVG catalogue](https://github.com/phergoualch/aws-icons).
