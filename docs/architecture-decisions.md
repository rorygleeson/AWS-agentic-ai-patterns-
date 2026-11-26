# Architecture Decision Records

Only **approved decisions** appear here. Technical implementation detail belongs in the architecture documents. Keep this one consolidated ADR file.

---

## ADR-001 — Separate human JWT and automation IAM Gateways

| Field | Value |
|---|---|
| Status | Accepted design; not yet implemented |
| Decision date | 2026-11-26 |
| Owner | Rory Gleeson |
| Technical design | [Two-Runtime / Two-Gateway architecture](two-gateway-design.md) |

### Context

We need interactive corporate users **and** scheduled machine automation. These callers have different inbound trust and authentication requirements: Entra JWT for humans versus AWS IAM for workloads.

AgentCore Gateway has **one inbound authoriser configuration per Gateway**, while each target can independently select a supported outbound credential provider. A single Gateway can therefore mix target types and outbound IAM/OAuth methods. **REST versus OpenAPI is not the reason to separate Gateways.**

### Decision

Use two Runtimes and two Gateways:

- **Runtime U + Gateway G1:** JWT inbound, for human interactive flows. G1 uses `CUSTOM_JWT`, Cedar user/tool policies and multiple targets: internal Entra OBO Network/Ticket APIs; a shared KB; Cognito vendor 3LO; and a native API Gateway REST/IAM Platform Health target.
- **Runtime M + Gateway G2:** IAM inbound for scheduled Network Health Reporter. G2 uses `AWS_IAM` and exposes only the aggregate `get_network_health_summary` OpenAPI OAuth 2LO target.

This separates inbound identity domains, roles, policies, exposed tools and execution boundaries while still allowing *per-target* outbound authentication on G1.

### Rationale

- **Different inbound authorisers:** A human bearer-token gateway and a machine IAM gateway have different trust contracts; separate Gateways make each enforceable and testable without weakening the other.
- **Least privilege and isolation:** G2 exposes only one aggregate tool, not every human tool or Cedar rule. Separate execution roles and authorisers reduce blast radius.
- **Clear evidence:** We can demonstrate a delegated human API call versus an IAM-scheduled app-token summary without conflating caller identity with target credentials.
- **No needless backend duplication:** Both lanes may reach the same Network API implementation through separately scoped operations; RDS stays behind APIs.

### Alternatives considered

| Alternative | Consequence |
|---|---|
| One Gateway for both humans and automation | Simplifies footprint but forces one inbound authorisation configuration; more complex separation of JWT people and IAM workload trust |
| Two Gateways split by read versus write | Doesn't reflect actual identity/trust boundary; rejected |
| Two Gateways split by REST versus OpenAPI | Based on a false requirement; one Gateway can mix outbound target authentication; rejected |

### Trade-offs / consequences

Additional Runtime/Gateway resources, execution roles, policy configuration, logs and costs. Entra JWT propagation through the thin adapter, IAM ingress restrictions, OBO token exchange, downstream RLS and vendor 3LO require explicit validation. **No claim of implemented security is made until tests pass.**

### References

- [AWS Gateway inbound authorisation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-inbound-auth.html)
- [AWS Gateway per-target outbound authorisation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html)
- [AWS Gateway target configuration](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-building-adding-targets.html)
