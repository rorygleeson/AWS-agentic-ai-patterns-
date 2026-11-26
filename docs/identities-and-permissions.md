# Test Identities and Permission Matrix

**Stage:** 1 / S1-02  
**Status:** Proposed baseline — awaiting confirmation of the machine-client permission set and precise Entra implementation.  
**Environment:** Test-only identities in a dedicated Microsoft Entra ID tenant. No personal accounts, real customer data, or production privileges.

## Principals

| ID | Principal | Login/token flow | Purpose |
|---|---|---|---|
| U-A | **User A — Operator** | Interactive OIDC sign-in; OAuth authorisation-code + PKCE (3LO context) | Full two-operation demonstration |
| U-B | **User B — Viewer** | Interactive OIDC sign-in; OAuth authorisation-code + PKCE (3LO context) | Read-only demonstration and denied diagnostic |
| M-1 | **Machine Client — Automation** | OAuth client credentials (2LO) | Independent non-interactive client; no human subject |
| S-1 | **Downstream Service Identity** | Application credential where a target requires a shared service principal | Compare app-only API access to delegated OBO |

**M-1 and S-1 represent distinct logical roles**, even if the implementation later uses one app registration for a safe, clearly documented reason. We should prefer separate registrations/credentials for clarity and least privilege. User A/B are test-only users, not real people.

## Candidate API operations and MCP tools

The fictional Network API has two logically separate operations:

| Tool (candidate name) | HTTP operation | Permission |
|---|---|---|
| `get_service_status` | `GET /network/services/{id}` | `network.status.read` |
| `request_diagnostic` | `POST /network/services/{id}/diagnostics` | `network.diagnostics.request` |

Tool names, URL paths and permission strings are *design labels* until validated against actual Gateway and Entra configuration. Tool/operation correspondence must be confirmed independently for REST API and OpenAPI targets.

## Authorisation matrix

| Principal | Read service status | Request diagnostic | Notes |
|---|---|---|---|
| U-A — Operator | **Allow** | **Allow** | Delegated user authorisation |
| U-B — Viewer | **Allow** | **Deny** | Denial enforced by trusted layer and downstream resource authority |
| M-1 — Automation | **Allow (proposal)** | **Deny (proposal)** | App-only; exact permissions to confirm |
| S-1 — Service Identity | **Requires separate policy** | **Requires separate policy** | Its downstream privileges are **not** a substitute for checking the initiating user |

We have already agreed User A gets both and User B gets read only. **M-1's access is proposed, not yet approved.**

## Trust and identity propagation

1. **Browser to API Gateway:** The user's token proves the caller's identity and relevant authorisation. Gateway validates appropriate issuer, audience, signature, expiry and claims.
2. **API Gateway to AgentCore Runtime:** Invocation is secured. The design must define exactly which verified identity claims and/or tokens cross this boundary; forwarding browser bearer tokens blindly is not acceptable.
3. **Runtime to AgentCore Gateway:** Gateway authenticates its calling agent and authorises tool access according to the configured controls.
4. **Gateway to API — OBO:** Where supported, exchange a delegated token for the correct API audience; the API validates the new token and applies the user's permissions.
5. **Gateway to API — service identity:** The API sees an application principal, **not User A or B**. The initiating user's tool permission must already be enforced in a trusted layer, and downstream app rights must be narrowly scoped.
6. **M2M:** Client credentials token represents the application; do not represent it as a 3LO user token or invent an end-user identity.

**Critical rule:** A caller-controlled `userId`, prompt or HTTP header is not proof of identity. User-based policy inputs must come from validated claims or trusted identity context. Tool discovery alone is not an authorisation boundary.

## Negative tests to prove

- U-B requests a diagnostic through agent → denied.
- U-B attempts direct API diagnostic invocation → denied when delegated; service-identity variant needs trusted pre-execution controls and direct-API isolation.
- U-A invokes both tools → allowed.
- M-1 gets an app token and attempts an unauthorised operation → denied.
- User token presented to API with incorrect audience → denied.
- Expired/forged token → denied.
- Service identity token cannot be used by browser callers.
- Logs identify caller, token audience/flow, selected tool and outcome without exposing raw tokens.

## Open confirmations

- Confirm whether M-1 should be **read-only** (recommended for a clean comparison) or authorised for both.
- Confirm the Entra tenant, exact role/scope issuance method, and whether gateway policy maps roles or scopes to MCP tool names.
- Confirm downstream OBO feasibility and token-exchange design against current service documentation.
- Confirm whether the service principal needs diagnostic permission in the service-account comparison; if so, enforce the initiating user's restrictions in a trusted layer.

See [Scope](scope.md) and [Master TODO](../TODO.md). No identities or infrastructure have been provisioned yet.
