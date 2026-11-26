# PoC Scope and Acceptance Criteria — Draft

**Status:** Draft for review (Stage 1 / S1-01). This captures agreed intent; implementation details requiring technical verification are deliberately marked **TBD**.

## Purpose

Build a small, working AWS AI-agent application to learn — and demonstrate — the full authentication, authorisation and API-tool integration journey.

**Reference path:** Browser chat → API Gateway → Amazon Bedrock AgentCore Runtime (Python Strands + Bedrock) → AgentCore Gateway → MCP tool discovery/invocation → downstream REST APIs.

The emphasis is **identity and access decisions at each hop**, not a sophisticated chatbot or production UI.

## Agreed learning outcomes

1. **Interactive identity:** Sign in via Microsoft Entra ID with OIDC/OAuth 2.0; distinguish authentication from OAuth resource authorisation.
2. **Machine identity:** Demonstrate a client-credentials (2LO) flow independently from the interactive user flow (3LO).
3. **Token trust:** Understand JWT signature, issuer, audience, expiry, scopes/roles and the validation performed by each receiving component.
4. **OBO:** Show a delegated/token-exchange path preserving the real user's authorisation context into a downstream API *where the products and token configuration actually support it*.
5. **Service account:** Compare an application/service identity used for downstream API calls and explicitly show where end-user identity is lost.
6. **Two gateways:** Use two AgentCore Gateways to compare responsibilities/configuration; do not claim they are inherently required for production.
7. **MCP:** Observe tool discovery, schemas, JSON-RPC `tools/list` and `tools/call`, selection and execution.
8. **Integration targets:** Compare a REST API target and an OpenAPI-defined target. Tool/operation mapping is separate from OBO/service-account authorisation.
9. **Fine-grained access:** Interactive User A can use both designated operations; User B can use only the allowed operation; unauthorised attempts fail.
10. **Cedar:** Explore a concrete policy-based access decision and compare it with simpler role/scope restrictions; avoid assuming Cedar is mandatory.
11. **Explainability:** Provide diagrams, recorded decisions, a pattern catalogue and an end-user Q&A.

## Proposed demonstration scenario (requires confirmation)

Use one harmless fictional **Network API** to support interview-relevant demonstrations without relying on a real telecom provider or external paid API.

| Candidate operation | HTTP example | Proposed MCP tool | User A | User B |
|---|---|---|---|---|
| Read network service status | `GET /network/services/{id}` | `get_service_status` | Allow | Allow |
| Request a simulated diagnostic | `POST /network/services/{id}/diagnostics` | `request_diagnostic` | Allow | Deny |

These are **proposed names**, not implemented endpoints. We should validate whether the two API operations are represented as two tools for each target configuration. A single REST API may expose multiple endpoints while an MCP tool has its own declared name and input schema.

**Proposed actors:** Entra User A (both), Entra User B (read only), and an Entra application/service principal for M2M and service-identity comparison. All identities synthetic/test-only.

## Pattern/test matrix

| Test | Caller | Downstream identity | Expected evidence |
|---|---|---|---|
| Interactive authorised read | User A or B (3LO) | Delegated/OBO where supported | Accepted token and successful read |
| Restricted mutation | User B (3LO) | Delegated or service identity | Denied by effective authorisation control |
| Permitted mutation | User A (3LO) | Delegated/OBO where supported | Diagnostic accepted and traced |
| Machine-to-machine | Client credentials (2LO) | Application identity | Scoped operation succeeds; user delegation not implied |
| Service account comparison | Interactive user → agent | Shared application identity | API sees service identity; policy layer still protects user restriction |
| OpenAPI target | Selected caller | Explicitly configured | Generated/discovered operation schema and call |
| REST API target | Selected caller | Explicitly configured | Exposed tool schema and call; exact support verified |
| Wrong/expired token | Any | None | Rejected at relevant validating hop |
| Cedar policy | A/B with tool metadata | As designed | Deterministic allow/deny and explanation |

**Important:** In a service-account pattern, the downstream API cannot infer the end user solely from the application token. User-based restrictions must be enforced by a trusted layer before invoking it. Do not claim that a Gateway target type automatically changes OAuth delegation.

## Initial success criteria

- [ ] Two interactive test users can authenticate; their effective permissions differ.
- [ ] One machine client can obtain a correctly scoped token and invoke only its intended flow.
- [ ] API Gateway and downstream APIs reject invalid issuer, audience, signature, expiry or insufficient authority as appropriate.
- [ ] AgentCore Runtime responds and makes an actual tool invocation through the configured Gateway(s).
- [ ] Both Gateways are present and their separate purpose is visible in the configuration and logs.
- [ ] REST API and OpenAPI target experiences are verified against current AgentCore product capabilities.
- [ ] Raw MCP discovery and invocation examples are captured with credentials redacted.
- [ ] OBO and service-account runs show the actual token subject/claims trusted at each hop; limitations are documented.
- [ ] User B cannot invoke the protected operation, including attempted direct API access.
- [ ] Cedar example demonstrates an auditable allow/deny case.
- [ ] Deploy, test, observe, and clean up are reproducible with documented cost boundaries.
- [ ] Architecture, pattern catalogue, standalone ADR and Q&A accurately reflect the running solution.

## Out of scope

Production telecom integrations; customer data; paid telecom APIs; elaborate UI; production tenancy; customer onboarding; a broad benchmark across AI frameworks; unrelated new auth patterns; and any assertion that this is a production security reference architecture without separate hardening.

## Decisions still needed

1. Confirm the fictional Network API and the read/diagnostic permission split.
2. Agree each Gateway's distinct use case after validating current product constraints.
3. Confirm concrete REST API vs OpenAPI target support, credentials and tool mapping from up-to-date AWS documentation.
4. Decide the specific Entra tenant/application layout and AWS Region.
5. Set budget and cleanup parameters before provisioning anything.

See [master TODO](../TODO.md). No cloud resources have been deployed.
