# Enterprise GenAI Platform on AWS — Architecture Baseline v1.0

**Status:** Agreed design baseline • **Date:** 27 November 2026 • **Scope:** Fictional enterprise reference implementation • **Deployment:** Not yet implemented.

## 1. Objective and design principles

Build a reusable AWS GenAI agent platform which exposes shared knowledge and governed enterprise REST tools. Compare **user-delegated identity**, **service identity**, and **independent third-party consent** without conflating authentication, authorization or data entitlements.

Rules: (1) identity comes from validated tokens, never prompts; (2) least privilege at each hop; (3) a successful Gateway decision does not replace API enforcement; (4) RDS is never an agent-facing endpoint; (5) human and machine lanes are deliberately separate; (6) avoid unnecessary tool or Cedar duplication; (7) record incompatible alternatives and trust boundaries; (8) no infrastructure-as-code requirement for initial learning build.

## 2. End systems and contracts

| System | Classification | Input/API contract | Output and restriction |
|---|---|---|---|
| Shared knowledge | Shared RAG repository | Native managed KB connector, native `Retrieve` MCP tool (shown to readers as “search knowledge”) | Cited procedure excerpts. Same approved documents for all five human users; no document-level ACL example. |
| Network Operations | Protected enterprise REST API | `GET /assets/{id}/status`, `GET /assets/{id}/diagnostics`, `POST /tickets`, `GET /network/health-summary` | Asset status/diagnostics, ticket creation and aggregate reporting; permission, region, field and write controls. |
| NetworkMonitor | External simulated SaaS | `GET /alerts` | Alerts tied to separately authenticated vendor account; own Cognito authorization server. |
| Platform Health | AWS-native protected REST API | `GET /platform/health` | Synthetic platform service health; IAM/SigV4 method, no human-specific data. |
| RDS PostgreSQL | Private internal database | SQL only from Network Operations application tier | Assets, faults, tickets and entitlements with RLS; not exposed as an MCP tool. |

**Target-type rule:** API Gateway REST API *stage* targets support IAM/SigV4 and API keys, not OAuth outbound. Register OAuth-protected Network API operations (including tickets) as an **OpenAPI target**, even if hosted on AWS API Gateway. Both remain ordinary REST APIs. The OpenAPI document describes operations; it is not an extra runtime service.

## 3. Actors and business use cases

| User | Permitted business outcomes | Explicitly forbidden |
|---|---|---|
| Sarah (National Admin) | Shared RAG; nationwide asset status and diagnostics; approved ticket creation | Oliver-only platform health; David's vendor account |
| James (VIC Engineer) | Shared RAG; VIC asset status, VIC diagnostics and VIC ticket creation | NSW restricted records; Oliver-only tool |
| Emma (Service Desk) | Shared RAG; nationwide **basic** status | Diagnostic fields and tools, ticket writes, Oliver-only tool |
| David (External Analyst) | Shared RAG; vendor alerts after independent NetworkMonitor consent | Internal privileged assets, diagnostics or tickets |
| Oliver (Platform Operations Analyst) | Shared RAG; platform health | Network diagnostics/tickets, other users' vendor alerts |
| AWS Network Health Reporter (IAM workload) | Aggregate report via G2 using IAM/SigV4 inbound | Human impersonation, diagnostics and writes |
| Entra Automation Client (service principal) | Same aggregate report via G1 using OAuth JWT inbound | Human tools, diagnostics and writes |

**Use case ≠ pattern.** “James investigates a VIC fault” is one business use case that composes SSO, JWT, OBO, MCP/OpenAPI, Cedar and RLS.

## 4. Component topology and trust boundaries

### Human lane

`Browser chat → Microsoft Entra (authorization code + PKCE) → API Gateway HTTP API (JWT) → proxy Lambda in an approved VPC → AgentCore PrivateLink → Runtime U (JWT inbound) → G1 (CUSTOM_JWT inbound) → MCP tool target → end system`.

- API Gateway, Lambda, Runtime U and G1 validate/forward the **AI Backend API audience** Token A as one logical API protection domain. Lambda forwards the unchanged signed user bearer JWT to Runtime; Runtime validates it independently.
- Runtime U uses JWT, not IAM, for invocation. Lambda's role alone does not authenticate as a user.
- For calls to G1, the agent explicitly passes its inbound user JWT via the allowlisted Authorization header; G1 creates an `OAuthUser` Cedar principal from the verified JWT. **Do not silently replace this with a Gateway M2M token**, as that would change the policy principal.
- AgentCore Identity obtains target-audience OBO Token B for the Network API's asset, diagnostics and ticket creation. The downstream API validates the token and applies its own authorizations.
- David's vendor tool uses independent Cognito 3LO consent. Oliver's native stage target deliberately uses Gateway IAM identity outbound, without OBO.
- The Gateway's execution role is separate from the JWT principal presented to Cedar.

### Machine lane

`EventBridge Scheduler → Reporter Lambda (Lambda execution role) → Runtime M (IAM inbound) → G2 (AWS_IAM inbound signed by Runtime M execution role) → health-summary OpenAPI target (OAuth 2LO) → Network API`.

The IAM principal observed by **G2 is Runtime M's execution role**, not automatically the upstream Lambda role. G2 obtains an Entra *application* token through its 2LO credential provider; the Network API sees the Reporter application identity and authorizes aggregate-only access.

### Separate Entra application-only lane

The **Entra Automation Client** is a **different non-human caller** from AWS Reporter Lambda. It has its **own Entra app registration/service principal**, obtains an AI-backend-audience JWT through **Client Credentials**, invokes JWT-protected **Runtime U → G1**, and is allowed by G1 Cedar to invoke **only** its dedicated health-summary tool. The G1 summary OpenAPI target uses that client's separately configured **Client Credentials** to obtain a **Network API-audience** token. The Network API validates the app ID and `HealthSummary.Read` application role. It is not OBO and does not represent a human. Its AI-backend JWT cannot be reused against Network API due to the different audience.

**Two different incoming machine identities:** AWS Reporter IAM role → **G2**; Entra Automation Client OAuth JWT → **G1**. Both ultimately call **one existing** `GET /network/health-summary` endpoint under their respective Entra application identities. No new downstream system or REST endpoint is introduced.

### Gateway boundaries

**G1: Entra JWT.** Human tools plus a separately authorized application-only summary tool: shared KB; basic status; diagnostics; ticket creation; vendor alerts; native platform health. Cedar distinguishes human and application claims and grants each only its approved tools. A distinct target provider can be OBO, 3LO or IAM.

**G2: AWS IAM.** One `get_network_health_summary` tool, one small role-restricted policy, limited Gateway execution role and 2LO target provider. There is no copy of all human Cedar rules in G2.

The API/Gateway and Runtime must not share unrestricted public bypasses. Runtime U is reached over its approved private ingress and separately constrained to the trusted Chat API/Lambda and approved automation route. **G1 itself is private**: Runtime U calls the AgentCore Gateway interface VPC endpoint, and a Gateway resource policy or organization resource control policy restricts G1 by `aws:SourceVpce`. Valid Entra JWTs alone must not permit a direct public G1 bypass. G2 restricts inbound IAM callers to Runtime M's role. Backend APIs independently require their own audience-specific token or AWS IAM signature.

## 5. Entra, token and authorization contracts

### App registrations

1. `enterprise-ai-web`: public browser client, authorization-code + PKCE, granted `chat.invoke`.
2. `enterprise-ai-backend`: protected logical AI API, exposes human delegated scope `chat.invoke` **and a separate application role `Automation.Summary.Invoke` (Applications only)**; confidential client performing OBO, audience for Token A.
   - **Same application ID, two uses:** AgentCore's **Microsoft Entra OBO credential provider** must use this **same `enterprise-ai-backend` client ID** and its confidential client credentials. Token A is issued for this backend app's valid audience. Do **not** create a separate OBO client with a different ID; Microsoft rejects the exchange when the assertion's audience belongs to a different application.
3. `enterprise-network-api`: one Network API resource, delegated scopes `status.read`, `diagnostics.read`, `tickets.write`, and application role `HealthSummary.Read`.
4. `network-health-reporter`: confidential service application granted **application** `HealthSummary.Read` only.
5. `entra-automation-client`: **separate** confidential service principal. Assign and admin-consent the AI Backend **application** role `Automation.Summary.Invoke` and the separate Network API **application** role `HealthSummary.Read`. It requests a Client Credentials access token for **each distinct audience** using each resource's `/.default`; never reuses the AI Backend JWT against the Network API. The G1 summary target has the application's separate outbound credential provider.
6. NetworkMonitor Cognito: separate vendor authorization server/resource, client and `alerts.read` user consent.

Token A (human): Entra issuer, **AI Backend API** audience, `chat.invoke` in `scp`, user `tid` + `oid` + `sub`, and directly assigned business-role claim(s). The backend checks the delegated scope.

Token A-M (Entra Automation Client): same **AI Backend API audience**, but an **app-only** Client Credentials token with `roles: [Automation.Summary.Invoke]`, approved client application identity, and **no human `scp`**. Neither the browser nor the automation client borrows the other's role. Runtime U and G1 validate issuer and audience; G1 Cedar checks the exact **Entra v2 `azp` client application ID** plus the appropriate delegated human permission or machine app role. AgentCore JWT `allowedClients` checks `client_id` and must **not** be used as if it checked Entra `azp`. Runtime U ingress is independently private/restricted. Avoid G1-wide `allowedScopes: [chat.invoke]` as it would exclude app-only tokens. A valid AI Backend token without its appropriate delegated scope or assigned application role is not enough.

Token B: issued via Entra OBO, **Network API** audience, the original user's delegated identity, narrow Network scopes. Ticket calls use the same Network API audience with the `tickets.write` delegated scope. The OBO assertion must have the audience of the confidential backend client performing the exchange.

Token C: Entra client credentials, **Network API** audience, Reporter application's `roles` (not human `scp`), no human `oid` identity claim.

Token D: Cognito-issued vendor access token, **NetworkMonitor API** audience, vendor-user `alerts.read`; cannot be exchanged for internal API permissions.

**Business roles:** directly assign `Network.Admin`, `Network.Engineer`, `Network.ServiceDesk`, `Vendor.Analyst`, `Platform.Analyst` to human users in the AI backend application. Maintain Entra groups for group/entitlement learning, mapping validated group object IDs in the API. Direct role assignment avoids requiring premium group-to-enterprise-app assignment. Missing/overage group signals fail closed; do not infer membership from user-supplied names.

## 6. Data contract and RLS enforcement

The API validates a downstream audience-specific JWT; it resolves `(tid, oid)` for a human or validated service app identity for Reporter. It obtains effective permissions from protected `principals` and `principal_entitlements` tables, then opens a transaction and sets request-local DB context.

Tables: `assets(asset_id, region, basic_status, ...)`; `diagnostics(..., asset_id, region, restricted_payload)`; `tickets(..., region, ...)`; `principals(...)`; `principal_entitlements(principal_id, permission, region)`.

RLS checks trusted principal entitlements, not a client-requested `region`. Enable/force RLS and ensure runtime DB roles are not owners or `BYPASSRLS`; transaction-local settings prevent cross-request bleed in pooled connections. Parameterized queries and restricted database privileges remain mandatory; SQL injection would undermine application-supplied session context. Emma's restricted **fields** additionally require a basic-status DTO—RLS is not a field-redaction feature.

Ticket `POST` requires confirmed intent, resource-level authorization, `WITH CHECK`-equivalent write controls and an auditable response. **The Network API checks a confirmation assertion issued by the trusted application/backend and bound to the authenticated user and proposed ticket**, not a user-supplied `confirmed: true` or an agent's text. An idempotency key is useful for safe retries but does **not** establish authorization or confirmation.

## 7. AgentCore MCP/tool inventory

| Gateway | MCP tool | Target and outbound auth | Core authorization |
|---|---|---|---|
| G1 | `Retrieve` (target-prefixed) | Managed KB connector, Gateway IAM retrieval | Available to all human roles |
| G1 | `get_asset_status` | Network OpenAPI target, Entra OBO | Emma basic DTO, Sarah national, James VIC |
| G1 | `get_diagnostics` | Network OpenAPI target, Entra OBO | Sarah national, James VIC; Emma denied |
| G1 | `create_ticket` | Network OpenAPI target, Entra OBO | Sarah/James, confirmation and idempotency |
| G1 | `search_vendor_alerts` | NetworkMonitor OpenAPI/MCP target, Cognito authorization-code 3LO | David with vendor consent only |
| G1 | `get_platform_health` | **Native API Gateway REST stage** target, Gateway role SigV4 | **Oliver only**, simulated read-only status |
| G1 | `get_network_health_summary` | Separate Network OpenAPI target; **Entra Automation Client** Client Credentials | Only this verified Entra automation app, summary-only |
| G2 | `get_network_health_summary` | Network OpenAPI target, Entra application token 2LO | Runtime M role + aggregate-only service permission |

**There are three separate OpenAPI Gateway target configurations for one backend API:** G1 exposes three human operations via **OBO** plus a separate Automation Client **Client Credentials** summary target; G2 exposes the same summary operation with the **AWS Reporter's own Client Credentials**. Each target has isolated outbound credentials. Each operation is a distinct MCP tool with its own authorization decision. The same Network Operations API and RDS database serve both Gateway targets. The Platform Health REST API remains a separate **native API Gateway stage** target using IAM.

## 8. Session, audit and network controls

- Bind chat session identifiers to verified token `iss/sub` (or stable scoped IDs), rather than trusting arbitrary client-supplied Runtime session IDs.
- Authenticated MCP sessions are scoped to Gateway principals; policy sessions and agent chat sessions are separate concepts.
- Log correlation IDs, authenticated principal, requested tool, outcome, downstream application and policy decision; redact bearer tokens, secrets and sensitive prompts.
- Lambda uses request-size/schema validation and response mediation; it does **not** mint user identity.
- Runtime U is JWT-only, Runtime M is IAM-only. Deny user-ID impersonation alternatives where unnecessary.
- Use AgentCore PrivateLink for Lambda-to-Runtime invocation and resource policy restrictions; an OAuth JWT alone must not bypass mandatory Lambda checks.
- Prevent the model from invoking direct privileged APIs outside the governed tool path; downstream API auth remains mandatory for defense-in-depth.
- Synthetic health reports, test tenants and fictional data only.

## 9. Deliberate exclusions

No document-specific RAG ACL exercise, real customer data or production-readiness claims. The vendor is a **simulated** service using off-the-shelf Cognito OAuth capabilities, not a homegrown OAuth server. Implementation details are maintained separately.

## 10. Reference architecture requirements

- Distinguish business use cases, reusable patterns and implementation components.
- Show all seven callers, both Runtime modes, both Gateway modes, token transitions and four end systems.
- Prefer short, accessible user-to-system diagrams before the detailed AWS service view; AWS-icon diagram may be expanded as a separate deliverable.
- Keep architecture, patterns, ADR, implementation and testing documents independent.
- Design intent is supported by AWS/Microsoft/PostgreSQL documentation; test procedures verify the future deployment **conforms** to the design.

## References

- [AgentCore CreateGateway inbound types](https://docs.aws.amazon.com/bedrock-agentcore-control/latest/APIReference/API_CreateGateway.html)
- [Gateway target outbound authorization matrix](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [Gateway VPC egress / native target OAuth limitation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html)
- [Microsoft OBO and audience contract](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [AgentCore OBO, including Microsoft](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html)
- [AWS sample secure API Gateway / Lambda / JWT Runtime](https://github.com/aws-samples/sample-expose-agentcore-via-api-gateway)
- [AgentCore resource-based policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
- [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Microsoft group/app-role claims](https://learn.microsoft.com/en-us/security/zero-trust/develop/configure-tokens-group-claims-app-roles)
