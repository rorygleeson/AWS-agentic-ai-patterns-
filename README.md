# Enterprise AI Agent — AWS AgentCore PoC

Agentic AI introduces a difficult enterprise architecture challenge: **an agent may need to access several systems, but each system can authenticate callers differently and enforce different permissions**. Preserving a person's identity, allowing a service account to act independently, and obtaining third-party consent are not the same problem.

**The objective of this project is to identify, explain and standardise these common authentication and authorization approaches into reusable AgentCore architecture patterns.** We use one fictional enterprise application, four end systems and six callers to show when each pattern applies—and how to prevent unauthorized access.

**Why these particular examples?** We chose them because each demonstrates a different access requirement:

- **Shared knowledge:** Anyone who successfully signs in to the AI chat app can search the same approved documents. **Why?** The documents are shared with all signed-in users, so we do not need to filter them by person.
- **Network Operations API:** Sarah, James and Emma sign in, but have different permissions. **Why?** This API must know **which person** is requesting data. James, for example, can see VIC diagnostics but not NSW.
- **Network Health Reporter service account:** This scheduled application requests an aggregate network-health report. **Why?** It runs without a human signing in, so it must use **its own application identity**, with access to the summary only.
- **External NetworkMonitor service:** David signs in to our AI app, then separately authorizes access to his account at another company. **Why?** Our corporate sign-in does not grant access to the vendor's system. David must consent to sharing his permitted alerts.
- **Platform Health API:** Oliver signs in to the AI app and is the only human allowed to use this tool. **Why?** The API accepts a request authenticated by an **AWS IAM role**, not Oliver's user token. We must check Oliver's permission **before** the Gateway uses that role.

Together, they show why enterprise AI agents need different authentication and authorization patterns.

**The key idea:** Signing in to the chat app is only the start. **Every connected system must still enforce its own access rules.**

## 1. End systems

| # | End system | Description | High-level access rule |
|---|---|---|---|
| **1** | **Knowledge Base (RAG)** | Uses retrieval-augmented generation (**RAG**) to answer questions from shared troubleshooting documents in Amazon Bedrock Knowledge Bases. | **All human users who successfully sign in to the AI chat app** can search the same approved knowledge. |
| **2** | **Network Operations API** | Read/write enterprise REST API backed by **Amazon RDS PostgreSQL**. Returns asset status, diagnostics and aggregate health, and reads/creates support tickets. | **Only authorized callers.** Human permissions depend on region and operation; the service account has aggregate-only access. |
| **3** | **NetworkMonitor** | Simulated monitoring service owned by **another company**, with its own user accounts, permissions and OAuth server (modelled using Amazon Cognito). Returns the vendor user's alerts. | **Only users granted tool access who separately authorize their vendor account.** Each user sees only their permitted alerts. |
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
| **Network Health Reporter** | **Automated service account** | **Network API** | **Aggregate network-health summary only**, using its own application identity. No individual asset access. |

### Where do these permissions come from?

**Entra groups, application roles and OAuth scopes do different jobs.** An API does not *own* an Entra group; it can check the authenticated user's group membership and apply its own rules.

For this PoC we make their responsibilities explicit:

| Control | Our example | Where it is enforced |
|---|---|---|
| **Application role** — what kind of tool the person may use | James: `Network.Engineer`; Emma: `Network.ServiceDesk` | Assigned directly to each test user in Entra; the role appears in the token issued for the AI backend. **AgentCore/Cedar** uses validated role claims to restrict tool calls. |
| **Entra security group** — which data the person may see | James belongs to `Network-VIC-Engineers` | The **Network API** verifies James's membership using trusted Entra group IDs and maps it to a **VIC-only entitlement**. |
| **OAuth scope** — which API operation the client can request | `diagnostics.read` in the Network API access token | The **Network API** checks the OBO token's audience and delegated scope; this **does not** automatically grant NSW access. |
| **Database row-level security (RLS)** — which records may be returned | James can retrieve VIC rows, not NSW rows | **PostgreSQL**, using the trusted entitlement set by the Network API, not a region supplied by the user or agent. |

**Example — James asks for NSW diagnostics:** He is signed in, has the Engineer role and may have the `diagnostics.read` scope. But his Entra group maps to **VIC only**, so the Network API denies the NSW request and database RLS prevents NSW rows being returned.

**Our Entra Free PoC choice:** Assign application roles **directly to users**, and use security-group membership separately for regional entitlements. Assigning an Entra **group to an enterprise-app role** is another valid production approach, but requires Entra ID P1/P2. Group membership does **not** automatically become an app role or an OAuth scope. Configure and validate group claims for the Network API; if membership cannot be verified, **deny access**. The service account instead uses an **application role** (`HealthSummary.Read`) and has no human group membership.

See [Microsoft's app-role guidance](https://learn.microsoft.com/en-us/entra/identity-platform/howto-add-app-roles-in-apps) and [group-assignment licensing](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-user-or-group-access-portal).

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

The access rules above lead to different technical patterns. **A use case is what someone wants to do; an authentication pattern is how the system grants and checks access.** One system can use more than one pattern: the **Network Operations API** uses OBO for Sarah, James and Emma, but **OAuth Client Credentials** for the Network Health Reporter service account.

The examples below show why **the identity reaching the end system matters**.

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

The same service account cannot replace every user. Some systems need to know **which person** made the request, some only **which application**, and the external vendor needs **separate user consent**.

| Why this pattern is needed | Authentication approach | What the downstream system knows |
|---|---|---|
| **Shared RAG:** Everyone signed into the AI app sees the same approved knowledge, so there is no need for per-user document access. | **Application sign-in + AWS IAM** for knowledge retrieval. | The AWS service identity, not individual document permissions. |
| **Network API:** Sarah, James and Emma have different region and action permissions. The API **must know which person** called. | **OAuth On-Behalf-Of (OBO)** with Microsoft Entra ID. | The human caller. The API enforces permissions and sensitive fields; PostgreSQL RLS restricts database rows. |
| **Service account:** Network Health Reporter runs on a schedule with **no human to represent**. | **OAuth Client Credentials** with Microsoft Entra ID. | The application identity and its aggregate-only permission. |
| **Third-party service:** David's vendor account belongs to **another company**. Our Entra login cannot authorize access to that account. | **OAuth Authorization Code** with the vendor's OAuth provider (simulated using Cognito). | Which vendor user authorized our application and what alerts they can access. |
| **Platform Health:** The downstream API accepts **AWS IAM-signed requests**, not a human's OAuth token. Oliver must be checked **before** Gateway calls it. | **IAM/SigV4:** AgentCore Gateway uses its role for the final API call. | The Gateway IAM role, **not Oliver**. Tool authorization enforces Oliver's access before invocation. |

### Authentication terms in plain English

- **OAuth OBO (On-Behalf-Of):** The backend obtains a new API-specific token **representing the signed-in user**.
- **OAuth Client Credentials:** An **application** gets a token for itself, without a human.
- **OAuth Authorization Code:** A **person authorizes an application** to access an account, such as an external vendor account.
- **AWS IAM:** An **AWS role** authenticates a request to an AWS-protected service.

### Why can't James use his first token for the Network API?

James signs in **once** to our AI chat app. Entra ID gives the app an access token. James then asks: “Show me VIC diagnostics.”

The Network API is a **different API**. His first token is meant for the AI app, **not** the Network API. The Network API should reject it. Otherwise, a token issued for one system could be reused to enter another.

**OBO (on-behalf-of) solves this without another sign-in:**

1. The trusted AI backend asks Entra ID for a token **for the Network API**, on James's behalf.
2. Entra ID issues a **new token** that still identifies James, but is intended for the Network API.
3. The Network API accepts that token only after validating it, then checks James's permissions: **VIC allowed, NSW denied**.

| | First token | New OBO token |
|---|---|---|
| **Represents** | James | James |
| **Intended for** | AI application | Network Operations API |
| **Example permission** | `chat.invoke` | `diagnostics.read` |

**Why this matters:** Each API should accept a token intended **for that API**, with the appropriate permissions. Microsoft recommends OBO when a middle-tier API calls another API on behalf of a signed-in user. See [Microsoft's guidance on APIs calling other APIs](https://learn.microsoft.com/en-us/security/zero-trust/develop/api-calls-api).

**Remember:** OBO preserves *who James is*. It does **not** give him permission to see everything.

### Authentication is not the same as authorization

- **Authentication** establishes the identity of the user, application or AWS role.
- **Authorization** decides which tool, operation, region, record or field that identity may access.
- **OBO preserves user identity; it does not automatically grant all data.** The Network API still enforces each user's permissions.
- **Cedar** (rules controlling which tools an agent may call) is used for restricted AgentCore tool access. **PostgreSQL RLS** enforces appropriate row restrictions behind the enterprise APIs. Shared RAG does not require a special per-user Cedar restriction.

### Two Gateway authentication boundaries

We also agreed on two separate AgentCore entry paths:

- **G1 — human Gateway:** inbound Microsoft Entra JWT (signed user access token) for Sarah, James, Emma, David and Oliver.
- **G2 — machine Gateway:** inbound AWS IAM for the Network Health Reporter runtime.

**Why two Gateways?** AgentCore Gateway lets you configure **only one inbound authentication type per Gateway**. G1 needs JWT for humans; G2 needs AWS IAM for the service runtime. **A single Gateway cannot use both inbound methods at once**, so the two-Gateway split is required for this design.

This is **not** because the end systems use different authentication. **One Gateway can have multiple targets, each with its own outbound authentication** (such as OAuth OBO, OAuth Authorization Code or IAM).

### Two ways to register REST APIs as MCP (agent) tools

Both systems expose ordinary **REST APIs**. We deliberately register them with AgentCore Gateway in two different ways:

| | Native API Gateway REST-stage target | OpenAPI-defined target |
|---|---|---|
| **Our example** | Platform Health API | Network Operations API |
| **How AgentCore connects** | Import the existing AWS API Gateway REST stage | Supply an OpenAPI description of the REST endpoints |
| **Outbound authentication** | IAM/SigV4 (or API key) | OAuth OBO for humans; OAuth Client Credentials for the service |
| **Downstream identity** | Gateway's IAM role | Delegated human or application identity |

**Why does Platform Health get its own pattern?** We intentionally chose a downstream API protected by **AWS IAM instead of OAuth**. The Network API needs the caller's identity to decide which records they may see. Platform Health needs only a trusted AWS role to read synthetic health data—provided the agent first verifies that **Oliver is allowed to use the tool**. It demonstrates the important distinction between **authorizing a human to invoke a tool** and **authenticating the tool's service identity at the final API**.

**Why the difference?** It's a limitation of the **AgentCore target type**, not REST itself. The native AWS API Gateway stage target supports IAM and API-key outbound authentication **but not OAuth**. An **OpenAPI target supports OAuth** through AgentCore's separately configured outbound credential provider—even if the actual REST API is hosted on AWS API Gateway. **The OpenAPI file describes the API; it doesn't provide OAuth or run another service.**

These are our **core integration/authentication patterns**. Detailed session handling, Cedar policies, database controls, logging and implementation decisions belong in the supporting architecture documents—not in the introductory access tables.

## Architecture at a glance

This diagram shows how the chat app, AWS services and two AgentCore Gateways connect to the end systems. **G1 handles signed-in users; G2 handles the service account.** The earlier tables explain who may use each connection.

![AWS AgentCore architecture with AWS service icons](docs/diagrams/aws-agentcore-architecture.svg)

**Terminology note:** You may see *two-legged OAuth (2LO)* used informally for application-only access, and *three-legged OAuth (3LO)* for user-authorized access. This guide uses the actual OAuth flow names—**Client Credentials** and **Authorization Code**—because they're clearer. OBO is a separate token-exchange flow used when a backend calls another API for a signed-in user.

## More detail (optional)

- [Architecture baseline](docs/architecture-baseline.md) — complete technical design.
- [Patterns guide](docs/architecture-patterns.md) — detailed authentication flows and examples.
- [Architecture decisions](docs/architecture-decisions.md) — all decisions in one ADR document.
- [Test plan](docs/test-and-evidence-plan.md) — expected success and denial scenarios.
