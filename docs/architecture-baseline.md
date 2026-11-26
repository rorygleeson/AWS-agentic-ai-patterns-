# Enterprise GenAI Platform on AWS — Architecture Baseline v1.0

**Status:** Agreed design baseline • **Date:** 27 November 2026 • **Scope:** Fictional enterprise reference implementation • **Deployment:** Not yet implemented.

## 1. Objective and design principles

Build a reusable AWS GenAI agent platform which exposes shared knowledge and governed enterprise REST tools. Compare **user-delegated identity**, **service identity**, and **independent third-party consent** without conflating authentication, authorization or data entitlements.

Rules: (1) identity comes from validated tokens, never prompts; (2) least privilege at each hop; (3) a successful Gateway decision does not replace API enforcement; (4) RDS is never an agent-facing endpoint; (5) human and machine lanes are deliberately separate; (6) avoid unnecessary tool or Cedar duplication; (7) record incompatible alternatives and trust boundaries; (8) no infrastructure-as-code requirement for initial learning build.

## 2. End systems and contracts

| System | Classification | Input/API contract | Output and restriction |
|---|---|---|---|
| Shared knowledge | Shared RAG repository | Native managed KB connector, `search_knowledge(query)` | Cited procedure excerpts. Same approved documents for all five human users; no document-level ACL example. |
| Network Operations | Protected enterprise REST API | `GET /assets/{id}/status`, `GET /assets/{id}/diagnostics`, `GET /network/health-summary` | Status DTO, restricted diagnostics, aggregated totals; regional and field authorization. |
| Ticket Management | Separate protected enterprise REST API | `GET /tickets/{id}`, `POST /tickets` | Ticket status or new ticket; confirm, authorize and deduplicate state changes. |
| NetworkMonitor | External simulated SaaS | `GET /alerts` | Alerts tied to separately authenticated vendor account; own Cognito authorization server. |
| Platform Health | AWS-native protected REST API | `GET /platform/health` | Synthetic platform service health; IAM/SigV4 method, no human-specific data. |
| RDS PostgreSQL | Private internal database | SQL only from Network/Ticket application tier | Assets, faults, tickets and entitlements with RLS; not exposed as an MCP tool. |

**Target-type rule:** API Gateway REST API *stage* targets support IAM/SigV4 and API keys, not OAuth outbound. Register OAuth-protected Network and Ticket API endpoints as **OpenAPI targets**, even if hosted on AWS API Gateway. Both remain ordinary REST APIs. The OpenAPI document describes operations; it is not an extra runtime service.

## 3. Actors and business use cases

| User | Permitted business outcomes | Explicitly forbidden |
|---|---|---|
| Sarah (National Admin) | Shared RAG; nationwide asset status and diagnostics; approved ticket read/create | Oliver-only platform health; David's vendor account |
| James (VIC Engineer) | Shared RAG; VIC asset status, VIC diagnostics and VIC ticket activity | NSW restricted records; Oliver-only tool |
| Emma (Service Desk) | Shared RAG; nationwide **basic** status | Diagnostic fields and tools, ticket writes, Oliver-only tool |
| David (External Analyst) | Shared RAG; vendor alerts after independent NetworkMonitor consent | Internal privileged assets, diagnostics or tickets |
| Oliver (Platform Operations Analyst) | Shared RAG; platform health | Network diagnostics/tickets, other users' vendor alerts |
| Network Health Reporter (non-human) | Scheduled aggregate read-only network health report | Human impersonation, diagnostics, writes, vendor consent |

**Use case ≠ pattern.** “James investigates a VIC fault” is one business use case that composes SSO, JWT, OBO, MCP/OpenAPI, Cedar and RLS.

## 4. Component topology and trust boundaries

### Human lane

`Browser chat → Microsoft Entra (authorization code + PKCE) → API Gateway HTTP API (JWT) → proxy Lambda in an approved VPC → AgentCore PrivateLink → Runtime U (JWT inbound) → G1 (CUSTOM_JWT inbound) → MCP tool target → end system`.

- API Gateway, Lambda, Runtime U and G1 validate/forward the **AI Backend API audience** Token A as one logical API protection domain. Lambda forwards the unchanged signed user bearer JWT to Runtime; Runtime validates it independently.
- Runtime U uses JWT, not IAM, for invocation. Lambda's role alone does not authenticate as a user.
- For calls to G1, the agent explicitly passes its inbound user JWT via the allowlisted Authorization header; G1 creates an `OAuthUser` Cedar principal from the verified JWT. **Do not silently replace this with a Gateway M2M token**, as that would change the policy principal.
- AgentCore Identity obtains target-audience OBO Token B for Network or a separate Ticket API token. The downstream API validates the token and applies its own authorizations.
- David's vendor tool uses independent Cognito 3LO consent. Oliver's native stage target deliberately uses Gateway IAM identity outbound, without OBO.
- The Gateway's execution role is separate from the JWT principal presented to Cedar.

### Machine lane

`EventBridge Scheduler → Reporter Lambda (Lambda execution role) → Runtime M (IAM inbound) → G2 (AWS_IAM inbound signed by Runtime M execution role) → health-summary OpenAPI target (OAuth 2LO) → Network API`.

The IAM principal observed by **G2 is Runtime M's execution role**, not automatically the upstream Lambda role. G2 obtains an Entra *application* token through its 2LO credential provider; the Network API sees the Reporter application identity and authorizes aggregate-only access.

### Gateway boundaries

**G1: Entra JWT.** Human tools: shared KB; basic status; diagnostics; ticket read/create; vendor alerts; native platform health. Cedar uses user principal + tool/input. A distinct target provider can be OBO, 3LO or IAM.

**G2: AWS IAM.** One `get_network_health_summary` tool, one small role-restricted policy, limited Gateway execution role and 2LO target provider. There is no copy of all human Cedar rules in G2.

The API/Gateway and Runtime must not share unrestricted public bypasses: the human Runtime resource policy restricts OAuth invocation to the approved VPC; the VPC endpoint security group allows the designated application path. Backend APIs also require their own valid target-audience token or AWS IAM signature.

## 5. Entra, token and authorization contracts

### App registrations

1. `enterprise-ai-web`: public browser client, authorization-code + PKCE, granted `chat.invoke`.
2. `enterprise-ai-backend`: protected logical AI API, exposes `chat.invoke`, confidential client performing OBO, audience for Token A.
3. `enterprise-network-api`: Network API resource, delegated scopes `asset.read` / `diagnostics.read` and application role `HealthSummary.Read`.
4. `enterprise-ticket-api`: separate Ticket API resource, delegated `ticket.read` / `ticket.create`.
5. `network-health-reporter`: confidential service application granted **application** `HealthSummary.Read` only.
6. NetworkMonitor Cognito: separate vendor authorization server/resource, client and `alerts.read` user consent.

Token A: Entra issuer, **AI Backend API** audience, `chat.invoke` in `scp`, user `tid` + `oid` + `sub`, and directly assigned business-role claim(s). JWT authorizers require valid signature/expiry/issuer/audience and appropriate scope.

Token B: issued via Entra OBO, **Network API** audience, the original user's delegated identity, narrow Network scopes. Ticket calls obtain a different Ticket API-audience token. The OBO assertion must have the audience of the confidential backend client performing the exchange.

Token C: Entra client credentials, **Network API** audience, Reporter application's `roles` (not human `scp`), no human `oid` identity claim.

Token D: Cognito-issued vendor access token, **NetworkMonitor API** audience, vendor-user `alerts.read`; cannot be exchanged for internal API permissions.

**Business roles:** directly assign `Network.Admin`, `Network.Engineer`, `Network.ServiceDesk`, `Network.ExternalAnalyst`, `Platform.Observer` to human users in the AI backend application. Maintain Entra groups for group/entitlement learning, mapping validated group object IDs in the API. Direct role assignment avoids requiring premium group-to-enterprise-app assignment. Missing/overage group signals fail closed; do not infer membership from user-supplied names.

## 6. Data contract and RLS enforcement

The API validates a downstream audience-specific JWT; it resolves `(tid, oid)` for a human or validated service app identity for Reporter. It obtains effective permissions from protected `principals` and `principal_entitlements` tables, then opens a transaction and sets request-local DB context.

Tables: `assets(asset_id, region, basic_status, ...)`; `diagnostics(..., asset_id, region, restricted_payload)`; `tickets(..., region, ...)`; `principals(...)`; `principal_entitlements(principal_id, permission, region)`.

RLS checks trusted principal entitlements, not a client-requested `region`. Enable/force RLS and ensure runtime DB roles are not owners or `BYPASSRLS`; transaction-local settings prevent cross-request bleed in pooled connections. Parameterized queries and restricted database privileges remain mandatory; SQL injection would undermine application-supplied session context. Emma's restricted **fields** additionally require a basic-status DTO—RLS is not a field-redaction feature.

Ticket `POST` requires confirmed intent, resource-level authorization, `WITH CHECK`-equivalent write controls, an idempotency key and auditable response.

## 7. AgentCore MCP/tool inventory

| Gateway | MCP tool | Target and outbound auth | Core authorization |
|---|---|---|---|
| G1 | `search_knowledge` | Managed KB connector, Gateway IAM retrieval | Available to all human roles |
| G1 | `get_asset_status` | Network OpenAPI target, Entra OBO | Emma basic DTO, Sarah national, James VIC |
| G1 | `get_diagnostics` | Network OpenAPI target, Entra OBO | Sarah national, James VIC; Emma denied |
| G1 | `get_ticket` | Ticket OpenAPI target, Entra OBO | Sarah and James authorized region |
| G1 | `create_ticket` | Ticket OpenAPI target, Entra OBO | Sarah/James, confirmation and idempotency |
| G1 | `search_vendor_alerts` | NetworkMonitor OpenAPI/MCP target, Cognito authorization-code 3LO | David with vendor consent only |
| G1 | `get_platform_health` | **Native API Gateway REST stage** target, Gateway role SigV4 | **Oliver only**, simulated read-only status |
| G2 | `get_network_health_summary` | Network OpenAPI target, Entra application token 2LO | Runtime M role + aggregate-only service permission |

An API with two OpenAPI operations yields two MCP tools. Native API Gateway targets may import/export a stage's operations, but do **not** require authoring a new separate OpenAPI integration in G1. The same underlying Network backend may be shared without duplicating API code, data or every policy.

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

No document-specific RAG ACL exercise; no citizen-development programme; no production subscriber data; no actual Splunk or Glean deployment; no IaC requirement; no claim that synthetic platform health reflects live AWS status. The vendor is a **simulated** service backed by real off-the-shelf Cognito OAuth capabilities, not a homegrown OAuth server.

## 10. Reference architecture requirements

- Distinguish business use cases, reusable patterns and implementation components.
- Show all six callers, both Runtime modes, both Gateway modes, token transitions and five end systems.
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
