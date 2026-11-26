# Enterprise AI Agent — AWS AgentCore PoC

Agentic AI introduces a difficult enterprise architecture challenge: **an agent may need to access several systems, but each system can authenticate callers differently and enforce different permissions**. Preserving a person's identity, allowing a service account to act independently, and obtaining third-party consent are not the same problem.

**The objective of this project is to identify, explain and standardise these common authentication and authorization approaches into reusable AgentCore architecture patterns.** We use one fictional enterprise application, four end systems and six callers to show when each pattern applies—and how to prevent unauthorized access.

## 1. End systems

| # | End system | Description | High-level access rule |
|---|---|---|---|
| **1** | **Knowledge Base (RAG)** | Retrieves information from shared troubleshooting documents using Amazon Bedrock Knowledge Bases. | **All human users who successfully sign in to the AI chat app** can search the same approved knowledge. |
| **2** | **Network Operations API** | Read/write enterprise REST API backed by **Amazon RDS PostgreSQL**. Returns asset status, diagnostics and aggregate health, and reads/creates support tickets. | **Only authorized callers.** Human permissions depend on region and operation; the service account has aggregate-only access. |
| **3** | **NetworkMonitor** | Simulated third-party SaaS returning monitoring alerts belonging to an individual vendor user. | **Only users granted tool access who separately authorize their vendor account.** Each user sees only their permitted alerts. |
| **4** | **Platform Health API** | AWS API Gateway REST API returning synthetic AI platform-health information. | **Only users explicitly granted access to this tool.** The downstream API authenticates the Gateway's IAM role. |

### Supporting database — Amazon RDS PostgreSQL

- Stores network assets, diagnostics, tickets and access entitlements.
- Is accessed **only through the Network Operations API**, never directly by the AI agent.
- Uses database permissions and **row-level security (RLS)** to restrict records.

## 2. Users and their access

The application has **five human users** and **one non-human service account**.

| User | Role | Systems they can access | Exact permissions |
|---|---|---|---|
| **Sarah** | National Admin | **RAG, Network API** | Shared knowledge; network status, diagnostics and ticket read/create in **all regions**. |
| **James** | VIC Engineer | **RAG, Network API** | Shared knowledge; network status, diagnostics and ticket read/create for **VIC only**. |
| **Emma** | Service Desk | **RAG, Network API** | Shared knowledge; **basic asset status in all regions**. No detailed diagnostics or sensitive fields. |
| **David** | External Analyst | **RAG, NetworkMonitor** | Shared knowledge; **his own third-party alerts**, after separate vendor authorization. |
| **Oliver** | Platform Analyst | **RAG, Platform Health API** | Shared knowledge; **read-only platform-health information**. |
| **Network Health Reporter** | **M2M Service Account** | **Network API** | **Aggregate network-health summary only**, using its own application identity. No individual asset access. |

### Explicit restrictions

| User | Access that must be denied |
|---|---|
| **Sarah** | NetworkMonitor alerts and Platform Health API. |
| **James** | NSW/other-region records, NetworkMonitor and Platform Health API. |
| **Emma** | Detailed diagnostics, sensitive asset fields, ticket actions, NetworkMonitor and Platform Health API. |
| **David** | Network API and Platform Health API. |
| **Oliver** | Network API and NetworkMonitor. |
| **Network Health Reporter** | RAG, individual asset records, diagnostics, ticket actions, NetworkMonitor, Platform Health and human impersonation. |

**Default rule:** A successful sign-in allows a human to use the AI chat app and shared RAG. Every other system requires its own authorization. If access is not explicitly listed above, it is denied. Ticket creation additionally requires explicit confirmation.


## 3. How these requirements lead to authentication patterns

**Why these use cases?** We selected them to demonstrate **different access and identity patterns**, not simply different business features. Shared RAG shows a resource available to everyone who signs in. Sarah, James and Emma call the **same Network API** using user-delegated OAuth (OBO), but receive different data because their permissions differ. The service account calls that API without a human, using app-only OAuth (2LO). David demonstrates separate consent at a third-party provider (3LO). Oliver demonstrates a different final hop: he is authorized to use the tool, but the Platform Health API receives the **Gateway's IAM identity**. Together, these examples show **when an agent must preserve the user, use an application identity or obtain independent authorization**.

### One API, different permissions

The **Network API** is the clearest example of why identity matters:

| Caller | Request | Result |
|---|---|---|
| **Sarah** | NSW diagnostics | ✅ Allowed |
| **James** | NSW diagnostics | ❌ Denied — VIC only |
| **Emma** | NSW basic asset status | ✅ Allowed — basic fields only |
| **Emma** | NSW diagnostics | ❌ Denied |
| **Service account** | National aggregate health summary | ✅ Allowed — aggregate only |

**Same API, different results:** the system uses each caller's identity and permissions to decide what can be returned.

We cannot use one shared service account for everything: some systems must know **which person** is making the request, some need only **which application**, and a third-party system must obtain **its own user's consent**.

| Access requirement from Sections 1–2 | Pattern we need | What the downstream system knows |
|---|---|---|
| All signed-in humans use the same shared knowledge base. | **Shared RAG access:** corporate sign-in for the chat app; **AWS IAM** for knowledge retrieval. | The AWS service identity. No individual document permissions are needed. |
| Sarah, James and Emma have different permissions for the same Network API, including its ticket endpoints. | **OAuth on-behalf-of (OBO)** with Microsoft Entra ID. | **Which human** called. The API then enforces that person's permissions; RLS restricts rows and the API restricts sensitive fields. |
| The scheduled Network Health Reporter needs aggregate data without a human signing in. | **OAuth 2LO (client credentials)** with Microsoft Entra ID. | **Which application** called, with aggregate-only app permission. No human identity. |
| David accesses data belonging to his account with an external vendor. | **OAuth 3LO (user authorization/consent)** with the vendor's separate OAuth provider. | **Which vendor user** authorized access and which alerts they may see. |
| Oliver alone can call Platform Health, but the final REST API uses an AWS service identity. | **AWS IAM/SigV4** for the downstream call. | The **AgentCore Gateway execution role**, not Oliver. Tool authorization must check Oliver *before* that call. |

### Four terms in plain English

- **OBO:** The agent accesses an API **on behalf of the signed-in person**.
- **2LO:** A **service application** accesses an API without a human.
- **3LO:** A person separately **authorizes access to a third-party account**.
- **IAM:** An **AWS role** is authenticated when calling an AWS-protected service.

### Authentication is not the same as authorization

- **Authentication** establishes the identity of the user, application or AWS role.
- **Authorization** decides which tool, operation, region, record or field that identity may access.
- **OBO preserves user identity; it does not automatically grant all data.** The downstream Network API still enforces their rules.
- **Cedar** is used for restricted AgentCore tool access. **PostgreSQL RLS** enforces appropriate row restrictions behind the enterprise APIs. Shared RAG does not require a special per-user Cedar restriction.

### Two Gateway authentication boundaries

We also agreed on two separate AgentCore entry paths:

- **G1 — human Gateway:** inbound Microsoft Entra JWT for Sarah, James, Emma, David and Oliver.
- **G2 — machine Gateway:** inbound AWS IAM for the Network Health Reporter runtime.

Each Gateway has one inbound authentication method, while **each downstream target can use its own outbound method** from the table above.

### Two ways to register REST APIs as MCP tools

Both systems expose ordinary **REST APIs**. We deliberately register them with AgentCore Gateway in two different ways:

| | Native API Gateway REST-stage target | OpenAPI-defined target |
|---|---|---|
| **Our example** | Platform Health API | Network Operations API |
| **How AgentCore connects** | Import the existing AWS API Gateway REST stage | Supply an OpenAPI description of the REST endpoints |
| **Outbound authentication** | IAM/SigV4 (or API key) | OAuth OBO for humans; OAuth 2LO for the service |
| **Downstream identity** | Gateway's IAM role | Delegated human or application identity |

**Why the difference?** It's a limitation of the **AgentCore target type**, not REST itself. The native AWS API Gateway stage target supports IAM and API-key outbound authentication **but not OAuth**. An **OpenAPI target supports OAuth** through AgentCore's separately configured outbound credential provider—even if the actual REST API is hosted on AWS API Gateway. **The OpenAPI file describes the API; it doesn't provide OAuth or run another service.**

These are our **core integration/authentication patterns**. Detailed session handling, Cedar policies, database controls, logging and implementation decisions belong in the supporting architecture documents—not in the introductory access tables.

## More detail (optional)

- [Architecture baseline](docs/architecture-baseline.md) — complete technical design.
- [Patterns guide](docs/architecture-patterns.md) — detailed authentication flows and examples.
- [Architecture decisions](docs/architecture-decisions.md) — all decisions in one ADR document.
- [Test plan](docs/test-and-evidence-plan.md) — expected success and denial scenarios.
