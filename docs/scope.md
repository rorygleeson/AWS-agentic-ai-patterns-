# PoC Scope and Acceptance Criteria

**Status:** Scope baseline approved (Stage 1 / S1-01). Technical implementation details requiring verification remain **TBD**.

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

## Agreed demonstration scenario

Use one harmless fictional **Network API** for realistic, repeatable technical demonstrations without depending on a real provider or external paid API.

| Candidate operation | HTTP example | Proposed MCP tool | User A | User B |
|---|---|---|---|---|
| Read network service status | `GET /network/services/{id}` | `get_service_status` | Allow | Allow |
| Request a simulated diagnostic | `POST /network/services/{id}/diagnostics` | `request_diagnostic` | Allow | Deny |

These are **agreed example operations**, not implemented endpoints. We should validate whether the two API operations are represented as two tools for each target configuration. A single REST API may expose multiple endpoints while an MCP tool has its own declared name and input schema.

**Agreed actors:** Entra User A (both operations), Entra User B (read only), a read-only machine client (2LO), and a separate logical downstream service identity. All identities are synthetic/test-only.

## Authentication and authorisation scenario matrix (S1-05)

Each row is an intended **test**, not a claim that the underlying AWS/Entra integration already works.

| Scenario | Caller / entry token | Gateway path | Downstream identity and validation | Expected result |
|---|---|---|---|---|
| A — interactive read (3LO) | U-A, Entra user access token for entry API | G1 REST/IAM | AWS service principal; trusted upstream policy checks user A | Read allowed |
| B — restricted interactive call | U-B, Entra user access token for entry API | G1 REST/IAM | Same service identity; trusted authorisation **must deny before calling** | Diagnostic denied |
| C — interactive diagnostic | U-A, Entra user access token for entry API | G1 REST/IAM | Service identity; trusted policy allows U-A | Diagnostic allowed |
| D — app-only (2LO) | M-1 client-credentials token for entry API | G1 REST/IAM | App/workload, no human subject; read-only permission | Read allowed; diagnostic denied |
| E — OBO read | U-A or U-B token for entry API | G2 OpenAPI/OAuth | New token **for Network API** with validated delegated user context, if exchange succeeds | Read allowed |
| F — OBO diagnostic | U-A and U-B tested separately | G2 OpenAPI/OAuth | Network API validates downstream token audience and delegated permissions | U-A allowed; U-B denied |
| G — OAuth without actual OBO | Test caller | G2 OpenAPI/OAuth | Inspect whether target uses app-only or user-delegated token | **Record actual behaviour; never claim OBO without evidence** |
| H — bad token / bypass | Forged, expired or wrong-audience token; direct API attempt | Both paths | Reject at validating entry, policy or downstream API | Access denied |
| I — policy | User/tool claims | Both paths / Cedar exercise | Trusted policy decision inputs, not prompt text | Predictable allow/deny and auditable reason |

**Identity handoff checkpoints:** Browser/M-1 → entry API (validate token) → Runtime (verified user/app context; secure invocation) → Gateway (authorised caller and tool policy) → Network API (validate SigV4 service identity *or* exchanged OAuth token). Precise hop-by-hop token forwarding is a Stage 2 design/Stage 6 verification task.

### REST target versus OpenAPI target (S1-06)

| Question | G1: API Gateway REST stage | G2: OpenAPI target |
|---|---|---|
| How are tools defined? | Derived from the selected AWS API Gateway REST stage/resources | From explicit OpenAPI operations and `operationId` |
| Downstream auth in this PoC | Gateway IAM role/SigV4 | OAuth credential provider; test token-exchange/OBO |
| Does the target format itself select 2LO/3LO/OBO? | **No** | **No** |
| How many operations? | Two logical API operations; validate exposed tools | Same two operations; validate exposed tools |
| Who enforces user-level permissions? | Trusted gateway/policy entry because API sees service identity | Network API after properly delegated exchange, plus trusted gateway/policy |
| Key proof | IAM-principal identity, tool mapping, policy denial | Token audience/delegated claims, tool mapping, policy denial |

AWS supports both target types, but outbound auth support differs by target. A single Gateway could host both; separate Gateways are our accepted trust-isolation choice ([ADR-001](architecture-decisions.md#adr-001--use-two-agentcore-gateways-for-separate-security-boundaries)).

## Acceptance criteria (S1-08) — implementation tests, not yet passed

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

**Additional explicit negative checks:** U-B and M-1 cannot request diagnostics through either Gateway; a forged/wrong-audience/expired token is rejected; service-account privileges cannot be used to bypass per-user policy; no raw bearer token is recorded in logs. Tests must capture redacted traces and distinguish an unsuccessful OBO experiment from working delegation.

## Out of scope

Production telecom integrations; customer data; paid telecom APIs; elaborate UI; production tenancy; customer onboarding; a broad benchmark across AI frameworks; unrelated new auth patterns; and any assertion that this is a production security reference architecture without separate hardening.

## Decisions still needed

1. The fictional Network API and read/diagnostic permission split are **approved**. Keep the entire project vendor-neutral; do not mention any real telecommunications company by name.
2. Agree each Gateway's distinct use case after validating current product constraints.
3. Confirm concrete REST API vs OpenAPI target support, credentials and tool mapping from up-to-date AWS documentation.
4. Decide the specific Entra tenant/application layout and AWS Region.
5. Set budget and cleanup parameters before provisioning anything.

See [master TODO](../TODO.md). No cloud resources have been deployed.
