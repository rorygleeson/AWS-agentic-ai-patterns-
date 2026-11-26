# Architecture Decision Register (ADR)

**Status:** Accepted design baseline, pending implementation evidence. This is a *single* decision register, separate from the reference architecture and console tutorial. Decisions reflect only this conversation. If AWS capabilities evolve, record a new superseding ADR rather than silently rewriting rationale.

| ID | Decision | Rationale / alternatives / consequence |
|---|---|---|
| ADR-001 | **Separate human Runtime U (JWT) and automation Runtime M (IAM).** | Enforces mutually exclusive inbound trust models. Alternative: one runtime with mixed identity handling, harder to verify. Consequence: two deployments. |
| ADR-002 | **Two AgentCore Gateways: G1 `CUSTOM_JWT`, G2 `AWS_IAM`.** | A Gateway has one inbound `authorizerType`; cannot assume both modes on one gateway. Alternative: shared IAM gateway drops caller-level JWT security. Consequence: two gateway endpoints/policy bundles. |
| ADR-003 | **Forward original Entra human JWT through Lambda and Runtime U, then use audience-correct OBO for downstream Network/Ticket APIs.** | Makes James's identity independently verifiable at downstream systems. Alternative: use Lambda IAM role as user identity, unacceptable for delegated API/RLS pattern. Consequence: token forwarding, audience, exchange, consent and expiry handling matter. |
| ADR-004 | **Retain a thin ingress Lambda.** | Request shape validation, audit and session binding justify it; Lambda does not invent user identities. Alternative: direct Runtime JWT invocation reduces components but loses chosen ingress controls. Consequence: maintain minimal Lambda code. |
| ADR-005 | **PrivateLink/VPC boundary and Runtime resource policy for Lambda ingress.** | Restricts unexpected bypass paths; follows documented private exposure pattern. Alternative: broadly internet-accessible runtime route. Consequence: private endpoint networking and policy configuration. |
| ADR-006 | **Entra user application roles and/or trusted group-to-entitlement mapping, not groups-as-scopes.** | App roles/groups/scopes differ; claims must be interpreted explicitly. Demo user roles can be assigned directly without requiring group-assignment licensing. Consequence: maintain entitlement mapping, fail closed on group overage. |
| ADR-007 | **Enforce human tool authorization in G1 Cedar AND downstream API.** | Defence in depth and avoids trusting tool arguments. Alternative: Cedar only; API bypass vulnerability. Consequence: two consistent but non-identical control layers. |
| ADR-008 | **Apply PostgreSQL RLS plus response field filtering at the Network/Ticket APIs.** | Scope rows by validated user; RLS alone cannot hide sensitive fields. Alternative: filter in agent prompt, insecure. Consequence: transaction-local principal context, no BYPASSRLS and careful pool cleanup. |
| ADR-009 | **Automation uses explicit Reporter Lambda role → Runtime M role → G2 IAM.** | Clarifies the principal each service sees. Alternative: pass a fabricated employee token. Consequence: tightly scoped AWS roles/resource policies. |
| ADR-010 | **M2M uses Entra client credentials (2LO) and app-only `HealthSummary.Read`.** | No end user is involved. Alternative: OBO, invalid without real user assertion. Consequence: isolated M2M app registration and secret lifecycle. |
| ADR-011 | **Vendor alerts use separate OAuth 3LO with Cognito as independent fictional vendor IdP.** | Demonstrates user-specific third-party consent distinct from corporate Entra OBO. Alternative: another Entra API duplicates existing pattern. Consequence: callback/consent/token storage/revocation tests. |
| ADR-012 | **Oliver's `get_platform_health` is a native AWS API Gateway REST API stage target, outbound IAM/SigV4.** | Demonstrates native AWS resource discovery and service-execution identity rather than OAuth OBO. Alternative: OpenAPI target duplicates P07. Consequence: backend sees Gateway IAM role, not Oliver. Cedar must still authorize Oliver first. |
| ADR-013 | **OAuth-protected REST targets use OpenAPI registration.** | Native API Gateway stage target supports IAM/API-key outbound, not OAuth. Alternative: native target with unsupported OAuth. Consequence: maintain schema separately from running REST API. |
| ADR-014 | **Shared non-sensitive RAG for every human, without document-level user ACL.** | User requested simple shared retrieval and authorization examples elsewhere. Alternative: per-user KB filtering adds complexity without proving new core pattern. Consequence: corpus must contain no restricted records. |
| ADR-015 | **Ticket creation requires confirmed intent, downstream entitlement and idempotency.** | Agent output must not silently create duplicate/unauthorized records. Alternative: free-form auto-create, unacceptable. Consequence: explicit UX confirmation and API support. |
| ADR-016 | **Architecture, patterns, decisions, console implementation and tests live in separate docs.** | Reusable article/reference vs hands-on guide vs results must remain distinct. Alternative: a sprawling single file. Consequence: cross-links and traceability maintenance. |
| ADR-017 | **Do not duplicate human MCP tools/Cedar policies into G2.** | Gateways split by principal type, not by blanket tool replication. Alternative: shared catalog everywhere risks extra privileges and policy drift. Consequence: narrow G2 inventory. |
| ADR-018 | **Each OAuth API has its own audience; 2LO `roles` and OBO `scp` must be distinguished.** | Prevents token substitution/confused deputy. Alternative: one bearer token trusted everywhere. Consequence: separate Entra API registrations/consents and strict audience checks. |
| ADR-019 | **Use synthetic domain services for the PoC, generic AWS enterprise framing.** | Clear capabilities without actual corporate/network data or named internal products. Alternative: claim integration with a real organization's services. Consequence: all data/test identities are fictional. |
| ADR-020 | **Console-first build; no IaC in this tutorial.** | Supports learning user preference. Alternative: CDK/Terraform, out of scope for setup. Consequence: capture reproducible settings/screenshots and clean-up costs. |
| ADR-021 | **Keep the repo documentation-only until implemented.** | Design can be researched before testing, while tests later confirm real behavior. Alternative: mislabel unverified flows as production-ready. Consequence: clearly show assumptions and evidence gates. |

## Non-negotiable invariants

1. No end-user identity comes from chat text, tool arguments or a Lambda-assumed role.
2. Human and machine Runtime/Gateway inbound authorization are separate.
3. Gateway target outbound authentication is configured **per target**, independently of inbound type.
4. Downstream resources re-authorize requests and do not trust Cedar as sole enforcement.
5. RAG is shared; fine-grained access demonstrations use the downstream APIs and third-party vendor.
6. 3LO, 2LO and OBO are separate OAuth patterns with distinct actors and tokens.
7. Native REST-stage and OpenAPI MCP targets remain both in scope.
8. Every denied tool/API/row/field path has a negative test.

## Change control

When implementation contradicts a decision:
- Record the observed AWS/API capability and date.
- Keep the original entry for history; add an ADR amendment or superseding entry.
- Update architecture/patterns/tests consistently.
- Never bypass a security boundary merely to make a demo pass.

## Primary docs for decision verification

- [AWS CreateGateway authorizer](https://docs.aws.amazon.com/bedrock-agentcore-control/latest/APIReference/API_CreateGateway.html)
- [AWS AgentCore outbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AWS Gateway native API Gateway targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [AWS AgentCore resource-based policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
- [AWS sample: expose AgentCore through API Gateway](https://github.com/aws-samples/sample-expose-agentcore-via-api-gateway)
- [Entra OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [PostgreSQL RLS](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
