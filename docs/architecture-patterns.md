# Architecture patterns catalogue

> Design baseline, not implementation evidence. The README presents **five downstream access patterns** and **three invocation paths**. This catalogue separately indexes **14 reusable technical building blocks (P01–P14)**, which can be combined in those paths. They are not 14 additional downstream systems or authentication protocols.

## Pattern map

| ID | Pattern | Identity at boundary | Demonstrated by |
|---|---|---|---|
| P01 | Corporate SSO and JWT | Entra human JWT | All five human users |
| P02 | Identity-preserving Lambda ingress | Original Entra JWT forwarded | Chat requests |
| P03 | Entra on-behalf-of (OBO) | User delegated token for downstream audience | James, Sarah, Emma (permitted operations) |
| P04 | OAuth Client Credentials for non-human callers | Separate application identities; no human | AWS Service Account and Entra Service Account |
| P05 | OAuth 3-legged consent (3LO) | Vendor user consent and vendor token | David |
| P06 | Native API Gateway REST stage MCP target | Gateway IAM/SigV4 identity downstream | Oliver |
| P07 | OpenAPI-defined REST MCP target | OAuth token downstream | Network API (including ticket operations) |
| P08 | JWT-inbound Gateway with Cedar | Human **or machine** JWT claims, distinct allow-lists | G1 |
| P09 | IAM-inbound Gateway with Cedar | Scheduled Lambda IAM execution role | G2 |
| P10 | Downstream API authorization and PostgreSQL RLS | Validated Entra human/app claims | Network API |
| P11 | Shared managed knowledge retrieval | Gateway/connector service identity | All human users |
| P12 | Secured AgentCore Runtime ingress for AI callers | JWT-protected Runtime for humans and Entra automation | AI requests only |
| P13 | Confirmed, idempotent writes | Delegated user identity | Ticket creation |
| P14 | Correlated audit and least privilege | Actor and execution identity both logged | All journeys |

## P01 — Corporate SSO and JWT

**Situation:** A human opens our Enterprise AI Application.

**Flow:** Browser signs in with Entra ID using authorization code + PKCE. The registered web client requests the AI backend's delegated `chat.invoke` scope. The browser submits a chat request and access token to AWS API Gateway. API Gateway validates issuer, signature, audience, expiry and configured authorization constraints. Authentication does not itself authorize all MCP tools.

**Principal:** Entra `tid` + `oid` is stable user identity. The token is for the AI backend, not Microsoft Graph or the downstream Network API.

## P02 — Lambda forwards the authenticated principal

**Situation:** Preserve James's user identity while keeping Lambda for real functions.

**Flow:** API Gateway → private Lambda → AgentCore AgentCore Runtime over PrivateLink. Lambda validates request shape and session association, produces correlation/audit records, and forwards the original permitted Entra JWT in an authorization header. AgentCore validates it independently. Lambda's IAM execution role grants invocation rights but **must not replace the user principal as the authorization basis**.

**Benefit:** Central ingress checks, audit, rate/size handling, session correlation and a VPC boundary. **Not** an excuse to trust `user=James` in JSON.

**Enforcement:** AgentCore Runtime is JWT-only and a resource policy restricts eligible ingress; IAM-only invocation cannot be an alternate route for human tools.

## P03 — Entra delegated OBO

**Situation:** James requests VIC diagnostics. The Network API must know **James**, not merely a shared service role.

1. Backend receives a valid access token whose audience is the registered middle-tier AI backend.
2. An OBO-capable trusted component exchanges that token at Entra's token endpoint for a new **Network API-audience** delegated access token, asking for `diagnostics.read`.
   The **AgentCore Entra OBO credential provider uses the same `enterprise-ai-backend` app registration/client ID** that Token A targets. If it uses another client ID, Entra rejects the assertion because the audience does not match.
3. AgentCore Gateway's outbound OAuth configuration obtains/uses the delegated downstream token as configured.
4. Network API validates its own audience, scopes and `tid/oid`, maps James to approved VIC rows and applies RLS.
5. Denied NSW diagnostic queries remain denied regardless of the model's instructions.

Ticket creation uses the **same Network API audience** but requires the delegated scope `tickets.write`. The exact Gateway outbound OBO token exchange integration and identity propagation must be matched to documented AWS connector capabilities before console configuration; no custom impersonation shortcut.

**Security invariant:** A shared Lambda/Gateway IAM role must not override the downstream delegated authorization decision.

## P04 — M2M 2LO

**Situation:** A scheduled AWS Service Account needs national aggregate health, but no employee is present.

`EventBridge Scheduler → Reporter Lambda (IAM role) → G2 (IAM/SigV4) → OpenAPI target → Entra OAuth 2.0 Client Credentials → Network Operations API → health summary`

Entra issues an **app-only** Network API access token. The service's `roles` claim, e.g. `HealthSummary.Read`, is distinct from a person's delegated `scp` claim. API checks app identity/assignment and offers only aggregate data. No OBO, no fabricated employee `oid`, no user claim.

**Important distinction:** G2's IAM caller is the **scheduled Lambda execution role**, because Lambda invokes the known MCP tool directly. There is no second Runtime and no AI prompt for this path.

**Second, independent non-human caller — Entra Service Account:** Unlike the AWS Service Account, this separate registered Entra application enters **JWT-protected AgentCore Runtime** with an **AI Backend-audience Client Credentials JWT** carrying `Automation.Runtime.Invoke` and an analysis prompt. The agent explicitly obtains a **separate JWT for the `enterprise-agentcore-g1` audience**, carrying that resource's Applications-only app role `Automation.Summary.Invoke` (admin-consented). G1 Cedar verifies the approved automation application ID and this G1-scoped role before permitting only the **health-summary tool** on a separate OpenAPI target. That target obtains a **new, Network API-audience Client Credentials token** for the *same* Entra Service Account app, which holds `HealthSummary.Read`. None of the three audience-specific JWTs is a human token; the Runtime JWT cannot be substituted at G1 or the Network Operations API. The two machines use the **same downstream summary REST endpoint**, but G1 and G2 demonstrate **different inbound authentication**. No additional end system or endpoint.

## P05 — Independent vendor 3LO

**Situation:** David asks “Show my vendor alerts.” The external fictitious **NetworkMonitor SaaS** uses its own Cognito-backed OAuth authorization server and its own vendor user permissions. It is not the corporate Entra authorization server.

First use: David is redirected to vendor authorization, signs in to vendor and consents to `alerts.read`; a confidential component completes authorization-code exchange and securely associates the vendor token with David's session (AgentCore Identity/outbound credentials as supported). Subsequent tool calls use **David's vendor delegated access token**. NetworkMonitor validates it and shows only alert resources assigned to that vendor account.

This is not Entra OBO and not 2LO. Token isolation and revocation are per David; Sarah cannot borrow his vendor credential. This simulates user-specific access to external enterprise SaaS, using **a separate OAuth provider**, and therefore demonstrates a genuinely different pattern.

## P06 vs P07 — Two ways to represent REST endpoints as MCP tools

Both underlying services expose **REST/HTTP APIs**. **OpenAPI is a description format**, not a different API protocol.

| | P06 Native API Gateway REST API stage target | P07 OpenAPI schema target |
|---|---|---|
| Gateway registration | AWS API Gateway REST stage resource | OpenAPI document with operations/schemas |
| Example | Oliver: `get_platform_health` | James: `get_diagnostics`; Sarah: `create_ticket` |
| Downstream auth | IAM/SigV4 Gateway execution role (or API key, when supported) | OAuth outbound, including delegated OBO |
| Downstream sees | Gateway execution/service identity | Caller-specific delegated token, if configured |
| Fine-grained decision | Cedar checks **Oliver** before tool invocation; backend IAM checks Gateway | Cedar checks **James**; downstream API enforces James + RLS |
| Why included | Demonstrates identity-changing service-to-service delegation | Demonstrates identity-preserving delegated authorization |
| Not a claim | IAM target does **not** give the backend Oliver's identity | Merely uploading a schema does **not** create OBO automatically |

The native stage target's supported outbound modes do **not** include OAuth. An OAuth-protected REST service behind API Gateway therefore needs an OpenAPI-defined Gateway target (or another documented compatible integration).

## P08 / P09 — Separate inbound Gateway authorizers

Each AgentCore Gateway selects **one** inbound `authorizerType` at creation, so do not assume mixed JWT and IAM on one Gateway.

- **G1 Entra JWT:** `CUSTOM_JWT`; accepts validated **human or Entra Service Account application JWTs**. Cedar distinguishes their identities: humans receive assigned tools; the app can access only its **dedicated summary tool**.
- **G2 IAM AWS Service Account:** `AWS_IAM`; accepts the approved scheduled Lambda execution role directly; Cedar restricts to `get_network_health_summary`. Exposes *no* duplicated human tools.

Outbound credential providers are configured **per target**, independently of the inbound Gateway choice. This is why G1 can have RAG IAM, OpenAPI OBO, vendor 3LO, and native stage IAM targets together.

**Cedar is not copied blindly:** define common principles/templates once, but apply distinct policy sets to distinct principal types and tool inventories.

**One JWT inbound type, two explicitly approved audiences at G1:** G1 serves human AI Backend-audience Token A and the approved Entra Service Account's **separately issued G1-audience** app-only token. A JWT inbound type does **not** require every caller to share one resource audience. The automation Runtime JWT is not passed unchanged to G1. Do not require human `chat.invoke` as a gateway-wide `allowedScopes` rule, because the app-only G1 token has an application `roles` claim rather than human `scp`. Constrain issuer and each permitted audience at Runtime/G1, and trusted Runtime ingress to approved paths. For **Entra v2** tokens, the client ID is **`azp`**, whereas AgentCore's `allowedClients` expects **`client_id`**, so do not configure an apparent Entra client allow-list using that unsupported claim mapping. **G1 Cedar** checks exact `azp` IDs and the required verified human role or automation application role; the Chat API checks human `chat.invoke` before Runtime ingress. Assign the automation `Automation.Runtime.Invoke` on the backend resource and `Automation.Summary.Invoke` on the G1 resource; deny unapproved or role-less app-only tokens. The downstream Network API independently checks `HealthSummary.Read` on its own audience token. See [AWS inbound JWT configuration](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html) and [Microsoft app-only permission enforcement](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow).

## P10 — API enforcement and RDS row-level security

Gateway's Cedar decision protects **tool invocation**; the downstream API must separately check the OAuth token. **Cedar receives Entra JWT `roles` arrays as serialized principal-tag strings**, so policies must match **complete quoted role names** (and fail closed on missing claims), not approximate substrings. The Entra v2 `azp` claim identifies the client application and must match the actual approved client ID. For each request:

1. Verify signature, issuer, audience, expiry; distinguish delegated `scp` from app `roles`. For the Network **REST API**, use its documented **native JWT authorizer** with the appropriate `scp` scope on each **human** method, but no delegated scope on the app-only health-summary method; the backend Lambda separately checks exact approved `azp` and `HealthSummary.Read` app role.
2. Map immutable Entra `tid/oid` or service app ID to an internal entitlement record.
3. Derive authorized regions and operations from server-side policy, not prompt arguments.
4. Use parameterized SQL and a transaction-scoped, server-set PostgreSQL principal context.
5. Apply RLS policies for region/owner rows; make DB API role non-owner, no `BYPASSRLS`; use `FORCE ROW LEVEL SECURITY` where appropriate.
6. Apply **field/column response shaping** too—Emma's basic asset status must not leak sensitive diagnostics even if a row is visible.

Do not mistake OAuth itself for SQL RLS: the API bridges validated OAuth identity to database session state. Ensure connection pool isolation and rollback/reset context to prevent identity leakage.

## P11 — Shared RAG

Every human may search the same non-sensitive reference corpus (S3-managed knowledge content indexed into Bedrock Knowledge Bases/S3 Vectors as chosen). G1's managed Knowledge Base target retrieves under its configured AWS service permissions. The tool returns cited passages, never database entitlements. No per-user document filtering is claimed or required in this PoC.

## P12 — AI Runtime ingress as a security boundary

The **AgentCore Runtime** is JWT-protected for two AI invocation paths: the Enterprise AI Application's human requests and the approved Entra Service Account's automated AI analysis. The scheduled AWS Service Account Lambda instead signs a **direct G2 MCP tool call using IAM/SigV4**. Runtime ingress has its own narrow resource policy and private path; G2 independently allows only the scheduled Lambda role. **G1 must also be restricted to AgentCore Runtime's approved Gateway PrivateLink endpoint** using a resource/organization policy with `aws:SourceVpce`; JWT validity and Cedar permissions by themselves do not guarantee the approved chat path was followed. Distinguish AWS **invocation permission** from the application **end-user authentication and authorization** context.

## P13 — Safe ticket write

`create_ticket` requires Sarah or James, appropriate asset/region entitlement, explicit user confirmation **proved by a trusted backend-issued assertion checked at the Network API**, payload validation and downstream scoped OBO. A user/model-supplied confirmation flag cannot substitute for that proof; a server-issued idempotency key only helps with retries. Emma is denied. A model's text alone cannot commit a write.

## P14 — End-to-end audit

For AI invocation, link ingress request ID → Runtime session → Gateway tool invocation → API request → RDS transaction. For direct AWS M2M, link the Lambda invocation and G2 tool call directly; do not invent a Runtime session. Never log bearer tokens or secrets. Log authenticated `tid/oid` or service app ID, Gateway execution principal, decision, target, outcome, latency, region and ticket ID. An IAM role's presence alone must never be presented as proof that James was authorized.

## Q&A

**Is Gateway inbound authorization the same as outbound authorization?** No. Inbound authenticates the agent/runtime to the MCP Gateway; target-level outbound credentials authenticate Gateway to a downstream system.

**Does OBO mean Lambda's role becomes James?** No. OBO exchanges a delegated user token for another audience. IAM execution remains a separate service permission.

**Can we use native API Gateway target for the Entra OBO Network API?** Not under the documented outbound modes; use an OpenAPI target with supported OAuth configuration.

**Why can't G1 serve automation as well?** One configured inbound authorizer and different principal/permission boundaries. G2 keeps the IAM automation surface small.

**Does Cedar secure RDS?** No. Cedar guards MCP tools; downstream APIs and RLS enforce authoritative resource permissions.

**Does RLS hide Emma's secret columns?** No. Pair row restrictions with response DTO and column-level rules.

**Is David's vendor token issued by Entra?** No; separate fictional NetworkMonitor provider using Cognito.

**Is OAuth itself RLS?** No; validated OAuth claims must be mapped to database policy context.

**Why a Lambda if AgentCore supports JWT?** Validated ingress, auditing, request/session binding and private routing; **not** to replace caller identity.

**Are app roles the same as scopes?** Delegated scopes usually appear in `scp`; app-only permissions in `roles`. Enterprise user application roles are also possible, and need explicit validation.

**Are Entra security groups automatically OAuth scopes?** No. Group membership, application roles and API scopes are distinct configurations.

**Does OpenAPI require a new server?** No. It documents an existing REST API.

**Can a service account use OBO?** Client credentials is the correct M2M pattern when no user is present.

**Is our design deployed?** No—this is the agreed reference design, with implementation and evidence to follow.

## Primary references

- [AgentCore Gateway authorizer API](https://docs.aws.amazon.com/bedrock-agentcore-control/latest/APIReference/API_CreateGateway.html)
- [AgentCore Gateway outbound auth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AgentCore OBO](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html)
- [Native API Gateway target](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [Microsoft Entra OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [PostgreSQL RLS](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
