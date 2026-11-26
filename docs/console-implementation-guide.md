# AWS + Entra console implementation guide

**Purpose:** A deliberately separate, console-first learning plan. **No IaC.** This is a configuration checklist, not a claim that anything has been deployed or tested. Follow the [architecture baseline](architecture-baseline.md) and [ADRs](architecture-decisions.md).

## Preflight

- Use a disposable AWS account, fixed region, cost/budget alarms and distinct names prefixed `poc-`.
- Use a nonproduction Entra tenant and entirely synthetic data; set short credential lifetime and track secrets safely.
- Confirm quotas, feature/region availability and the precise supported AgentCore Identity/Gateway integrations against the linked vendor docs before entering secrets.
- Record every resource name, region, ARN, URL, principal, issuer, audience and teardown dependency in a private deployment log. Do **not** commit secrets, tokens or identity PII.
- Configure CloudTrail, CloudWatch, tracing and synthetic audit correlation early.

## Build order, in console

### Phase 1 — End-system fixtures

1. Create private RDS PostgreSQL with small assets/diagnostics/tickets and entitlement tables. Use synthetic VIC, NSW and national regions. Keep database unreachable from public internet.
2. Create least-privileged database API roles, RLS policies and a service-side entitlement mapping keyed by Entra `tid/oid` and automation app identity. Never use a table-owner/BYPASSRLS role for application requests.
3. Deploy a small Network REST API exposing `get_asset_status`, `get_diagnostics`, `get_network_health_summary` against RDS. Build independent delegated versus app-only authorization.
4. Deploy a separate Ticket REST API exposing `get_ticket` and `create_ticket`. Support confirmation state, region authorization and idempotency.
5. Deploy Platform Health as **AWS API Gateway REST API stage** backed by a tiny synthetic endpoint (e.g. Lambda), secured to accept IAM/SigV4 calls from the Gateway execution role. Its response is fictional health, not real infrastructure diagnostics.
6. Create a small non-sensitive S3 guide corpus, then a Bedrock Knowledge Base/vector store. All human identities may access its search tool.
7. Create a fictional NetworkMonitor external vendor API and a separate Cognito user pool/app client. Configure authorization-code + PKCE/consent for per-David `alerts.read`. This is a *simulation of a third-party service*, not actual vendor integration.

### Phase 2 — Entra identity setup

1. Register `enterprise-ai-backend` as the protected API. Expose `chat.invoke`; configure JWT issuer and audience.
2. Register `enterprise-ai-web` as the browser/public client, with precise redirect URIs and authorization-code + PKCE. Grant delegated `chat.invoke`.
3. Register `enterprise-network-api` with delegated `asset.read` and `diagnostics.read`, plus app permission `HealthSummary.Read`. Assign that application permission only to the reporter app.
4. Register `enterprise-ticket-api` with delegated `ticket.read` and `ticket.create`.
5. Register the confidential middle-tier/OBO credential and `network-health-reporter` service principal/client. Configure exact audience, consent and allowed credential handling.
6. Create Sarah, James, Emma, David and Oliver nonproduction accounts. Assign app roles corresponding to agreed personas; optionally map trusted group object IDs server-side. **Groups are not OAuth scopes.**
7. Check API delegated scopes against downstream OBO exchanges. Enforce `scp` for delegated calls and `roles` + app identity for app-only calls. No universal bearer token.

### Phase 3 — Human Runtime U and G1

1. Deploy AgentCore Runtime U with JWT inbound authorization referencing the Entra issuer and AI backend audience. Configure header allow-list forwarding and session binding per documented capability.
2. Restrict resource policy and networking to the intended private Lambda/VPC route; verify runtime isn't accidentally invokable on an alternative trust path.
3. Create G1 with inbound `CUSTOM_JWT`. Point to the Entra JWT issuer/audience. Register only human tools:
   - Managed Knowledge Base target `search_knowledge` — IAM outbound.
   - OpenAPI Network target `get_asset_status`, `get_diagnostics` — Entra OBO.
   - OpenAPI Ticket target `get_ticket`, `create_ticket` — Entra OBO.
   - OpenAPI external NetworkMonitor target `search_vendor_alerts` — independent vendor 3LO.
   - **Native AWS API Gateway REST API stage target** `get_platform_health` — IAM outbound.
4. Define Cedar policy by actual subject role and operation/arguments. Sarah nationally authorized, James VIC only, Emma basic asset only, David only vendor, Oliver only platform health plus RAG.
5. Deploy thin private Lambda ingress; put AWS API Gateway JWT authorizer in front. Lambda validates request schema, logs safe principal/correlation details, binds session ID to authenticated subject and forwards original token to Runtime U over PrivateLink. Do not rely on Lambda IAM role for human downstream permissions.

### Phase 4 — Machine Runtime M and G2

1. Deploy a **separate IAM-authenticated** Runtime M and limit its invocation to the reporter trigger role.
2. Create **separate** G2 with inbound `AWS_IAM`, exposing only `get_network_health_summary` through an OpenAPI target.
3. Configure target outbound Entra **client-credentials (2LO)** app identity with only `HealthSummary.Read`. Give API a distinct app-only policy path, no user OBO.
4. Configure Cedar to allow precisely Runtime M's execution role and summary tool. Note this is not necessarily the upstream Reporter Lambda's role.
5. Create EventBridge Scheduler → Reporter Lambda → Runtime M IAM invocation. Log caller and resultant report delivery as synthetic output.

### Phase 5 — Tightening

- Enforce exact issuer, token signature, audience, expiry, `scp`/`roles` and app/client restrictions at each hop.
- Ensure RLS session context is transaction-local, set by the server, and cannot be overwritten by request text. Restrict fields in Emma's basic DTO.
- Record Cedar deny tests (especially argument manipulation) and downstream API 403, row/field controls.
- For 3LO, store/rotate/revoke per-user vendor credentials securely; never log refresh/access tokens. Confirm consent and callback origin.
- Enforce write confirmation, idempotency, least-privilege AWS roles and replay-safe APIs.
- Verify logging correlation and that rejection responses don't disclose internal content.
- Track costs of Bedrock, RDS, vector storage, networking/PrivateLink, Cognito, API Gateway and CloudWatch.

## Credentials/identity reference

| Leg | Authentication | Token's real subject |
|---|---|---|
| Browser → API Gateway → Runtime U → G1 | Entra JWT for AI backend | Sarah/James/Emma/David/Oliver |
| G1 → Network/Ticket OpenAPI | Entra OBO, API-specific audience | The same human, delegated |
| G1 → vendor OpenAPI | Vendor OAuth 3LO | Authorized vendor user account |
| G1 → native Platform Health REST stage | AWS IAM/SigV4 | G1 Gateway execution role |
| Reporter Lambda → Runtime M → G2 | AWS IAM | Runtime M execution IAM role at G2 |
| G2 → Network aggregate OpenAPI | Entra OAuth client credentials 2LO | Reporter application, **no human** |
| Network/Ticket APIs → RDS | Database service login plus mapped context/RLS | Server-set validated user or app entitlements |

## Implementation evidence to capture

For each phase record a screenshot/configuration export (secrets redacted), relevant policy JSON, OpenAPI files, exact authorization rule, test ID, observed status/response and AWS resource IDs. Do not substitute a green frontend response for proof of downstream enforcement.

## Teardown

Delete scheduled triggers and credentials first, then runtime/gateway/targets, APIs, Lambda, KB/vector resources, private endpoints, RDS and test identity registrations. Verify all PoC resources and cost-bearing services are removed. Keep design docs and sanitized evidence.

## Docs

- [AWS Gateway docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AWS Gateway private egress](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html)
- [AWS AgentCore runtime policy](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
- [AWS native API Gateway target](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [Entra OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
