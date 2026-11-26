# Master TODO — AWS Agentic AI Patterns

> **Single source of truth / project board.** This file tracks **all currently known work** from design to cleanup. Add newly discovered work here **before doing it**; never silently remove earlier decisions or tasks.

## Board rules

1. **One place:** Every planned task appears below under a stage, with a stable ID and checkbox. Subtasks can be inserted when needed.
2. **Progress:** `[ ]` planned, `[x]` completed with evidence, `⏳` active stage, `⏸` deferred/blocked (record why). Do not claim completion from discussion alone.
3. **Next up:** Update the **Current focus** section whenever work moves forward; keep only one immediate next task.
4. **Stage gates:** Complete the stage's checks before moving ahead, or explicitly record a dependency/deferral. Stages can overlap only when documented.
5. **Change control:** Append new stages as needed; revise tasks transparently. Record decisions and alternatives in the separate ADR. Preserve completed history.
6. **Verification:** Link evidence (doc, diagram, code, test, cloud resource or run) beside completed tasks. Keep documentation aligned with the working implementation.
7. **No surprise scope:** The existing agreed patterns are the baseline. New patterns need an explicit decision before adding them.
8. **Safety and cost:** No cloud deployment or paid services without reviewing scope, expected costs, budget alarms and cleanup approach.

## Current focus

- **Active stage:** Stage 1 — Scope & inventory
- **Immediate next task:** `S1-04` Define and verify the distinct roles and target assignment of the two AgentCore Gateways.
- **Implementation status:** No PoC infrastructure deployed.
- **Last board update:** 2026-11-26

## Stage overview

| Stage | Outcome | Status |
|---|---|---|
| 0. Foundation | Short README and master task board | ✅ Complete |
| 1. Scope & inventory | Frozen learning goals, actors, tools and scenarios | ⏳ In progress |
| 2. Architecture & decisions | Diagrams, patterns, ADR, security model and Q&A | Planned |
| 3. Accounts & deployment foundations | AWS/Entra prerequisites, IaC, budget controls | Planned |
| 4. Identity & application entry | Web chat, sign-in, API Gateway, Runtime agent | Planned |
| 5. Gateway, MCP & API integration | Two Gateways, discovery, REST/OpenAPI targets | Planned |
| 6. Identity propagation & authorisation | OBO, M2M, service account, restrictions, Cedar | Planned |
| 7. Verification & observability | End-to-end matrices, traces, failure testing | Planned |
| 8. Publication & handover | Reproducible walkthrough, documentation, demos | Planned |
| 9. Cleanup & cost review | Teardown checks and actual cost analysis | Planned |

## Stage 0 — Foundation

- [x] **S0-01** Create concise `README.md` describing the purpose and reference flow. Evidence: [README.md](README.md).
- [x] **S0-02** Create this single master `TODO.md` with stages, granular tasks and board rules. Evidence: this file.
- [x] **S0-03** Review board completeness with Rory and amend missing tasks or decisions. Evidence: Rory's go-ahead on 2026-11-26; new scope questions tracked in [docs/scope.md](docs/scope.md).

**Exit:** README and board exist; Rory approves the baseline.

## Stage 1 — Scope & inventory

- [x] **S1-01** Review and confirm all agreed PoC objectives, boundaries and acceptance criteria. Evidence: Rory approved fictional Network API scenario and vendor-neutral scope; [docs/scope.md](docs/scope.md).
- [x] **S1-02** Define named actors and identities: interactive User A, User B with restricted access, and a machine/service principal; confirm privileges. Evidence: Rory approved machine-client read-only; [docs/identities-and-permissions.md](docs/identities-and-permissions.md).
- [x] **S1-03** Inventory intended MCP tools, endpoints, sample downstream APIs and allowed operations; distinguish tool from HTTP endpoint. Evidence: [docs/tool-and-api-inventory.md](docs/tool-and-api-inventory.md); operational capabilities will be tested in Stage 5.
- [ ] **S1-04** Specify the two AgentCore Gateways' purposes, boundaries, target placements and comparative learning goals.
- [ ] **S1-05** Define the scenario matrix for interactive 3LO, client-credentials 2LO, service-account downstream access and OBO.
- [ ] **S1-06** Define REST API target and OpenAPI target scenarios; distinguish integration method from caller/downstream authentication.
- [ ] **S1-07** Establish realistic mock/sample API data and permitted writes so tests are safe and deterministic.
- [ ] **S1-08** Define non-goals and success criteria, including a demonstrable unauthorised-tool denial and token-validation failure.
- [ ] **S1-09** Confirm implementation region, model, AWS/Entra account ownership, budget and operational constraints.

## Stage 2 — Architecture & decisions

- [ ] **S2-01** Create `docs/architecture.md`: components, trust boundaries, deployment topology and end-to-end flow.
- [ ] **S2-02** Draw a clear high-level architecture diagram showing web app, API Gateway, Runtime, Strands/Bedrock, both Gateways, MCP and APIs.
- [ ] **S2-03** Draw sequence diagrams for user sign-in, token validation/propagation, discovery, invocation and response.
- [ ] **S2-04** Document JWT issuers, audiences, scopes, roles, validation points, token lifetimes and refresh/expiry handling.
- [ ] **S2-05** Document OBO token-exchange mechanics, downstream token audience and exactly where user identity is preserved versus terminated.
- [ ] **S2-06** Create `docs/patterns.md` with agreed patterns, prerequisites, examples, trade-offs and test expectations.
- [ ] **S2-07** Explain REST API versus OpenAPI Gateway target behaviour and how tools map to operations/endpoints.
- [ ] **S2-08** Create standalone `docs/architecture-decisions.md` (ADR collection) with status, context, choice, alternatives and consequences for each decision.
- [ ] **S2-09** Capture decisions already made: OBO preference, service-account comparison, two Gateways for learning, two users, REST/OpenAPI comparison, Cedar evaluation and no further patterns for now.
- [ ] **S2-10** Create `docs/security.md`: identity boundaries, least privilege, secrets, policy points, threat scenarios and logging/redaction.
- [ ] **S2-11** Create `docs/questions-and-answers.md` covering why Gateway, why two, MCP vs endpoint, OBO vs service account, REST vs OpenAPI, Cedar, JWT and 2LO/3LO.
- [ ] **S2-12** Cross-check all diagrams, documents and stage acceptance criteria; resolve open design decisions before implementation.

## Stage 3 — Accounts & deployment foundations

- [ ] **S3-01** Confirm AWS account access, Region support, IAM boundaries and required Bedrock/AgentCore quotas.
- [ ] **S3-02** Establish a free/eligible Entra tenant and app-registration prerequisites; verify actual licensing/cost assumptions.
- [ ] **S3-03** Register/configure the required Entra applications/service principals, redirect URIs, scopes, roles and consent (after design).
- [ ] **S3-04** Decide infrastructure-as-code approach and establish one repeatable deployment workflow.
- [ ] **S3-05** Configure credentials and secret storage securely; ensure no secrets enter Git.
- [ ] **S3-06** Set tagging, budget alerts, logging retention and an explicit teardown strategy before deployment.
- [ ] **S3-07** Create environment configuration examples, least-privilege deployment roles and setup instructions.
- [ ] **S3-08** Deploy minimal foundations and verify they can be redeployed safely.

## Stage 4 — Identity & application entry

- [ ] **S4-01** Build simple HTML/JavaScript chat UI with states for login, prompt, reply and errors.
- [ ] **S4-02** Integrate Entra OIDC interactive sign-in using appropriate authorisation-code/PKCE flow.
- [ ] **S4-03** Define and secure the API Gateway entry endpoint and browser CORS settings.
- [ ] **S4-04** Implement JWT validation, issuer/audience/scope checks and meaningful rejection responses at entry.
- [ ] **S4-05** Deploy a minimal Python Strands agent on AgentCore Runtime with Amazon Bedrock model access.
- [ ] **S4-06** Connect API Gateway to Runtime through an appropriate secured invocation integration; document credentials and identity handoff.
- [ ] **S4-07** Prove User A can sign in, submit a prompt and receive a basic agent response.
- [ ] **S4-08** Validate User B's entry behaviour, expired/malformed tokens and missing permissions.

## Stage 5 — Gateway, MCP & API integration

- [ ] **S5-01** Configure first AgentCore Gateway, inbound authorisation and Runtime-to-Gateway credential handling.
- [ ] **S5-02** Configure second AgentCore Gateway with documented distinction and boundary.
- [ ] **S5-03** Build/configure simple downstream REST APIs with sufficient endpoints to demonstrate separate tool permissions.
- [ ] **S5-04** Configure an AgentCore Gateway **OpenAPI target** with operations exposed as MCP tools.
- [ ] **S5-05** Configure an AgentCore Gateway **REST API target** and document any required adapter/tool-definition differences.
- [ ] **S5-06** Verify supported target types and exact AWS capabilities against current official docs before deployment; correct assumptions in docs.
- [ ] **S5-07** Demonstrate MCP discovery (`tools/list`), tool schemas and JSON-RPC invocation (`tools/call`).
- [ ] **S5-08** Connect Strands Runtime agent to both Gateways and demonstrate selection and calls.
- [ ] **S5-09** Show whether a tool maps to one or multiple downstream API operations, using actual Gateway configuration and traces.
- [ ] **S5-10** Handle upstream/downstream schema, validation, timeout and API errors.

## Stage 6 — Identity propagation & authorisation

- [ ] **S6-01** Implement and test machine-to-machine 2LO/client-credentials entry path.
- [ ] **S6-02** Implement and test service-account / application-identity downstream access.
- [ ] **S6-03** Implement and test end-to-end OBO/token exchange where supported; document constraints and every identity transition.
- [ ] **S6-04** Prove downstream API independently validates token issuer, audience, expiry and permissions.
- [ ] **S6-05** Enforce a permission matrix: User A accesses both agreed tools; User B only the permitted tool; M2M has explicitly scoped access.
- [ ] **S6-06** Clarify which controls live at API Gateway, AgentCore Gateway, policy layer and downstream API.
- [ ] **S6-07** Define Cedar policy decision inputs, claims mapping and deny-by-default policy examples.
- [ ] **S6-08** Implement a focused Cedar authorisation demonstration and compare it against simpler native/scoped controls.
- [ ] **S6-09** Test denial, token substitution, wrong audience, privilege escalation and service-account confused-deputy scenarios.
- [ ] **S6-10** Record actual user/agent/service identities seen at each hop for every pattern.

## Stage 7 — Verification & observability

- [ ] **S7-01** Build a traceable test matrix covering every agreed pattern and user/tool/endpoint combination.
- [ ] **S7-02** Add repeatable unit, integration and end-to-end test scripts with expected outcomes.
- [ ] **S7-03** Capture token/identity flow metadata safely without logging secrets or raw bearer tokens.
- [ ] **S7-04** Configure and demonstrate AgentCore/CloudWatch logs, metrics, tracing and correlation IDs.
- [ ] **S7-05** Demonstrate MCP discovery and tool-call traffic with redacted JSON-RPC examples.
- [ ] **S7-06** Exercise authentication failures, authorisation denials, expired tokens, rate limits, service errors and timeouts.
- [ ] **S7-07** Review security, IAM scope, secret handling and cost against the design.
- [ ] **S7-08** Update architecture, patterns and ADRs to match observed behaviour; clearly label unsupported or simulated functionality.

## Stage 8 — Publication & handover

- [ ] **S8-01** Write a concise reproducible setup/run/test/teardown guide.
- [ ] **S8-02** Add sanitised configuration examples, API contracts and safe sample requests/responses.
- [ ] **S8-03** Prepare one demonstration walkthrough per agreed pattern with expected results.
- [ ] **S8-04** Polish architecture diagrams and README links for public GitHub readers.
- [ ] **S8-05** Final consistency review: all claims evidenced, all links work, no secrets or identifiers exposed.
- [ ] **S8-06** Create final outcomes, lessons learned, known limitations and interview-ready summary.

## Stage 9 — Cleanup & cost review

- [ ] **S9-01** Inventory deployed resources, recurring charges and retained data.
- [ ] **S9-02** Execute approved cleanup/teardown after demos, preserving required redacted evidence.
- [ ] **S9-03** Check for remaining resources, secret registrations, orphaned log groups and billable services.
- [ ] **S9-04** Record actual AWS and Entra costs and operational lessons.
- [ ] **S9-05** Mark project complete or create an explicitly scoped new stage for future work.

## Decisions and blockers register

| ID | Item | Status / next action |
|---|---|---|
| D-01 | Preserve all earlier agreed patterns; no new ones for now | Baseline; detail in ADR |
| D-02 | Two Gateways are included for comparative learning | Define roles in S1-04 |
| D-03 | Prefer OBO for preserving user identity; compare with service-account model | Validate exact supported flows |
| D-04 | REST API vs OpenAPI target comparison is **orthogonal** to OBO vs service-account credentials | Verify capabilities in S5-06 |
| D-05 | User A and User B have differing tool permissions | Finalise matrix in S1-02/S6-05 |
| D-06 | Cedar is an authorisation-learning exercise, not assumed mandatory for every tool | Confirm policy enforcement design |
| D-07 | Free Entra ID tenant is intended for the PoC | Verify eligibility and app registration requirements |
| D-08 | Document architecture, patterns, standalone ADR and Q&A separately | Stage 2 |
| D-09 | Fictional Network API with read and diagnostic operations; no real provider names or interview framing anywhere in project | Approved 2026-11-26 |
| D-10 | Machine client (2LO) is read-only | Approved 2026-11-26 |
| D-11 | Native API Gateway REST stage target currently uses IAM or API key, not OAuth outbound; compare with OpenAPI target for OAuth | AWS documentation reviewed; verify in Stage 5 |

## Future discoveries / new stages

Add new tasks or stages here when they emerge, assign stable IDs, then update the overview and current focus. Do **not** treat unscheduled ideas as completed project work.
