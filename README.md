# Enterprise GenAI Platform on AWS

A fictional, **console-first AWS AgentCore proof of concept** exploring how AI agents securely access enterprise systems. Start with the business story. The AWS architecture and identity mechanics come later.

> **Design baseline, not a deployed solution.** All users, records, endpoints and vendor applications are fictional. No organization-specific product is assumed.

## 1. The end systems — what can the agent actually access?

Each end system has a useful business purpose and a clearly defined interface.

| End system | Type | Example API/tool | What the user gets |
|---|---|---|---|
| **1. Knowledge Repository** | Shared **RAG** knowledge base (Amazon Bedrock Knowledge Bases) | `search_knowledge(query)` | Cited troubleshooting guides and operating procedures. **Every human user** can search it. |
| **2. Network Operations** | **Internal enterprise REST API**, backed by Amazon RDS PostgreSQL | `GET /assets/{id}/status` → `get_asset_status`; `GET /assets/{id}/diagnostics` → `get_diagnostics`; `GET /network/health-summary` | Asset state, restricted diagnostic details, or aggregate national health. User/region/field permissions apply. |
| **3. Ticket Management** | Separate **internal enterprise REST API**, backed by RDS | `GET /tickets/{id}` → `get_ticket`; `POST /tickets` → `create_ticket` | Read or create a regional support ticket. Creation needs explicit confirmation. |
| **4. NetworkMonitor** | Simulated **third-party SaaS API** with its **own OAuth server** (Amazon Cognito) | `GET /alerts` → `search_vendor_alerts` | Alerts belonging to the individual vendor account after separate login/consent. |
| **5. Platform Health** | AWS API Gateway **native REST API stage**, protected by IAM/SigV4 | `GET /platform/health` → `get_platform_health` | Synthetic platform service status—not network-customer data. |
| **Supporting database** | Private **Amazon RDS PostgreSQL**, accessed by application APIs only | SQL with PostgreSQL row-level security (RLS) | Asset, diagnostic, ticket and entitlement records. **The agent never connects directly to the database.** |

**Important:** All four API-backed systems are REST/HTTP services. **OpenAPI is the specification used to register some REST operations as MCP tools; it is not another running API.**

## 2. The use cases — who wants to do what?

| Use case | Caller | User story | End system and expected result |
|---|---|---|---|
| **UC00 — Knowledge search** | **All five human users** | “Find the approved troubleshooting procedure for a network outage.” | **Knowledge Repository (RAG):** cited answers from the shared approved documents. |
| **UC01 — National investigation** | **Sarah**, National Admin | “Investigate faults across VIC and NSW and create a support ticket.” | **Network Operations + Ticket Management:** nationwide diagnostics and authorized, confirmed ticket creation. |
| **UC02 — Regional engineering** | **James**, VIC Engineer | “Diagnose this Victorian asset and raise a ticket. Can I also inspect NSW?” | **Network Operations + Ticket Management:** VIC allowed; NSW **denied**. |
| **UC03 — Service desk enquiry** | **Emma**, Service Desk | “Is this network asset up?” | **Network Operations:** national **basic status only**; diagnostics and restricted fields **denied**. |
| **UC04 — External vendor alerts** | **David**, External Analyst | “Show my NetworkMonitor alerts.” | **NetworkMonitor SaaS:** independently authorized 3LO access to David's vendor records. |
| **UC05 — Platform operations** | **Oliver**, Platform Analyst | “Is the AI platform healthy?” | **Platform Health REST API:** read-only synthetic platform status via native API Gateway stage target. |
| **UC06 — Scheduled reporting** | **Network Health Reporter**, service identity | “Generate the national health summary.” | **Network Operations aggregate REST endpoint:** app-only, read-only report via 2LO. |

## 3. Who can access each system?

| Caller | Shared RAG | Network status | Diagnostics | Ticket read/create | Vendor alerts | Platform health |
|---|---|---|---|---|---|---|
| **Sarah** | Yes | National | National | National / confirmed | No | No |
| **James** | Yes | VIC only | VIC only | VIC / confirmed | No | No |
| **Emma** | Yes | National **basic fields** | No | No | No | No |
| **David** | Yes | No | No | No | **Own vendor account** | No |
| **Oliver** | Yes | No | No | No | No | **Yes** |
| **M2M Reporter** | No | **Aggregate health only** | No | No | No | No |

The reporter's “Network status” entry means the **separate aggregate summary operation**, not ordinary asset status or diagnostics. **Signing into the chat app never grants automatic access to every tool.**

## 4. The patterns — what does each use case teach?

A **use case** is what the caller does. A **pattern** is the reusable way we authenticate, authorize and connect systems. One use case can use several patterns.

| Key pattern | In plain English | Used by |
|---|---|---|
| **Corporate SSO + JWT** | Humans log into the web chat using Entra ID; their signed access token accompanies requests. | UC00–UC05 |
| **JWT preserved through Lambda** | Lambda validates/audits the request and forwards the original human JWT. Its IAM role **does not impersonate** the user. | UC00–UC05 |
| **Entra OBO (on-behalf-of)** | The downstream Network or Ticket API receives a **new, API-audience token representing the same human**. The API and PostgreSQL RLS check that person's entitlements. | UC01–UC03 |
| **OAuth 2LO (client credentials)** | A machine gets an **app-only token**; there is no end user. | UC06 |
| **OAuth 3LO (authorization code + consent)** | David separately authorizes the external vendor application to retrieve *his* alerts. | UC04 |
| **Native AWS API Gateway REST stage target** | AgentCore registers an AWS API Gateway stage directly; outbound auth uses **Gateway IAM/SigV4**, not Oliver's OAuth identity. | UC05 |
| **OpenAPI-defined REST target** | AgentCore imports an **OpenAPI specification describing existing REST endpoints**; outbound OAuth supports delegated access to Network/Ticket APIs. | UC01–UC03 and UC06 (app-only) |
| **Cedar + downstream enforcement** | Gateway decides whether the agent may call a tool; downstream API independently checks the delegated/app token; RLS and response DTOs protect data. | All authorized API flows |
| **Two Gateway identity boundaries** | G1 accepts **human JWT**; G2 accepts **AWS IAM** from machine Runtime M. Each has one inbound authorizer; targets have their own outbound authentication. | UC00–UC06 |

The [full catalogue](docs/architecture-patterns.md) defines **14 reusable patterns**, including session security, shared RAG, write confirmation and audit.

### The deliberate REST comparison

This comparison was a key reason for adding Oliver's user and tool.

| | **Native AWS API Gateway stage target** | **OpenAPI specification target** |
|---|---|---|
| Example user | **Oliver** | **James** |
| Underlying service | Platform Health **REST API** | Network Operations **REST API** |
| Registered with AgentCore as | Existing AWS API Gateway REST **stage** | **OpenAPI document** describing REST operations |
| Outbound authentication | AWS IAM/SigV4 | Microsoft Entra OAuth **OBO** |
| Who the backend recognizes | Gateway execution **IAM role** | **James**, using delegated OAuth |
| Data authorization | Cedar checks Oliver before the call; backend checks IAM | Cedar checks James; API enforces James's regional permissions and **RLS** |
| What it proves | **Service identity** across the final hop | **Human identity preservation** across the final hop |

The native stage target doesn't support OAuth outbound in the documented mode, which is why the OBO-protected Network/Ticket API operations use OpenAPI targets. The **same Gateway G1** can host both target styles, because its inbound JWT setting is separate from each target's outbound credentials.

## 5. The AWS architecture

Now that the end systems, users and patterns are clear, here is how the components connect. **AWS service icons are embedded in the diagram**; there are no Mermaid diagrams.

![AWS AgentCore reference architecture with AWS service icons, human and machine flows](docs/diagrams/aws-agentcore-architecture.svg)

- **Human:** Web chat + Entra ID → AWS API Gateway → Lambda → JWT AgentCore Runtime U → JWT Gateway G1 → permitted MCP tool and end system.
- **Machine:** EventBridge Scheduler → Reporter Lambda → IAM Runtime M → IAM Gateway G2 → app-only 2LO Network summary.
- **Database:** RDS is behind the enterprise APIs, not exposed as an MCP tool.
- **Independent third party:** NetworkMonitor has its own Cognito OAuth provider and per-user vendor permissions.

## 6. Documentation

| Document | Why it exists |
|---|---|
| [Architecture baseline](docs/architecture-baseline.md) | Complete system boundaries, personas, tool inventory, token contracts and RLS controls. |
| [Architecture patterns](docs/architecture-patterns.md) | Fourteen patterns, detailed flows, native REST versus OpenAPI comparison, and Q&A. |
| [Architecture decisions](docs/architecture-decisions.md) | A standalone **single ADR register** preserving decisions and rationale. |
| [Console implementation guide](docs/console-implementation-guide.md) | Separate step-by-step AWS/Entra **console setup**, with no IaC. |
| [Test and evidence plan](docs/test-and-evidence-plan.md) | Planned success, denial, bypass, token and data-isolation tests—**not yet executed**. |

The architecture is deliberately **generic**, without internal company-product names. Real customer data, real integrations and production-readiness claims are out of scope.

### Icon attribution

AWS product/service symbols in the diagram are from the [AWS Architecture Icons](https://aws.amazon.com/architecture/icons/) set (retrieved via the [SVG catalogue](https://github.com/phergoualch/aws-icons)); AWS marks remain the property of Amazon Web Services.
