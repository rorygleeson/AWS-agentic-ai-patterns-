# Architecture Decision Register (ADR)

This is the **single source of truth for architectural decisions** in the Enterprise GenAI on AWS proof of concept. The [README](../README.md) introduces the end systems and users; the [architecture baseline](architecture-baseline.md) describes the design; the [patterns guide](architecture-patterns.md) explains reusable mechanisms. **The ADRs live here only.**

**Status:** Accepted as a design baseline; implementation and tests have not yet occurred. These decisions are based on the design agreed in this chat.

## How to read this register

Each numbered ADR answers three practical questions:

1. **Decision:** What did we choose?
2. **Why and alternatives:** Why does it fit the PoC better than the alternative?
3. **Consequence:** What does it require or limit?

An ADR is **not another use case or tool**. For example, James diagnosing a VIC asset is a use case; OBO is a pattern; deciding to preserve his JWT through Lambda is an ADR.

## Decision index

| Area | ADRs |
|---|---|
| Identity and ingress | 001, 003, 004, 005, 006, 018 |
| Runtime, Gateway and tool authorization | 002, 007, 009, 017 |
| Downstream API, OAuth and database | 008, 010, 011, 012, 013, **022** |
| Shared RAG and safe writes | 014, 015 |
| Documentation, scope and delivery | 016, 019, 020, 021 |


## Identity and ingress boundaries

### ADR-001

- **Decision:** Separate human Runtime U (JWT) and automation Runtime M (IAM).
- **Why:** Enforces mutually exclusive inbound trust models.
- **Alternative considered:** one runtime with mixed identity handling, harder to verify.
- **Consequence:** two deployments.

### ADR-003

- **Decision:** Forward original Entra human JWT through Lambda and Runtime U, then use audience-correct OBO for the downstream Network API (including ticket operations).
- **Why:** Makes James's identity independently verifiable at downstream systems.
- **Alternative considered:** use Lambda IAM role as user identity, unacceptable for delegated API/RLS pattern.
- **Consequence:** token forwarding, audience, exchange, consent and expiry handling matter.

### ADR-004

- **Decision:** Retain a thin ingress Lambda.
- **Why:** Request shape validation, audit and session binding justify it; Lambda does not invent user identities.
- **Alternative considered:** direct Runtime JWT invocation reduces components but loses chosen ingress controls.
- **Consequence:** maintain minimal Lambda code.

### ADR-005

- **Decision:** PrivateLink/VPC boundary and Runtime resource policy for Lambda ingress.
- **Why:** Restricts unexpected bypass paths; follows documented private exposure pattern.
- **Alternative considered:** broadly internet-accessible runtime route.
- **Consequence:** private endpoint networking and policy configuration.

### ADR-006

- **Decision:** Entra user application roles and/or trusted group-to-entitlement mapping, not groups-as-scopes.
- **Why:** App roles/groups/scopes differ; claims must be interpreted explicitly. Demo user roles can be assigned directly without requiring group-assignment licensing.
- **Consequence:** maintain entitlement mapping, fail closed on group overage.

### ADR-018

- **Decision:** Each OAuth API has its own audience; 2LO `roles` and OBO `scp` must be distinguished.
- **Why:** Prevents token substitution/confused deputy.
- **Alternative considered:** one bearer token trusted everywhere.
- **Consequence:** separate Entra API registrations/consents and strict audience checks.


## AgentCore Runtime, Gateway and tool authorization

### ADR-002

- **Decision:** Two AgentCore Gateways: G1 `CUSTOM_JWT`, G2 `AWS_IAM`.
- **Why:** A Gateway has one inbound `authorizerType`; cannot assume both modes on one gateway.
- **Alternative considered:** shared IAM gateway drops caller-level JWT security.
- **Consequence:** two gateway endpoints/policy bundles.

### ADR-007

- **Decision:** Enforce human tool authorization in G1 Cedar AND downstream API.
- **Why:** Defence in depth and avoids trusting tool arguments.
- **Alternative considered:** Cedar only; API bypass vulnerability.
- **Consequence:** two consistent but non-identical control layers.

### ADR-009

- **Decision:** Automation uses explicit Reporter Lambda role → Runtime M role → G2 IAM.
- **Why:** Clarifies the principal each service sees.
- **Alternative considered:** pass a fabricated employee token.
- **Consequence:** tightly scoped AWS roles/resource policies.

### ADR-017

- **Decision:** Do not duplicate human MCP tools/Cedar policies into G2.
- **Why:** Gateways split by principal type, not by blanket tool replication.
- **Alternative considered:** shared catalog everywhere risks extra privileges and policy drift.
- **Consequence:** narrow G2 inventory.


## Downstream API, OAuth and database patterns

### ADR-008

- **Decision:** Apply PostgreSQL RLS plus response field filtering at the Network API.
- **Why:** Scope rows by validated user; RLS alone cannot hide sensitive fields.
- **Alternative considered:** filter in agent prompt, insecure.
- **Consequence:** transaction-local principal context, no BYPASSRLS and careful pool cleanup.

### ADR-010

- **Decision:** M2M uses Entra client credentials (2LO) and app-only `HealthSummary.Read`.
- **Why:** No end user is involved.
- **Alternative considered:** OBO, invalid without real user assertion.
- **Consequence:** isolated M2M app registration and secret lifecycle.

### ADR-011

- **Decision:** Vendor alerts use separate OAuth 3LO with Cognito as independent fictional vendor IdP.
- **Why:** Demonstrates user-specific third-party consent distinct from corporate Entra OBO.
- **Alternative considered:** another Entra API duplicates existing pattern.
- **Consequence:** callback/consent/token storage/revocation tests.

### ADR-012

- **Decision:** Oliver's `get_platform_health` is a native AWS API Gateway REST API stage target, outbound IAM/SigV4.
- **Why:** Demonstrates native AWS resource discovery and service-execution identity rather than OAuth OBO.
- **Alternative considered:** OpenAPI target duplicates P07.
- **Consequence:** backend sees Gateway IAM role, not Oliver. Cedar must still authorize Oliver first.

### ADR-013

- **Decision:** OAuth-protected REST targets use OpenAPI registration.
- **Why:** Native API Gateway stage target supports IAM/API-key outbound, not OAuth.
- **Alternative considered:** native target with unsupported OAuth.
- **Consequence:** maintain schema separately from running REST API.


## Shared knowledge and safe agent behaviour

### ADR-014

- **Decision:** Shared non-sensitive RAG for every human, without document-level user ACL.
- **Why:** User requested simple shared retrieval and authorization examples elsewhere.
- **Alternative considered:** per-user KB filtering adds complexity without proving new core pattern.
- **Consequence:** corpus must contain no restricted records.

### ADR-015

- **Decision:** Ticket creation requires confirmed intent, downstream entitlement and idempotency.
- **Why:** Agent output must not silently create duplicate/unauthorized records.
- **Alternative considered:** free-form auto-create, unacceptable.
- **Consequence:** explicit UX confirmation and API support.


## Documentation, scope and delivery

### ADR-016

- **Decision:** Architecture, patterns, decisions, console implementation and tests live in separate docs.
- **Why:** Reusable article/reference vs hands-on guide vs results must remain distinct.
- **Alternative considered:** a sprawling single file.
- **Consequence:** cross-links and traceability maintenance.

### ADR-019

- **Decision:** Use synthetic domain services for the PoC, generic AWS enterprise framing.
- **Why:** Clear capabilities without actual corporate/network data or named internal products.
- **Alternative considered:** claim integration with a real organization's services.
- **Consequence:** all data/test identities are fictional.

### ADR-020

- **Decision:** Console-first build; no IaC in this tutorial.
- **Why:** Supports learning user preference.
- **Alternative considered:** CDK/Terraform, out of scope for setup.
- **Consequence:** capture reproducible settings/screenshots and clean-up costs.

### ADR-021

- **Decision:** Keep the repo documentation-only until implemented.
- **Why:** Design can be researched before testing, while tests later confirm real behavior.
- **Alternative considered:** mislabel unverified flows as production-ready.
- **Consequence:** clearly show assumptions and evidence gates.

### ADR-022

- **Decision:** Merge the previously separate Ticket Management API into the **Network Operations API**. The Network API hosts asset status, diagnostics, ticket read/create and aggregate-health endpoints, all backed by the same RDS database.
- **Why:** The separate Ticket API did not demonstrate a new authentication pattern: both enterprise human paths used Entra OBO and user-specific authorization. A single API makes endpoint-, region-, read/write- and app-only permission differences easier to teach.
- **Alternative considered:** Keep separate Network and Ticket APIs with separate Entra audiences. This adds deployment and token-registration complexity without supporting a distinct core use case.
- **Consequence:** One Entra Network API resource/audience with delegated `asset.read`, `diagnostics.read`, `ticket.read`, `ticket.create`, plus the app-only `HealthSummary.Read` permission. Each operation remains a separate MCP tool. The native AWS API Gateway REST-stage vs OpenAPI registration comparison **still exists** through Platform Health vs Network Operations.
- **Supersedes:** The separate-Ticket-API assumption in earlier drafts; all other identity and authorization decisions remain in force.

## Design invariants

These statements connect the decisions without duplicating their rationale:

- A user's identity comes from a **validated token**, never the prompt, HTTP JSON `userId`, or Lambda IAM role.
- G1 accepts **human JWT**, G2 accepts **machine IAM**; each Gateway has one inbound authorizer. A target's outbound credentials are configured separately.
- The **OpenAPI + OBO** REST path preserves James's delegated identity; the **native API Gateway REST stage + IAM** path deliberately uses the Gateway execution role after Cedar authorizes Oliver.
- OAuth **2LO** is app-only; OAuth **3LO** is independent vendor-user consent; **OBO** preserves a corporate user's delegated identity. Do not collapse them.
- Downstream APIs re-authorize and RDS applies RLS; Cedar alone is insufficient.
- The RAG corpus is shared and non-sensitive; region/field restrictions are demonstrated elsewhere.
- Tests must include denial and bypass scenarios, not just successful agent replies.

## How to change a decision

When vendor documentation or implementation evidence makes a choice obsolete, add a new **superseding ADR here** and cross-reference the old ID. Record the date, observed restriction, chosen alternative, effects on user permissions and required test updates. Do not quietly edit history or scatter new decision lists across other docs.

## Supporting references

- [AWS AgentCore CreateGateway API — inbound authorizer](https://docs.aws.amazon.com/bedrock-agentcore-control/latest/APIReference/API_CreateGateway.html)
- [AWS Gateway outbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AWS native API Gateway REST targets](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html)
- [AWS AgentCore resource policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
- [AWS API Gateway → Lambda → AgentCore sample](https://github.com/aws-samples/sample-expose-agentcore-via-api-gateway)
- [Microsoft Entra OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [PostgreSQL row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

## ADR — Reduce duplicate Network API read endpoints while retaining authorized writes

- **Decision:** Remove `GET /tickets/{id}` from the PoC. Keep four Network Operations API endpoints: `GET /assets/{id}/status`, `GET /assets/{id}/diagnostics`, `POST /tickets`, and `GET /network/health-summary`.
- **Why:** Ticket read duplicates the human OBO + region-restricted read path already demonstrated by diagnostics. Ticket **creation** remains because it demonstrates a materially different **state-changing action**, explicit human confirmation, idempotency and write-specific authorization.
- **Impact:** The Network API still tests different tools and result filtering, regional RLS, human OBO, app-only OAuth, and authorized writes. One Network API target exposes four tools. The delegated scope for ticket creation is `tickets.write`; no ticket-read scope is needed.
- **Supersedes in part:** The five-endpoint inventory in the earlier API consolidation ADR. That ADR's decision to keep **one API and one Entra API audience** remains unchanged.

## ADR — Pin Entra OBO client identity and separate Gateway target credentials

- **Decision:** Token A is issued for the `enterprise-ai-backend` Entra app registration. AgentCore's confidential Entra OBO credential provider uses **that same app's client ID** to request Token B for the separate `enterprise-network-api` audience.
- **Why:** Microsoft OBO rejects a token whose audience belongs to a different application from the confidential client performing the exchange.
- **Decision:** Keep one Network Operations backend REST API, but create **two separate OpenAPI Gateway targets**: G1 human OBO (status, diagnostics, ticket creation) and G2 Reporter Client Credentials (health-summary only).
- **Why:** The Gateway target's outbound credential configuration differs, while the underlying API is shared.
- **Decision:** Use the native managed Knowledge Bases Gateway connector's actual `Retrieve` MCP tool (with its Gateway target prefix). “Search knowledge” is only the friendly use-case description.
- **Consequence:** No additional end systems, authentication patterns or business endpoints.
