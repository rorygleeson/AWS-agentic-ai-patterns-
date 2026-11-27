# Enterprise AI Agent — AWS AgentCore PoC

**Status:** Documented architecture design; the PoC has not yet been deployed or validated end-to-end.

Agentic AI introduces a difficult enterprise architecture challenge: **an agent may need to access several systems, but each system can authenticate callers differently and enforce different permissions**. Preserving a person's identity, allowing a service account to act independently, and obtaining third-party consent are not the same problem.

**The objective of this project is to identify, explain and standardise these common authentication and authorization approaches into reusable AgentCore architecture patterns.** We use one fictional enterprise application, four end systems and six callers to show when each pattern applies—and how to prevent unauthorized access.

**Why isn't one authentication pattern enough?** Because the end systems trust **different identities**: some need the signed-in person, some need an application identity, some require separate authorization from an external provider, and some accept an AWS IAM role. One token or service account cannot safely satisfy all of those requirements.

**Why these particular examples?** We chose them because each demonstrates a different access requirement:

- **Shared knowledge (RAG):** **What:** Our AI chat application retrieves approved documents from a retrieval-augmented generation (RAG) knowledge base to ground answers and reduce hallucinations. **Who:** Any human user who successfully signs in to **our AI chat application**. **Why:** Everyone uses the same approved knowledge, so no per-user document filtering is needed.
- **Network Operations API:** **What:** Returns network asset status, diagnostics and support tickets from our enterprise database. **Who:** Signed-in users with different permissions. **Why:** The API must preserve **which person** called, because some users may read only basic status, while others may see detailed diagnostics for their permitted region.
- **Automated network-health reporting:** **What:** Requests an aggregate network-health report from the Network Operations API. **Who:** A service account running automatically, without a human signing in. **Why:** It uses **its own application identity**, with summary-only access rather than a person's permissions.
- **External NetworkMonitor service:** **What:** Returns alerts from a different company's monitoring system. **Who:** An approved user who signs in to our AI chat application **and separately authorizes their vendor account**. **Why:** Our corporate sign-in cannot grant access to another company's user data.
- **Platform Health API:** **What:** Returns synthetic AI platform-health information. **Who:** A signed-in user explicitly authorized to invoke this tool. **Why:** The downstream API trusts the **Gateway's AWS IAM service role**, not that user's OAuth token, so tool access must be checked **before** the role makes the call.

Together, they show why enterprise AI agents need different authentication and authorization patterns.

**The key idea:** Signing in to the chat app is only the start. **Every connected system must still enforce its own access rules.**

## 1. End systems

| # | End system | Description | High-level access rule |
|---|---|---|---|
| **1** | **Knowledge Base (RAG)** | Uses retrieval-augmented generation (**RAG**) to answer questions from shared troubleshooting documents in Amazon Bedrock Knowledge Bases. | **All human users who successfully sign in to the AI chat app** can search the same approved knowledge. |
| **2** | **Network Operations API** | Read/write enterprise REST API backed by **Amazon RDS PostgreSQL**. Returns asset status, diagnostics and aggregate health, and reads/creates support tickets. | **Only authorized callers.** Human permissions depend on region and operation; the service account has aggregate-only access. |
| **3** | **NetworkMonitor** | Simulated monitoring service owned by **another company**, with its own user accounts, permissions and OAuth server (modelled using Amazon Cognito). Returns the vendor user's alerts. | **Only users granted tool access who separately authorize their vendor account.** Each user sees only their permitted alerts. |
| **4** | **Platform Health API** | AWS API Gateway REST API returning synthetic AI platform-health information. | **Only users explicitly granted access to this tool.** The downstream API authenticates the Gateway's IAM role. |

### Network Operations API — five endpoints

These are the complete planned REST endpoints for our PoC. They all belong to **one Network Operations API** backed by RDS PostgreSQL:

| Endpoint | What it returns or does | Access rule |
|---|---|---|
| `GET /assets/{id}/status` | Basic network asset status | Signed-in users **granted Network API access**; basic fields only for limited roles |
| `GET /assets/{id}/diagnostics` | Detailed fault and diagnostic information | Authorized engineers/admins; **permitted regions only** |
| `GET /tickets/{id}` | Reads an existing support ticket | Authorized engineers/admins; **permitted regions only** |
| `POST /tickets` | Creates a support ticket | Authorized engineers/admins; **permitted regions only**, with user confirmation |
| `GET /network/health-summary` | Aggregate network-health summary | **Automated reporting service account only**; no individual asset details |

The Network API enforces each caller's permissions. The agent never queries RDS directly. We introduce the named people and their exact permissions next.

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

**Endpoint access at a glance:** Sarah and James can use **status, diagnostics and ticket read/create** (with their respective regional limits). Emma can use **basic status only**. The Network Health Reporter can use **health-summary only**. David and Oliver have **no Network Operations API access**.

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

The same service account cannot replace every user. Some systems need to know **which person** made the request, some only **which application**, and the external vendor needs **a separate user-authorized OAuth grant**.

| Why this pattern is needed | Authentication approach | What the downstream system knows |
|---|---|---|
| **Shared RAG:** Everyone signed into the AI app sees the same approved knowledge, so there is no need for per-user document access. | **Application sign-in + AWS IAM** for knowledge retrieval. | The AWS service identity, not individual document permissions. |
| **Network API:** Sarah, James and Emma have different region and action permissions. The API **must know which person** called. | **OAuth On-Behalf-Of (OBO)** with Microsoft Entra ID. | The human caller. The API enforces permissions and sensitive fields; PostgreSQL RLS restricts database rows. |
| **Service account:** Network Health Reporter runs on a schedule with **no human to represent**. | **OAuth Client Credentials** with Microsoft Entra ID. | The application identity and its aggregate-only permission. |
| **Third-party service:** David's vendor account belongs to **another company**. Our Entra login cannot authorize access to that account. | **OAuth Authorization Code** with the vendor's OAuth provider (simulated using Cognito). David completes a separate vendor OAuth sign-in/authorization, and the resulting vendor credential is bound to his authenticated session. | Which vendor user authorized our application and what alerts they can access. |
| **Platform Health:** The downstream API accepts **AWS IAM-signed requests**, not a human's OAuth token. Oliver must be checked **before** Gateway calls it. | **IAM/SigV4 (AWS-signed service request):** AgentCore Gateway signs the final API call using its **service IAM role**, not Oliver's user credentials. | The Gateway IAM role, **not Oliver**. Tool authorization enforces Oliver's access before invocation. |

**Cognito simulation note:** Cognito supports OAuth Authorization Code and custom API scopes. Its managed login does not by itself demonstrate a separate, explicit consent screen like every real-world SaaS provider; this PoC demonstrates **independent vendor sign-in and delegated access**, not the visual consent experience.

### Authentication terms in plain English

- **OAuth OBO (On-Behalf-Of):** The backend obtains a new API-specific token **representing the signed-in user**.
- **OAuth Client Credentials:** An **application** gets a token for itself, without a human.
- **OAuth Authorization Code:** A **person authorizes an application** to access an account, such as an external vendor account.
- **AWS IAM:** An **AWS role** authenticates a request to an AWS-protected service.

### How Entra protects the Chat API and Network API

**First, what is the Chat API?** It is simply the AWS API Gateway endpoint our web chat calls when James sends a message: **`POST /ai/chat`**. That request goes through Lambda to AgentCore Runtime. It is **not another AI service**.

Entra ID protects both this **Chat API** and the separate **Network Operations API**. We register them as protected APIs in Entra, alongside the browser client application. Entra issues a different access token for each:

| | Token A — Chat API (`POST /ai/chat`) | Token B — Network Operations API |
|---|---|---|
| **Issued by** | Microsoft Entra ID | Microsoft Entra ID, using OBO |
| **Intended for (audience)** | Chat API | Network Operations API |
| **Example permission** | James's `Network.Engineer` app role and `chat.invoke` scope | `diagnostics.read` delegated scope |
| **Who enforces access?** | Chat API validates Token A; **Cedar (AgentCore tool-access policies)** restricts which tools James may use | **Network API** checks scope, identity and data entitlements; **PostgreSQL RLS** restricts rows |

**The Network API must not accept Token A:** it was issued for a different API. OBO obtains Token B **without another sign-in** and still represents James. The two APIs enforce different rules, even though both tokens come from Entra.

**Simple rule:** Entra defines and assigns roles and issues tokens. **Our systems enforce what those roles, scopes and groups mean.**

For James, this is what we configure:

1. **In Entra ID:** Define the `Network.Engineer` **app role** on our Chat API app registration, then assign James that role in its Enterprise Application. Entra includes it in the access token issued for the Chat API.
2. **In AgentCore/Cedar:** Define a rule that users with `Network.Engineer` may call the diagnostics tool. Entra does **not** create this rule for us.
3. **In the Network API:** After OBO provides a new token for this API, validate its `diagnostics.read` scope and James's identity. Resolve and verify his `Network-VIC-Engineers` membership using trusted group object IDs from Entra token claims **or an authorized Microsoft Graph lookup** (including any group-overage case). Our API maps that group to VIC-only access. PostgreSQL RLS then restricts the actual rows.

| Item | Plain-English purpose |
|---|---|
| **App role** (`Network.Engineer`) | **What kind of tools** James may use. Defined/assigned in Entra; enforced by our Cedar policy. |
| **Entra group** (`Network-VIC-Engineers`) | **Which region's data** James may access. Membership is managed in Entra; the mapping and restriction are enforced by the Network API and database. |
| **OAuth scope** (`diagnostics.read`) | **Which API operation** the client may request. Issued for and checked by the Network API. It is not permission to read every region. |

**Example:** James asks for NSW diagnostics. He has the Engineer role and a diagnostics scope, but his group grants **VIC only**. The Network API denies the request; database RLS also prevents NSW rows being returned.

**For our free Entra tenant:** We'll assign app roles **directly to test users**. We are **not** mapping groups automatically into app roles: group-to-enterprise-app role assignment requires Entra P1/P2. Each API must validate the claims intended for it; OBO does not automatically copy every claim from the original token. If group membership cannot be verified, deny access. The Network Health Reporter instead receives the application permission `HealthSummary.Read`, with no human user.

Further reading: [Microsoft on app roles](https://learn.microsoft.com/en-us/entra/identity-platform/howto-add-app-roles-in-apps) and [group-assignment licensing](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-user-or-group-access-portal).

### Why can't James use his first token for the Network API?

James signs in **once** to the web chat. The browser obtains **Token A for the Chat API (`POST /ai/chat`)**. James then asks: “Show me VIC diagnostics.”

The Network API is a **different API**. His first token is meant for the **Chat API**, **not** the Network API. The Network API should reject it. Otherwise, a token issued for one system could be reused to enter another.

**OBO (on-behalf-of) solves this without another sign-in:**

1. The Chat API validates **Token A** and forwards that same validated bearer token through Lambda to JWT-protected AgentCore Runtime and human Gateway G1. Runtime must explicitly propagate the bearer token to G1; **Lambda's IAM role is not a substitute for James's identity**.
2. The Network API's OpenAPI target uses **AgentCore Gateway/Identity's configured Entra OAuth credential provider** to exchange Token A for **Token B** using OBO. The credential provider's confidential Entra client must be authorized to redeem Token A (its `aud` must match that client/API registration), and delegated access to the Network API must be consented.
3. Entra issues Token B, which still identifies James but has the **Network API audience**. The Network API validates it and checks his permissions: **VIC allowed, NSW denied**.

The **Token A / Token B** comparison above shows the different audiences and permissions.

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

### Two ways to connect REST APIs to AgentCore tools

**First, what is an AgentCore Gateway target?** A **target** is the Gateway's configuration for connecting an AI agent to a backend system. It tells Gateway **where the system is, which operations to expose as tools, and how Gateway authenticates when calling it**. A target is **not a new API or another running service**.

**Model Context Protocol (MCP)** is the standard interface the agent uses to discover and call those tools.

Think of the terms this way:

- **End system:** Our Network Operations REST API.
- **Endpoint:** One operation on that API, such as `GET /assets/{id}/diagnostics`.
- **AgentCore Gateway target:** The configured connection to that API, including its operation definitions and outbound authentication.
- **MCP tool:** The action the agent sees and invokes, such as `get_diagnostics`. Gateway translates the tool call into the REST endpoint request.

**One Gateway can contain several targets, and one target can expose several tools.** Our Network API has five endpoints; each is exposed as its own agent tool.

**Both are ordinary REST APIs.** A REST API **can** authenticate users with OAuth. The difference is how we **register the API as a target in AgentCore Gateway** and which outbound credentials that particular target type supports.

| | Native AWS API Gateway stage target | OpenAPI schema target |
|---|---|---|
| **Our example** | Platform Health API | Network Operations API |
| **What we give AgentCore** | An existing AWS API Gateway REST API stage | An **OpenAPI document** listing the REST URLs, operations and parameters |
| **Outbound authentication AgentCore supports** | **AWS IAM/SigV4** or API key; **no OAuth option** for this target type | **OAuth**, including OBO for a person and Client Credentials for a service |
| **Identity the final API receives in our PoC** | Gateway's **AWS IAM service role** | A **user-delegated Entra token** or the service account's Entra token |

**Why does Network Operations need the OpenAPI option?** Its REST endpoints must identify the human caller to enforce regional permissions. When James requests VIC diagnostics:

1. AgentCore discovers the diagnostics tool from the **OpenAPI definition**.
2. AgentCore's configured **OAuth credential provider** uses OBO to obtain an Entra token **for the Network API**, still representing James.
3. Gateway calls the REST endpoint with that token. **The Network API validates it** and enforces James's VIC-only permissions.

The native API Gateway stage target **cannot perform step 2 using OAuth outbound authentication**. This is a **limitation of that AgentCore target type**, **not of REST APIs or AWS API Gateway itself**. The actual REST endpoint may still be hosted on AWS API Gateway.

**Why use the native target for Platform Health?** That API is deliberately protected by **AWS IAM**, so Gateway can import the REST stage and sign requests with **its own service role (SigV4)**. Oliver must first be authorized to use the tool; the final API authenticates the service role, **not Oliver's Entra token**.

**Important:** The OpenAPI document **does not authenticate anyone or create another service**. It describes the existing REST API. AgentCore's credential provider obtains the OAuth token, and the downstream API validates and authorizes it. See AWS's [Gateway outbound-authentication matrix](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html) and [native API Gateway target limitation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html).

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
