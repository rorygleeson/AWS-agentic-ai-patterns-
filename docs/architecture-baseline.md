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
| Platform Health | AWS-native protected REST API | `GET /platform/health` | Read-only operational health information; IAM/SigV4 method, no human-specific data. |
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
| AWS Service Account (using IAM Role) | Aggregate report by scheduled Lambda calling G2 directly using IAM/SigV4 inbound | Human impersonation, diagnostics and writes |
| Entra Service Account | Automated AI analysis via Runtime and G1 using Entra application JWTs; same summary endpoint | Human tools, diagnostics and writes |

**Use case ≠ pattern.** “James investigates a VIC fault” is one business use case that composes SSO, JWT, OBO, MCP/OpenAPI, Cedar and RLS.

## 4. Component topology and trust boundaries

### Human lane

`Enterprise AI Application (browser) → Microsoft Entra (authorization code + PKCE) → API Gateway HTTP API (JWT) → proxy Lambda in an approved VPC → AgentCore PrivateLink → AgentCore Runtime (JWT inbound) → G1 (CUSTOM_JWT inbound) → MCP tool target → end system`.

- API Gateway, Lambda, AgentCore Runtime and G1 validate/forward the **AI Backend API audience** Token A as one logical API protection domain. Lambda forwards the unchanged signed user bearer JWT to Runtime; Runtime validates it independently.
- AgentCore Runtime uses JWT, not IAM, for invocation. Lambda's role alone does not authenticate as a user.
- For calls to G1, the agent explicitly passes its inbound user JWT via the allowlisted Authorization header; G1 creates an `OAuthUser` Cedar principal from the verified JWT. **Do not silently replace this with a Gateway M2M token**, as that would change the policy principal.
- AgentCore Identity obtains target-audience OBO Token B for the Network API's asset, diagnostics and ticket creation. The downstream API validates the token and applies its own authorizations.
- David's vendor tool uses independent Cognito Authorization Code consent. Oliver's native stage target deliberately uses Gateway IAM identity outbound, without OBO.
- The Gateway's execution role is separate from the JWT principal presented to Cedar.

### Direct AWS service-account tool lane

`EventBridge Scheduler → Reporter Lambda (Lambda execution role) → G2 (AWS_IAM inbound, signed by the Lambda execution role) → health-summary OpenAPI target (OAuth 2.0 Client Credentials) → Network Operations API`.

The IAM principal observed by **G2 is the scheduled Lambda execution role**. No AgentCore Runtime or AI reasoning is involved: the workload invokes a known MCP tool directly. G2 obtains a separately configured Entra application-only token through its outbound credential provider; the Network Operations API sees the reporter's Entra application identity and authorizes health summary only.

### Entra service-account AI lane

The **Entra Service Account** is a **different non-human caller** from AWS Service Account Lambda. It has its **own Entra app registration/service principal**, obtains an **AI Backend/Runtime-audience JWT** through Client Credentials to invoke JWT-protected **AgentCore Runtime** with an AI analysis prompt. The running agent then obtains a **separate G1-audience application JWT**, and G1 Cedar permits **only** its dedicated health-summary tool. The G1 summary OpenAPI target uses that client's separately configured **Client Credentials** to obtain a **Network API-audience** token. The Network API validates the app ID and `HealthSummary.Read` application role. It is not OBO and does not represent a human. Its AI-backend JWT cannot be reused against Network API due to the different audience.

**Two different incoming machine identities:** AWS Service Account Lambda IAM role → **G2 directly**; Entra Service Account OAuth JWT → **AgentCore Runtime → G1**. Both ultimately call **one existing** `GET /network/health-summary` endpoint under their respective Entra application identities. No new downstream system or REST endpoint is introduced.

### Gateway boundaries

**G1: Entra JWT.** Human tools plus a separately authorized application-only summary tool: shared KB; basic status; diagnostics; ticket creation; vendor alerts; native platform health. Cedar distinguishes human and application claims and grants each only its approved tools. A distinct target provider can be OBO, Authorization Code or IAM.

**G2: AWS IAM.** One `get_network_health_summary` tool, one small role-restricted policy, limited Gateway execution role and Client Credentials target provider. There is no copy of all human Cedar rules in G2.

The API/Gateway and Runtime must not share unrestricted public bypasses. AgentCore Runtime is reached over its approved private ingress and separately constrained to the trusted Chat API/Lambda and approved automation route. **G1 itself is private**: AgentCore Runtime calls the AgentCore Gateway interface VPC endpoint, and a Gateway resource policy or organization resource control policy restricts G1 by `aws:SourceVpce`. Valid Entra JWTs alone must not permit a direct public G1 bypass. G2 restricts inbound IAM callers to the approved scheduled Lambda execution role. Backend APIs independently require their own audience-specific token or AWS IAM signature.

## 5. Entra, token and authorization contracts

### App registrations

1. `enterprise-ai-web`: public browser client, authorization-code + PKCE, granted `chat.invoke`.
2. `enterprise-ai-backend`: protected logical AI API, exposes human delegated scope `chat.invoke` **and an application role `Automation.Runtime.Invoke` (Applications only)** for the approved Entra automation's Runtime ingress; confidential client performing human OBO, audience for Token A.
   - **Same application ID, two uses:** AgentCore's **Microsoft Entra OBO credential provider** must use this **same `enterprise-ai-backend` client ID** and its confidential client credentials. Token A is issued for this backend app's valid audience. Do **not** create a separate OBO client with a different ID; Microsoft rejects the exchange when the assertion's audience belongs to a different application.
3. `enterprise-agentcore-g1`: separate Entra resource/API registration for the automation's **G1-audience JWT**, with Applications-only app role `Automation.Summary.Invoke`. G1 permits that audience for this approved automation while also permitting the carefully scoped human Token A audience.
4. `enterprise-network-api`: one Network Operations API resource, delegated scopes `status.read`, `diagnostics.read`, `tickets.write`, and application role `HealthSummary.Read`.
5. `network-health-reporter`: confidential service application granted **application** `HealthSummary.Read` only.
6. `entra-automation-client`: **separate** confidential service principal. Assign/admin-consent `Automation.Runtime.Invoke` on the AI Backend, `Automation.Summary.Invoke` on G1's resource registration, and `HealthSummary.Read` on the Network Operations API. It requests **three separately audience-scoped Client Credentials access tokens** for Runtime, G1 and the final API (the G1 target's outbound provider obtains the last token). Never reuse the inbound Runtime JWT at G1 or Network Operations API.
7. NetworkMonitor Cognito: separate vendor authorization server/resource, client and `alerts.read` user consent.

Token A (human): Entra issuer, **AI Backend API** audience, `chat.invoke` in `scp`, user `tid` + `oid` + `sub`, and directly assigned business-role claim(s). The backend checks the delegated scope.

**Entra automation tokens:** Runtime ingress receives an **AI Backend-audience** app-only JWT carrying `Automation.Runtime.Invoke`. The agent explicitly obtains a **second G1-audience JWT** carrying `Automation.Summary.Invoke`; G1 Cedar verifies the exact **Entra v2 `azp` client application ID** and app role before allowing the summary tool. The G1 target obtains a **third, Network Operations API-audience** token carrying `HealthSummary.Read` independently. None represents a human; no automatic forwarding or token reuse across these three audiences. G1 accepts both the approved human Token A audience and this separate automation audience. AgentCore JWT `allowedClients` checks `client_id` and must **not** be mistaken for Entra `azp`; avoid a gateway-wide human `chat.invoke` requirement.

Token B: issued via Entra OBO, **Network API** audience, the original user's delegated identity, narrow Network scopes. Ticket calls use the same Network API audience with the `tickets.write` delegated scope. The OBO assertion must have the audience of the confidential backend client performing the exchange.

Token C: Entra client credentials, **Network API** audience, Reporter application's `roles` (not human `scp`), no human `oid` identity claim.

Token D: Cognito-issued vendor access token, **NetworkMonitor API** audience, vendor-user `alerts.read`; cannot be exchanged for internal API permissions.

**Business roles:** directly assign `Network.Admin`, `Network.Engineer`, `Network.ServiceDesk`, `Vendor.Analyst`, `Platform.Analyst` to human users in the AI backend application. Maintain Entra groups for group/entitlement learning, mapping validated group object IDs in the API. Direct role assignment avoids requiring premium group-to-enterprise-app assignment. Missing/overage group signals fail closed; do not infer membership from user-supplied names.

## 6. Data contract and RLS enforcement

The Network REST API native JWT authorizer validates a downstream audience-specific JWT; it resolves `(tid, oid)` for a human or validated service app identity for Reporter. It verifies Entra security-group membership (from trustworthy token group object IDs or an authorized lookup), maps those IDs to permitted operations and regions using the protected RDS `group_entitlements` table, then opens a transaction and sets request-local DB context. Entra is authoritative for **membership**; RDS is authoritative for this PoC's **business permission mapping**.

Tables: `assets(asset_id, region, basic_status, ...)`; `diagnostics(..., asset_id, region, restricted_payload)`; `tickets(..., region, ...)`; `principals(...)`; `group_entitlements(group_object_id, permission, region)`.

RLS checks trusted principal entitlements, not a client-requested `region`. Enable/force RLS and ensure runtime DB roles are not owners or `BYPASSRLS`; transaction-local settings prevent cross-request bleed in pooled connections. Parameterized queries and restricted database privileges remain mandatory; SQL injection would undermine application-supplied session context. Emma's restricted **fields** additionally require a basic-status DTO—RLS is not a field-redaction feature.

Ticket `POST` requires confirmed intent, resource-level authorization, `WITH CHECK`-equivalent write controls and an auditable response. **The Network API checks a confirmation assertion issued by the trusted application/backend and bound to the authenticated user and proposed ticket**, not a user-supplied `confirmed: true` or an agent's text. An idempotency key is useful for safe retries but does **not** establish authorization or confirmation.

## 7. AgentCore MCP/tool inventory

| Gateway | MCP tool | Target and outbound auth | Core authorization |
|---|---|---|---|
| G1 | `Retrieve` (target-prefixed) | Managed KB connector, Gateway IAM retrieval | Available to all human roles |
| G1 | `get_asset_status` | Network OpenAPI target, Entra OBO | Emma basic DTO, Sarah national, James VIC |
| G1 | `get_diagnostics` | Network OpenAPI target, Entra OBO | Sarah national, James VIC; Emma denied |
| G1 | `create_ticket` | Network OpenAPI target, Entra OBO | Sarah/James, confirmation and idempotency |
| G1 | `search_vendor_alerts` | NetworkMonitor OpenAPI/MCP target, Cognito Authorization Code (with PKCE) | David with vendor consent only |
| G1 | `get_platform_health` | **Native API Gateway REST stage** target, Gateway role SigV4 | **Oliver only**, simulated read-only status |
| G1 | `get_network_health_summary` | Separate Network OpenAPI target; **Entra Service Account** Client Credentials | Only this verified Entra automation app, summary-only |
| G2 | `get_network_health_summary` | Network OpenAPI target, Entra application token via OAuth 2.0 Client Credentials | Scheduled Lambda execution role + summary-only service permission |

**There are three separate OpenAPI Gateway target configurations for one backend API:** G1 exposes three human operations via **OBO** plus a separate Entra Service Account **Client Credentials** summary target; G2 exposes the same summary operation with the **AWS Service Account's own Client Credentials**. Each target has isolated outbound credentials. Each operation is a distinct MCP tool with its own authorization decision. The same Network Operations API and RDS database serve both Gateway targets. The Platform Health REST API remains a separate **native API Gateway stage** target using IAM. The native target requires an **API Gateway public endpoint type** even though the methods require **AWS_IAM/SigV4** authorization; this is not anonymous public access, and private integrations may sit behind API Gateway.

## 8. Session, audit and network controls

- Bind chat session identifiers to verified token `iss/sub` (or stable scoped IDs), rather than trusting arbitrary client-supplied Runtime session IDs.
- Authenticated MCP sessions are scoped to Gateway principals; policy sessions and agent chat sessions are separate concepts.
- Log correlation IDs, authenticated principal, requested tool, outcome, downstream application and policy decision; redact bearer tokens, secrets and sensitive prompts.
- Lambda uses request-size/schema validation and response mediation; it does **not** mint user identity.
- The AI Runtime is JWT-protected for approved human and Entra automation requests. The AWS scheduled Lambda calls IAM-protected G2 directly, without a second Runtime. Deny user-ID impersonation alternatives.
- Use AgentCore PrivateLink for Lambda-to-Runtime invocation and resource policy restrictions; an OAuth JWT alone must not bypass mandatory Lambda checks.
- Prevent the model from invoking direct privileged APIs outside the governed tool path; downstream API auth remains mandatory for defense-in-depth.
- Fictional health-report records, test tenants and fictional data only.

## 9. Deliberate exclusions

No document-specific RAG ACL exercise, real customer data or production-readiness claims. The vendor is a **simulated** service using off-the-shelf Cognito OAuth capabilities, not a homegrown OAuth server. Implementation details are maintained separately.

## 10. Reference architecture requirements

- Distinguish business use cases, reusable patterns and implementation components.
- Show all seven callers, one AI Runtime path shared by human and Entra automation, the AWS Lambda direct-to-G2 tool route, both Gateway authorization types, token transitions and four end systems.
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
