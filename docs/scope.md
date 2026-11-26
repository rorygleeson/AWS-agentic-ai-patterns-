# PoC Scope and Acceptance Criteria

**Status:** Stage 1 reconciliation in progress, based on the agreed enterprise scenario and later addition of Oliver. This is a synthetic demonstration, not a real provider environment.

## Purpose

Demonstrate end-to-end agent architecture across **human interactive** and **scheduled machine** lanes, including token validation and propagation, MCP discovery/invocation, target-specific outbound authentication, Cedar, downstream API authorisation, PostgreSQL RLS, safe ticket mutations, retrieval and observability.

## Agreed actors and journeys

| Actor | Business journey | Expected authorisation |
|---|---|---|
| **Emma — Service Desk** | Check basic status and ask standard outage procedures | Nationwide **basic** asset status + shared knowledge; no diagnostics/ticket actions |
| **James — VIC Engineer** | Investigate a VIC fault and create a follow-up ticket if authorised | VIC assets/diagnostics and VIC ticket operations after explicit confirmation; NSW denied |
| **Sarah — National Admin** | Review national faults and create authorised tickets | National diagnostics/tickets; no automatic access to David's vendor account |
| **David — External Analyst** | Link a separate NetworkMonitor account and inspect alerts | Vendor alerts only after independent Cognito 3LO consent; no privileged internal network access |
| **Oliver — Platform Operations Analyst** | View platform-health information | `get_platform_health` only; no broader Network/Ticket privileges |
| **Network Health Reporter — machine** | Produce scheduled daily aggregate summary | IAM workload into Runtime M/G2; read-only OAuth 2LO to summary API; no user impersonation or write tools |

## Two lanes, not two generic user permissions

1. **Human:** Web Chat → Entra OIDC/OAuth → API Gateway JWT validation → thin Lambda adapter for request validation/correlation → **Runtime U (JWT inbound)** → **G1 (CUSTOM_JWT inbound + Cedar)** → targets with independent outbound auth.
2. **Machine:** EventBridge Scheduler → Lambda using IAM role → **Runtime M (IAM inbound)** → **G2 (AWS_IAM inbound)** → `get_network_health_summary` → restricted OAuth **2LO** Network API endpoint.
3. **Vendor subflow:** David is already signed into our web app through Entra; linking NetworkMonitor requires **separate Cognito 3LO** and a vendor token that Sarah cannot reuse.

The adapter must not terminate or fabricate user identity. Its token-propagation design, Runtime ingress protection and exact Entra token audiences must be validated before deployment.

## Agreed downstream systems

- Shared Knowledge repository — synthetic S3 documents → Bedrock Knowledge Bases / S3 Vectors.
- Network Operations REST API — synthetic asset/fault data stored in protected PostgreSQL.
- Ticket Management REST API — synthetic tickets in PostgreSQL, with explicit confirmation and idempotent creation.
- NetworkMonitor SaaS mock REST API — **separate Cognito identity provider** and personal vendor alerts.
- Platform Health REST API — native API Gateway stage target, IAM outbound.
- Network Health Summary endpoint — OpenAPI target, read-only OAuth client credentials.
- RDS PostgreSQL is **not an MCP target**: only Network/Ticket APIs access it, with validated claims mapped to transaction-scoped RLS context.

[Detailed tool mapping](tool-and-api-inventory.md) and [identity matrix](identities-and-permissions.md) are authoritative for tool membership/permissions. [ADR-001](architecture-decisions.md#adr-001--separate-human-jwt-and-automation-iam-gateways) explains the Gateway separation.

## Reusable patterns — existing agreed scope only

| ID | Pattern | Demonstration |
|---|---|---|
| P01 | Corporate Entra OIDC and JWT entry | Valid/invalid user access token at API Gateway and Runtime U |
| P02 | Entra OAuth OBO | User-delegated API token for Network/Ticket calls |
| P03 | IAM service + OAuth 2LO | Scheduled report through Runtime M/G2 to aggregate API |
| P04 | External OAuth 3LO | David independently connects Cognito vendor account |
| P05 | MCP and Cedar tool governance | Tool/parameter authorisation at G1; narrow IAM policy at G2 |
| P06 | API and PostgreSQL RLS | James VIC-only data, Sarah national data, no prompt-based privilege |
| P07 | Shared RAG retrieval | Approved corpus and citations, no unsupported per-document ACL claims |

Logging, tracing, guardrails, throttling, secrets and cost controls are cross-cutting, **not additional patterns**.

## Acceptance criteria — tests not yet executed

- [ ] Five humans and reporter exercise their designated flows; all forbidden cases deny.
- [ ] Absent, expired, invalid and wrong-audience JWTs fail at the appropriate ingress.
- [ ] Runtime U cannot be bypassed using only a Lambda IAM role; Runtime M admits only authorised IAM workloads.
- [ ] G1 JWT user identity and G2 IAM workload identity are separately authenticated and authorised.
- [ ] James receives VIC-only rows through API and RLS; NSW canaries remain inaccessible; Sarah gets authorised national rows.
- [ ] `create_ticket` requires explicit confirmation, enforced authorisation and idempotency; denial creates no ticket.
- [ ] David completes separate vendor consent; Sarah cannot inherit his vendor token; revocation blocks access.
- [ ] Reporter produces a read-only aggregate; its 2LO identity cannot call diagnostics or ticket mutations.
- [ ] Shared KB answers cite approved documents; no claim of document-level ACL enforcement.
- [ ] Native REST target, OpenAPI OAuth OBO, Cognito 3LO, OpenAPI OAuth 2LO, MCP JSON-RPC, Cedar and backend validation are evidenced by redacted traces.
- [ ] Every successful and denied path has a correlation ID; no bearer token or secret is logged.

## Test data and boundaries

Synthetic fixtures: **20–40 assets** across VIC/NSW, **30 faults**, **10 tickets**, **12 short knowledge articles**, **15 vendor alerts**, plus deterministic forbidden canaries. No real operational data, real provider integration or production personal identities.

**Environment baseline:** Existing user-owned AWS account, Sydney `ap-southeast-2`, free Entra test tenant, low-cost model to select at setup, AUD $30 equivalent *budget alert* (not hard cap), and post-test cleanup. No cloud resource creation authorised yet.

**Still to validate before Stage 2 is signed off:** Entra claims/groups, distinct API audiences/scopes, full Runtime JWT handoff, both ingress perimeters, RLS schema/context, exact target-specific OAuth support and vendor callback/refresh/revocation.

See [master TODO](../TODO.md) for internal project status.
