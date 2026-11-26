# Identities and Authorisation Matrix

**Status:** Restored agreed design baseline. Five named humans **plus** one automation principal; implementation claims are unverified.

| Identity | Auth at entry | Approved business scope | Explicit denial |
|---|---|---|---|
| **Sarah — National Admin** | Entra user JWT → Runtime U / G1 JWT | National assets, diagnostics, ticket read/create with confirmation, shared KB | Cannot use David's separately authorised vendor account |
| **James — VIC Engineer** | Entra user JWT → Runtime U / G1 JWT | VIC asset/diagnostic/ticket actions, shared KB | NSW rows and ticket mutations without confirmation |
| **Emma — Service Desk** | Entra user JWT → Runtime U / G1 JWT | Nationwide **basic** asset status, shared approved procedures | Diagnostic details, ticket actions |
| **David — External Analyst** | Entra user JWT; independent Cognito 3LO for vendor | Vendor alerts linked to his own NetworkMonitor account | Privileged internal Network/Ticket tools and other users' vendor tokens |
| **Oliver — Platform Operations Analyst** | Entra user JWT → Runtime U / G1 JWT | `get_platform_health` native target only | Network/Ticket, vendor and broader privileged functions |
| **Network Health Reporter** | IAM Lambda → Runtime M / G2 IAM; Entra OAuth 2LO outbound | `get_network_health_summary` aggregate only | Any personal delegation, diagnostics, tickets, or write tools |

**Not one-user-per-pattern:** Roles drive business permissions; several users may traverse the same technical integration. The reporter is a service identity, not a sixth human user.

## Enforcement boundaries

1. **Entry:** Validate JWT signature, issuer, audience, expiry, permitted scope/role. Entra sign-in is not enough to authorise tools.
2. **Runtime U/G1:** Preserve validated human identity; G1 inbound `CUSTOM_JWT`; Cedar uses trusted user roles/region/tool/parameters and denies by default.
3. **Runtime M/G2:** AWS IAM inbound for the scheduled workload; a narrow policy allows only the health-summary tool. G2's IAM principal is the Runtime M workload, not implicitly the Scheduler Lambda.
4. **Internal Network/Ticket APIs:** Validate **new audience-specific Entra OBO tokens** where supported. Map verified claims to permitted operations/regions.
5. **Database:** Network/Ticket APIs alone connect to PostgreSQL and set transaction-scoped RLS context. The DB does not directly consume Entra user tokens.
6. **Vendor:** Independent Cognito authorisation-code 3LO; vendor validates vendor token and own entitlements. No token sharing between David and Sarah.
7. **Platform Health:** Native API Gateway REST stage target uses AWS IAM/SigV4 outbound, with Oliver's tool restriction enforced before service-identity call.
8. **Automation:** Restricted OAuth 2LO client credentials for a Network health-summary API audience and aggregate-only data; no end-user identity.

## Named tools

**G1 (human JWT):** `search_knowledge`, `get_asset_status`, `get_diagnostics`, `get_ticket`, `create_ticket`, `search_vendor_alerts`, `get_platform_health`.

**G2 (IAM automation):** `get_network_health_summary` only.

## Essential negative tests

- Emma directly invokes `get_diagnostics` → Cedar deny; downstream protected API also denies.
- James requests NSW asset/ticket rows or says “I am national admin” → token claims and RLS remain VIC-only.
- James/Sarah trigger ticket creation without explicit confirmation → **no mutation**; duplicate confirmed submission produces only one ticket.
- Sarah attempts David's vendor alerts without own Cognito authorisation → deny.
- Oliver attempts any tool beyond platform health → deny.
- Reporter invokes any human tool or write operation → deny.
- Wrong audience/issuer/expired token at each resource boundary → deny.
- No relying on prompt text or client-supplied `userId` as identity.

Open implementation verification: Entra app registrations, group/claim mapping, exact audiences/scopes, policy inputs, RLS context and target-credential compatibility. See [scope](scope.md).
