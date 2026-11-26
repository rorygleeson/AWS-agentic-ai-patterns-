# Enterprise AI Agent — AWS AgentCore PoC

Agentic AI introduces a difficult enterprise architecture challenge: **an agent may need to access several systems, but each system can authenticate callers differently and enforce different permissions**. Preserving a person's identity, allowing a service account to act independently, and obtaining third-party consent are not the same problem.

**The objective of this project is to identify, explain and standardise these common authentication and authorization approaches into reusable AgentCore architecture patterns.** We use one fictional enterprise application, five end systems and six callers to show when each pattern applies—and how to prevent unauthorized access.

## 1. End systems

| # | End system | Description | High-level access rule |
|---|---|---|---|
| **1** | **Knowledge Base (RAG)** | Retrieves information from shared troubleshooting documents using Amazon Bedrock Knowledge Bases. | **All human users who successfully sign in to the AI chat app** can search the same approved knowledge. |
| **2** | **Network Operations API** | Enterprise REST API that retrieves network asset status, diagnostics and aggregate health information from **Amazon RDS PostgreSQL**. | **Only authorized users.** Access depends on the user's permissions and region. The service account has separate aggregate-only access. |
| **3** | **Ticket Management API** | Enterprise REST API that reads and creates support tickets stored in **Amazon RDS PostgreSQL**. | **Only authorized users.** Permissions determine which tickets they can read or create and in which regions. |
| **4** | **NetworkMonitor** | Simulated third-party SaaS returning monitoring alerts belonging to an individual vendor user. | **Only users granted tool access who separately authorize their vendor account.** Each user sees only their permitted alerts. |
| **5** | **Platform Health API** | AWS API Gateway REST API returning synthetic AI platform-health information. | **Only users explicitly granted access to this tool.** The downstream API authenticates the Gateway's IAM role. |

### Supporting database — Amazon RDS PostgreSQL

- Stores network assets, diagnostics, tickets and access entitlements.
- Is accessed **only through the Network and Ticket application APIs**, never directly by the AI agent.
- Uses database permissions and **row-level security (RLS)** to restrict records.

## 2. Users and their access

The application has **five human users** and **one non-human service account**.

| User | Role | Systems they can access | Exact permissions |
|---|---|---|---|
| **Sarah** | National Admin | **RAG, Network API, Ticket API** | Shared knowledge; network status and diagnostics in **all regions**; read/create tickets nationally. |
| **James** | VIC Engineer | **RAG, Network API, Ticket API** | Shared knowledge; network status, diagnostics and tickets for **VIC only**. |
| **Emma** | Service Desk | **RAG, Network API** | Shared knowledge; **basic asset status in all regions**. No detailed diagnostics or sensitive fields. |
| **David** | External Analyst | **RAG, NetworkMonitor** | Shared knowledge; **his own third-party alerts**, after separate vendor authorization. |
| **Oliver** | Platform Analyst | **RAG, Platform Health API** | Shared knowledge; **read-only platform-health information**. |
| **Network Health Reporter** | **M2M Service Account** | **Network API** | **Aggregate network-health summary only**, using its own application identity. No individual asset access. |

### Explicit restrictions

| User | Access that must be denied |
|---|---|
| **Sarah** | NetworkMonitor alerts and Platform Health API. |
| **James** | NSW/other-region records, NetworkMonitor and Platform Health API. |
| **Emma** | Detailed diagnostics, sensitive asset fields, Ticket API, NetworkMonitor and Platform Health API. |
| **David** | Internal Network API, Ticket API and Platform Health API. |
| **Oliver** | Network API, Ticket API and NetworkMonitor. |
| **Network Health Reporter** | RAG, individual asset records, diagnostics, tickets, NetworkMonitor, Platform Health and human impersonation. |

**Default rule:** A successful sign-in allows a human to use the AI chat app and shared RAG. Every other system requires its own authorization. If access is not explicitly listed above, it is denied. Ticket creation additionally requires explicit confirmation.


## 3. How these requirements lead to authentication patterns

The tables above describe **what access the business needs**. They also explain **why we need several different authentication patterns**.

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
| Sarah, James and Emma get different Network API data; Sarah and James have different Ticket API permissions. | **OAuth on-behalf-of (OBO)** with Microsoft Entra ID. | **Which human** called. The API then enforces that person's permissions; RLS restricts rows and the API restricts sensitive fields. |
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
- **OBO preserves user identity; it does not automatically grant all data.** The downstream Network/Ticket APIs still enforce their rules.
- **Cedar** is used for restricted AgentCore tool access. **PostgreSQL RLS** enforces appropriate row restrictions behind the enterprise APIs. Shared RAG does not require a special per-user Cedar restriction.

### Two Gateway authentication boundaries

We also agreed on two separate AgentCore entry paths:

- **G1 — human Gateway:** inbound Microsoft Entra JWT for Sarah, James, Emma, David and Oliver.
- **G2 — machine Gateway:** inbound AWS IAM for the Network Health Reporter runtime.

Each Gateway has one inbound authentication method, while **each downstream target can use its own outbound method** from the table above.

Finally, the two REST integration approaches are intentional: the **Network/Ticket APIs use OpenAPI-defined targets** for OAuth, while **Platform Health uses a native AWS API Gateway REST-stage target** with IAM. Both ultimately call ordinary REST APIs.

These are our **core integration/authentication patterns**. Detailed session handling, Cedar policies, database controls, logging and implementation decisions belong in the supporting architecture documents—not in the introductory access tables.

## More detail (optional)

- [Architecture baseline](docs/architecture-baseline.md) — complete technical design.
- [Patterns guide](docs/architecture-patterns.md) — detailed authentication flows and examples.
- [Architecture decisions](docs/architecture-decisions.md) — all decisions in one ADR document.
- [Test plan](docs/test-and-evidence-plan.md) — expected success and denial scenarios.
