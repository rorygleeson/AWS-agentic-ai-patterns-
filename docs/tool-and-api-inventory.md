# Tool and API Inventory

**Stage:** 1 / S1-03  
**Status:** Initial inventory recorded; service capabilities are sourced from AWS documentation and will be verified by deployment.

## Logical API and operations

One fictional **Network API**, with two operations and no external provider dependency:

| ID | Method / resource | Logical operation | MCP tool name (candidate) | Permission | Side effect |
|---|---|---|---|---|---|
| API-01 | `GET /network/services/{id}` | Read service status | `get_service_status` | `network.status.read` | None |
| API-02 | `POST /network/services/{id}/diagnostics` | Request simulated diagnostic | `request_diagnostic` | `network.diagnostics.request` | Creates mock diagnostic record only |

**API operation ≠ MCP tool by definition.** A REST API can expose multiple endpoints. A Gateway target can expose qualifying operations as individual MCP tools; exact names and visibility depend on target schemas, filters and product capabilities. No need to build a separate REST API for every tool.

## Mock/sample API contract (S1-07) — design baseline

### `GET /network/services/{id}`

- Input: `id` (fictional service identifier, e.g. `svc-001`)
- Output: `{ "serviceId": "svc-001", "status": "operational", "region": "test-zone" }`
- Test records: `svc-001` operational, `svc-002` degraded, unknown ID not found.

### `POST /network/services/{id}/diagnostics`

- Input: `id`, `{ "checkType": "connectivity" }`
- Output: `{ "diagnosticId": "diag-001", "serviceId": "svc-001", "result": "scheduled" }`
- Creates a synthetic diagnostic outcome only. No actual network calls, customer records or infrastructure changes.

**Deterministic fixtures and safety rules:** Use only `svc-001` (operational) and `svc-002` (degraded); unknown IDs return a consistent not-found response. A diagnostic request creates only a fictional record with a deterministic test identifier and `scheduled` status. Reject unsupported `checkType`, missing IDs, unauthorised callers and duplicate or malformed inputs predictably. No real network diagnostics, external network integrations or personal data. Final OpenAPI shapes and error codes are implementation details to test.

## Target comparison — same logical operations

| Dimension | Native API Gateway REST API stage target | OpenAPI specification target |
|---|---|---|
| Source of tool definitions | API Gateway REST API stage export / selected resources | Explicit OpenAPI 3.x document |
| MCP exposure | Operations surfaced as tools, subject to filters | `operationId` determines exposed tool name |
| Example API operations | API-01 and API-02 | API-01 and API-02 |
| Downstream authentication | AWS IAM/SigV4 or API key per current AWS guidance | Credential provider; OAuth or API-key patterns to investigate |
| Learning focus | AWS-native stage-to-tool mapping, service credentials | Explicit API contract and OAuth/OBO feasibility |

**Critical constraint from AWS:** The native API Gateway REST API target does **not** currently support OAuth-based outbound authorisation. For an OAuth-protected API Gateway endpoint, AWS directs use through the OpenAPI target instead. Do not treat target type as an intrinsic 2LO/3LO/OBO setting. An actual delegated OBO exchange must be independently designed and proven.

Relevant AWS references:
- [API Gateway REST API stages as targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [OpenAPI schema targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-schema-openapi.html)
- [Gateway VPC egress / authentication limitations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html)
- [Using MCP with AgentCore Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using.html)

## Proposed deployment inventory

| Component | Count | Purpose | Confirmed? |
|---|---:|---|---|
| Fictional Network API | 1 | Host two test operations | Agreed |
| HTTP operations | 2 | Read and mock diagnostic | Agreed |
| Logical MCP tools | 2 per target (to validate) | Independently authorisable operations | To verify |
| AgentCore Gateways | 2 | Compare target integration/security | Agreed |
| API Gateway REST stage target | 1 | Native integration | Agreed |
| OpenAPI target | 1 | Explicit contract / OAuth experiment | Agreed |
| Test user principals | 2 | Operator and Viewer | Agreed |
| Machine client principal | 1 | 2LO, read-only | Agreed |
| Downstream service identity | 1 logical role | Compare app identity vs delegated OBO | Agreed for comparison |
| Runtime / Strands agent | 1 | Discover and invoke both target families | Agreed |

Gateway assignment is approved: Gateway 1 = REST stage/IAM, Gateway 2 = OpenAPI/OAuth experiment. See [technical design](two-gateway-design.md) and [ADR-001](architecture-decisions.md#adr-001--use-two-agentcore-gateways-for-separate-security-boundaries).

## Candidate policy evidence

- U-A can call both read and diagnostic.
- U-B can call read but not diagnostic.
- M-1 can call read but not diagnostic.
- A service-account token alone does not prove U-A/U-B identity.
- OBO is proven only if a downstream API receives a correctly exchanged user-delegated token for its audience.
- Both target types expose the intended tool schemas through MCP discovery, with observed behaviour documented.

See [identity matrix](identities-and-permissions.md) and [master board](../TODO.md).
