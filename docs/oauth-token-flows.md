# Identity Flow Notes — 2LO, 3LO and OBO

Brief notes **for this architecture**, not an OAuth course. [Technical flow](two-gateway-design.md).

| Flow | Our concrete example | Identity at protected system |
|---|---|---|
| **Corporate OIDC/3LO sign-in** | Emma, James, Sarah, David or Oliver signs in to web chat using **Entra** | Entry JWT represents the signed-in human; not automatically valid for downstream API |
| **OBO** | James invokes `get_diagnostics`; AgentCore Identity seeks a **new Entra token for Network API** | Delegated James identity/scope, VIC only, when verified and exchanged |
| **Vendor 3LO** | David explicitly authorises his **separate NetworkMonitor/Cognito** account | Vendor token/consent associated with David, not inherited by Sarah |
| **IAM + OAuth 2LO** | Scheduled reporter IAM Lambda → Runtime M/G2 → Health Summary OpenAPI target | Restricted application token; **no human user** |
| **IAM outbound service identity** | Oliver invokes `get_platform_health` via native API Gateway REST stage target | Gateway IAM execution identity; tool permissions enforced by trusted G1/Cedar policy |

**3LO is not OBO:** interactive login does not automatically exchange an audience-specific downstream token. **OAuth target support is not proof of OBO.** Wrong issuer/audience/expiry or insufficient scope must fail at resource boundaries. Never trust a prompt or caller-supplied user ID and never log raw bearer tokens.

**Where decisions are enforced:** Entry/API Gateway validates Entra JWT; G1/Cedar checks tool and region; Network/Ticket APIs validate OBO token and map authorised claims to PostgreSQL RLS; Cognito vendor API checks David's vendor grant; G2/IAM permits only read-only reporter; Health Summary API validates 2LO app scope.

See [users and permissions](identities-and-permissions.md), [tool inventory](tool-and-api-inventory.md), [ADR-001](architecture-decisions.md#adr-001--separate-human-jwt-and-automation-iam-gateways).
