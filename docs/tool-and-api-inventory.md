# Downstream Systems and MCP Tool Inventory

**Status:** Restored agreed inventory. All systems/data are synthetic. Target and identity mechanisms must be validated in the running PoC.

## Tool catalogue

| Gateway | MCP tool | REST/backend operation | Downstream system | Outbound authentication |
|---|---|---|---|---|
| **G1 — JWT humans** | `search_knowledge` | Retrieve procedure excerpts/citations | Bedrock Knowledge Base: S3 corpus / S3 Vectors | AWS-controlled retrieval |
| G1 | `get_asset_status` | `GET /assets/{id}` | Network Operations REST API | Entra delegated **OBO** |
| G1 | `get_diagnostics` | `GET /assets/{id}/diagnostics` | Network Operations REST API | Entra delegated **OBO** |
| G1 | `get_ticket` | `GET /tickets/{id}` | Ticket Management REST API | Entra delegated **OBO** |
| G1 | `create_ticket` | `POST /tickets` | Ticket Management REST API | Entra OBO + Cedar + explicit confirmation + idempotency |
| G1 | `search_vendor_alerts` | `GET /alerts` | Mock NetworkMonitor SaaS REST API | **Cognito vendor 3LO**, separate from Entra |
| G1 | `get_platform_health` | Platform health GET (path to finalise) | Platform Health REST API via native API Gateway stage target | Gateway IAM/SigV4 |
| **G2 — IAM reporter** | `get_network_health_summary` | `GET /network/health-summary` | Network health API, read-only aggregate | OpenAPI target + restricted OAuth **2LO** |

A REST API can expose several operations/tools. **Target type and OAuth grant type are different design dimensions.** The same Network API implementation may serve G1 and G2 with separate audiences/operations, without duplicating its database.

## Trust boundaries and data stores

- **Network Operations API:** validated user-delegated scopes and region claims; protected RDS PostgreSQL assets/faults with RLS.
- **Ticket API:** validated scopes, region claims, confirmation and idempotent writes; protected RDS tickets with RLS.
- **Knowledge repository:** approved shared corpus, 12 synthetic short articles; source citations; no unverified per-document ACLs.
- **NetworkMonitor:** mock vendor data but real separate Cognito authorisation server; per-user consent, callback and token lifecycle.
- **Platform Health:** separately exposed native REST stage to demonstrate API Gateway target + IAM, for Oliver.
- **Network summary:** aggregate-only endpoint restricted to reporter; app token cannot read individual diagnostics or create tickets.
- **RDS PostgreSQL:** reached only by Network/Ticket APIs, **not** direct MCP tool or agent SQL access.

## Synthetic deterministic fixtures

- 20–40 assets across VIC/NSW, including known canaries `VIC-102` and `NSW-201`.
- 30 synthetic faults; 10 initial tickets; 12 short approved procedure articles; 15 vendor alerts.
- `get_asset_status` returns basic fields for Emma, richer authorised data for James/Sarah where policy permits.
- `get_diagnostics` enforces role/region and returns only authorised fault rows.
- `create_ticket` is the only agreed state-changing operation; cannot execute without confirmed intent, valid authority and idempotency key.
- `get_network_health_summary` returns aggregate counts/health only; no individual user identity, raw fault detail or mutations.
- Missing IDs, expired tokens, wrong regions and insufficient scopes produce deterministic, testable denials.
- No real network equipment, customers or live external integrations.

## REST API stage vs OpenAPI target — actual placement

- **Native API Gateway REST stage target:** G1's `get_platform_health`, using IAM/SigV4. It does **not** do OAuth OBO.
- **OpenAPI OAuth OBO targets:** G1's Network and Ticket APIs for delegated human operations, subject to proof of actual Entra token exchange.
- **Vendor Cognito 3LO target:** G1's NetworkMonitor integration, distinct from corporate login.
- **OpenAPI OAuth 2LO target:** G2's read-only health summary.
- Knowledge retrieval uses the agreed Bedrock KB integration, subject to actual target/tool configuration validation.

## Verification requirements

Confirm the exact supported target and credential combinations, OpenAPI `operationId` mappings, tool JSON schemas, `tools/list` and `tools/call`, outbound token audiences, IAM roles, vendor consent/refresh/revocation, RLS claims and per-tool Cedar policies. Record observed behaviour and any unsupported assumptions without silently substituting new patterns.

Relevant AWS docs: [API Gateway stage targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html) · [OpenAPI targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-schema-openapi.html) · [outbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html).

See [identities](identities-and-permissions.md), [two-Gateway technical design](two-gateway-design.md) and [master TODO](../TODO.md).
