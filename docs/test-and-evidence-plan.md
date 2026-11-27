# Acceptance tests and evidence plan

**Status: Planned, not executed.** Every test below is a design acceptance criterion. Positive tests alone are insufficient: inspect the **actual downstream principal**, response and denial as well as the agent's text.

**Consolidated API design:** Asset status, diagnostics, ticket creation and aggregate health are endpoints of the **same Network Operations REST API** backed by RDS. Each endpoint remains a separately authorized MCP tool; ticket writes are not a new end system.

## Personas and system-access tests

| Test | Caller | Request | Expected result | Boundary being proven |
|---|---|---|---|---|
| T01 | Sarah | Search shared operational guides | Cited synthetic RAG content | All-human RAG |
| T02 | James | Search same guides | Same permitted corpus | Shared RAG |
| T03 | Emma | Search same guides | Allowed | Shared RAG |
| T04 | David | Search same guides | Allowed | Shared RAG |
| T05 | Oliver | Search same guides | Allowed | Shared RAG |
| T06 | Sarah | NSW/VIC asset diagnostics | Both permitted | National entitlement + OBO |
| T07 | James | VIC asset diagnostics | VIC records only | Delegated James OBO + RLS |
| T08 | James | NSW diagnostics, including prompt injection | **Denied** at tool/API/DB | Region restriction |
| T09 | Emma | National basic asset status | Basic fields only | DTO/field filtering |
| T10 | Emma | Diagnostics, tickets or sensitive fields | **Denied** even with crafted arguments | Operation restriction |
| T11 | Sarah | Create ticket after explicit confirmation | Once only; authorized region | Write confirmation + OBO |
| T12 | James | Create VIC ticket after confirmation | Once only | Regional write |
| T13 | James | Create NSW ticket or replay same idempotency key | **Denied** / no duplicate | Downstream write control |
| T14 | David | Vendor alert request, no initial consent | Vendor auth/consent requested, no data leakage | 3LO first use |
| T15 | David | Vendor alerts after consent | Only David's assigned alerts | Independent vendor user |
| T16 | Sarah/James/Emma | David's vendor alert endpoint | **Denied** or require independent, individually granted vendor consent | No shared vendor token |
| T17 | Oliver | Platform health | Synthetic status permitted | Native API GW stage + IAM |
| T18 | Any other human | Platform health | **Denied by G1 Cedar** | Tool allow-list |
| T19 | Reporter M2M | Scheduled national aggregate health | Aggregate only | G2 IAM + 2LO |
| T20 | Reporter M2M | Raw rows / diagnostics / ticket tool | **Denied** | App-only scope and G2 inventory |
| T21 | Human | Direct G2 IAM tool call using JWT only | **Denied** | Separate inbound authorizer |
| T22 | Automation | Direct G1 access with IAM only | **Denied** | Separate inbound authorizer |
| T23 | Caller | Bypass API Gateway to invoke Runtime U | **Denied** unless explicitly authorized private route | Ingress policy |
| T24 | Caller | Send token with wrong audience / expired / altered signature | **Denied** at corresponding API/runtime | JWT validation |
| T25 | Any caller | Change JSON `userId` or agent instruction to "I am Sarah" | No elevation | Server-authenticated principal |
| T26 | One human | Reuse another user's runtime session identifier | **Denied** | Session principal binding |
| T27 | Runtime M | G2 tool invocation | Cloud logs record **Runtime M role** | Correct IAM caller |
| T28 | Oliver | Platform Health downstream invocation | Backend authorizes **Gateway execution IAM role**, not "Oliver" | Native target semantics |
| T29 | James | Network API diagnostics | Downstream audit shows James `tid/oid` and proper Network audience | OBO end-to-end |
| T30 | Reporter | Network API summary | Downstream audit shows app ID/`roles`, **not** employee `oid`/`scp` | 2LO subject |
| T31 | David | Vendor revocation and retry | Vendor data access ceases | Token revocation |
| T32 | Emma/James | SQL context attempted in tool args; pooled request after another user | No broadened rows; context isolated | RLS session safety |
| T33 | Entra Automation Client | Invoke G1 using AI-backend-audience Client Credentials JWT, request summary | Allowed; aggregate only | Non-human JWT inbound |
| T34 | Entra Automation Client | Attempt human status/diagnostics/ticket/RAG/Platform tools | **Denied** at G1 | App-only Cedar permission |
| T35 | Entra Automation Client | Reuse its inbound AI JWT directly against Network API | **Denied** due to wrong audience | Separate JWTs for separate APIs |
| T36 | AWS Reporter IAM identity | Attempt G1 invocation without a JWT | **Denied** | G1 JWT versus G2 IAM inbound |
| T37 | Both automated callers | Request identical summary; inspect backend audit identities | Same aggregate only; two **different** app IDs and credentials | Two separate non-human callers |
| T38 | Entra Automation Client | AI Backend token has valid signature/audience but **no** assigned `Automation.Summary.Invoke` app role | **Denied** from summary tool | Backend machine app-role enforcement, not signature alone |
| T39 | Unapproved Entra service principal | Request AI Backend JWT and G1 summary with no approved client assignment | **Denied** at client/role authorization | G1 client allow-list and Cedar default-deny |
| T40 | Human | Valid AI Backend JWT with missing `chat.invoke` delegated scope | **Denied** before human tool call | Chat API delegated-scope enforcement |


## What evidence is required?

For every test, collect the following without exposing tokens or personal details:

1. **Input:** scenario, user/service pseudonym, relevant region/tool parameters, correlation ID.
2. **Gateway decision:** authenticated subject and Cedar allow/deny with tool/action/arguments.
3. **Downstream:** API name, token issuer/audience and *type* (`scp` or `roles`); sanitized identity key; API decision.
4. **Database:** RLS policy that applied, permitted query count/region set; no secret values.
5. **Result:** HTTP status, redacted response schema/fields and whether a mutation occurred.
6. **Audit:** matching request/session/tool/API IDs and logs that omit access/refresh tokens.
7. **Outcome:** Pass/Fail/Blocked, date, exact service/version and linked redacted evidence.

For native stage T28, specifically show the **change in identity**: Oliver authorizes MCP access but downstream AWS IAM identity is Gateway execution role. For OBO T29, show **preservation of user identity** at the Network API through an audience-specific delegated token. This is the centerpiece of the article.

## Pattern coverage

| Pattern | Main evidence tests |
|---|---|
| P01 SSO | T01–T05, T24 |
| P02 Lambda propagation | T23, T25–T26, T29 |
| P03 OBO | T06–T13, T29 |
| P04 Client Credentials | T19–T20, T30, T33–T37 |
| P05 3LO | T14–T16, T31 |
| P06 Native REST stage | T17–T18, T28 |
| P07 OpenAPI OAuth REST | T06–T13, T29 |
| P08 G1 JWT/Cedar | T08, T10, T18, T22, T33–T35 |
| P09 G2 IAM/Cedar | T19–T21, T27 |
| P10 API + RLS | T06–T13, T32 |
| P11 Shared RAG | T01–T05 |
| P12 Runtime ingress | T21–T26 |
| P13 Safe write | T11–T13 |
| P14 Audit | T27–T30 and all scenarios |

## Exit criteria

- Every agreed end system has at least one success and one applicable denial scenario.
- Identity-preserving OBO, IAM service execution, M2M app-only and independent vendor 3LO are **distinguishable in recorded downstream evidence**.
- Row-level and response-field access controls are independently proven.
- No unauthenticated/mis-audienced/token-substitution shortcut works.
- No unconfirmed or duplicate ticket is committed.
- Secrets/tokens are absent from screenshots, logs and Git.
- Any blocked unsupported AWS integration is recorded honestly; no "assumed pass".
- Costs and cleanup completed after final evidence.

## Result log template

| ID | Date | Outcome | Observed subject/audience | Expected / observed | Redacted evidence | Follow-up |
|---|---|---|---|---|---|---|
| T01 | — | Not run | — | Shared RAG expected | — | — |

Do not change `Not run` to `Pass` until evidence exists.
