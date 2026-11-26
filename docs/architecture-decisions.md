# Architecture Decision Records

This document records **why** architectural choices were made, their alternatives and consequences. Technical implementation details belong in the related architecture documents.

---

## ADR-001 — Use two AgentCore Gateways for separate security boundaries

| Field | Value |
|---|---|
| Status | **Accepted** (design); implementation unverified |
| Decision date | 2026-11-26 |
| Decision owner | Rory Gleeson |
| Related task | S1-04 |
| Related design | [Two-Gateway Architecture](two-gateway-design.md) |

### Context

Our PoC compares two ways an AI agent reaches the **same logical Network API** (status read and simulated diagnostic):

- A native API Gateway REST stage target using an **IAM/SigV4 service identity**.
- An OpenAPI target using **OAuth**, with a planned **on-behalf-of (OBO)** token-exchange experiment to preserve delegated user identity.

AWS AgentCore Gateway allows **multiple targets with different outbound authentication methods on one Gateway**. Therefore, **two Gateways are not required merely because IAM and OAuth differ**.

### Decision

Deploy **two separately configured AgentCore Gateways**:

1. **Gateway 1 — Service identity:** Native API Gateway REST stage target, IAM/SigV4 outbound credentials, and trusted controls that enforce user/tool restrictions before any shared-service-identity call.
2. **Gateway 2 — Delegated identity:** OpenAPI target with OAuth outbound configuration, to evaluate and prove an actual user-delegated OBO token reaching the Network API.

Each Gateway has its own inbound authoriser configuration, execution role, credential/policy boundary and observability. Exact inbound settings and OBO compatibility will be validated during design and implementation.

### Why

- **Separate trust boundaries and reduced blast radius:** Service-identity privileges and delegated-identity experiments do not share one Gateway IAM role and policy boundary.
- **Independent security configurations:** We can test different inbound authorisation and policy controls without altering the other path.
- **Clear identity evidence:** It is easier to prove whether a downstream request represents a workload versus User A/User B.
- **Controlled comparison:** Both paths offer equivalent logical operations, so differences in authorisation behaviour are visible.

AWS recommends separate Gateways for targets with different trust boundaries/sensitivity. It also notes that the execution role is shared across targets using the Gateway IAM role.

### Alternatives considered

| Alternative | Assessment |
|---|---|
| **One Gateway with both targets** | Fully supported and simpler; may be preferable in production when trust boundaries are shared, but less isolation for this controlled PoC |
| **Two Gateways with the same target and auth** | Extra operational work without meaningful security/learning differentiation |
| **Only OpenAPI** | Omits AWS-native REST stage target behaviour |
| **Only API Gateway REST target** | Omits the OAuth/OBO experiment |

### Consequences and trade-offs

**Benefits:** Clear separation of security models, independently scoped roles/policies, easier negative testing and audit trails.

**Costs:** More resources, configuration, deployment effort, credential management, monitoring and potential charges.

**Risks to validate:** An OpenAPI OAuth target does **not** automatically establish end-to-end OBO; the actual Entra exchange and downstream token audience must be proven. Service identity alone cannot identify the original user; trusted upstream authorisation must prevent User B requesting diagnostics.

### Evidence required before claiming implementation complete

- [ ] Independently configure and trace both Gateways' inbound/outbound authorisation.
- [ ] Verify Gateway 1 uses application identity and prevents unauthorised diagnostic requests.
- [ ] Demonstrate whether Gateway 2 obtains a downstream token with validated delegated user context.
- [ ] Confirm User A can call both tools; User B and machine client remain read-only.
- [ ] Verify isolation and compare operational costs.

### References

- [AWS: Add Gateway targets — per-target credential providers](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-building-adding-targets.html)
- [AWS: Gateway outbound authorisation and role isolation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AWS: Gateway inbound authorisation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-inbound-auth.html)

---

## ADR register — other decisions awaiting formal write-up

The following decisions are recorded in the [master task board](../TODO.md) and related design notes, but **have not yet been expanded into full ADRs**:

- Prefer user-delegated OBO where feasible; compare with service identity.
- Two test users plus read-only machine client and explicit tool permissions.
- Compare native REST stage and OpenAPI target integrations.
- Use Cedar to evaluate explicit fine-grained policies.
- Keep the public README architecture-focused.

New ADRs will be added here as the design progresses. Do not mark them accepted without a recorded decision.
