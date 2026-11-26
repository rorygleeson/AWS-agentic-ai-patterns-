# Two-Gateway Architecture — Approved Baseline

**Decision:** Two AgentCore Gateways will be used for a controlled comparison of API integration and downstream identity.  
**Approved by:** Rory, 2026-11-26  
**Task:** S1-04  
**Implementation:** Not yet deployed; capabilities and identity flow must be tested.

## Side-by-side design

| | Gateway 1 — AWS-native REST | Gateway 2 — OpenAPI + OAuth |
|---|---|---|
| Target type | Amazon API Gateway REST API **stage target** | **OpenAPI specification target** |
| Target configuration | Reference REST API ID/stage, resource selection | OpenAPI contract with `operationId` per operation |
| Downstream authentication | IAM/SigV4 (selected) | OAuth 2.0 (selected for investigation) |
| Identity focus | AWS application/service identity | Delegated user / OBO experiment |
| APIs exposed | Fictional Network API: read status and request diagnostic | Same two logical Network API operations |
| Tools expected | `get_service_status`, `request_diagnostic` | Same operation concepts; verify actual discovery/names |
| Key learning outcome | Native API Gateway export, IAM boundary, tool mapping | Explicit schema, OAuth target credential behaviour, true user delegation if supported |

## Intended call path

```mermaid
flowchart TD
  U["Test user or machine client"] --> W["Web app / API entry"]
  W --> R["AgentCore Runtime — Strands agent"]
  R --> G1["Gateway 1 — API Gateway REST target"]
  R --> G2["Gateway 2 — OpenAPI target"]
  G1 --> API["Fictional Network API"]
  G2 --> API
```

Logical diagram only. The exact API deployment, authentication boundary and routing to shared or separate testing endpoints remain an implementation choice. No direct assertion is made that a single deployed endpoint supports both inbound IAM and delegated OAuth without additional configuration.

## Why two Gateways?

1. Compare **two different target integration approaches** independently, while holding the logical API operations constant.
2. Independently configure and inspect **inbound authorisation**, Gateway execution roles, policy attachment and security boundaries. One Gateway can hold multiple targets with their own outbound credential providers; using two is **not a product requirement for mixing IAM and OAuth**.
3. Make the **downstream credential model** visible: Gateway 1's IAM-backed application identity versus Gateway 2's OAuth delegated-identity goal.
4. Trace `tools/list`, `tools/call`, tool schema generation and access outcomes on each.
5. Show that two Gateways are a **deliberate learning choice**, not a general production requirement.

## Critical security distinction

**Target format and identity pattern are orthogonal.**

- The REST API stage target defines *how* AgentCore Gateway discovers and calls API Gateway operations. Its documented outbound options include IAM and API key, not OAuth.
- The OpenAPI target defines *how* REST operations become tools from a supplied contract. OAuth support at the target is **not itself proof** that the original interactive user is represented at the final API.
- **True OBO** requires a supported and explicitly configured token exchange/delegation path, the correct audience for the downstream API, validated user claims, and enforcement at the resource API. Until end-to-end tests verify these, label Gateway 2 **OAuth/OBO experiment**, not “OBO implemented.”
- Gateway 1's service identity cannot establish which interactive user initiated a call. Fine-grained User A/User B restrictions must be enforced by trusted controls before the service-identity invocation; direct API bypass must also be controlled.
- Both integrations must still enforce the agreed permission matrix: User A read+diagnostic, User B read-only, machine client read-only.

## Alternatives and trade-offs

| Alternative | Reason not chosen for this PoC |
|---|---|
| One Gateway with both targets | Simpler and often sufficient operationally, but less clear separation for comparison |
| Two identical Gateways | Adds resources without teaching a distinct integration/authentication boundary |
| Only OpenAPI target | Would omit the native API Gateway stage integration behaviour |
| Only API Gateway stage target | Would not demonstrate the OAuth-protected API integration path |

**Costs and complexity:** Additional Gateway deployment, policy/credential configuration, logs, tests and potential cost. Review budget before creation.

## Validation gates

- [ ] Confirm AWS documentation and deployed account/Region support for both target types.
- [ ] Validate the REST stage target IAM/SigV4 call and operation-to-MCP tool mapping.
- [ ] Validate OpenAPI target credentials and actual OAuth token used downstream.
- [ ] Prove or explicitly rule out true user-delegated OBO for the selected downstream path.
- [ ] Ensure end-user restrictions cannot be bypassed through the service identity.
- [ ] Verify both Gateways' discovery and call traces with redacted evidence.

## Source documentation

- [AWS: API Gateway REST API stages as AgentCore Gateway targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [AWS: OpenAPI schema targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-schema-openapi.html)
- [AWS: VPC egress and supported outbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html)

This is a **recorded design decision**, not a completed implementation. Stage 2 will incorporate it into the standalone ADR document, and Stage 5/6 will verify the behaviour.
