# Pattern 2 — Human identity and OAuth 2.0 OBO (Entra ID): implementation deep dive

[← Main README](../../README.md) · [All access patterns](../README.md)


**Design specification — human OBO token and Entra application mapping.** This pattern preserves a signed-in person's identity across **Enterprise AI Application → AgentCore Runtime → G1 → Network Operations API**. These names are **proposed Entra app-registration labels**, not deployed resource IDs.

| Entra registration | Role in this pattern |
|---|---|
| `enterprise-ai-application-spa` | **Public client** for the Enterprise AI Application's Authorization Code + PKCE sign-in. No confidential client secret in the browser. |
| `enterprise-agentcore-runtime` | **Protected Runtime audience** and **confidential OBO client** for obtaining a G1 token. Runtime's AgentCore Identity credential provider holds its confidential credential. |
| `enterprise-agentcore-g1` | **Protected G1 audience** and **confidential OBO client** for obtaining a Network Operations API token. G1's outbound AgentCore Identity provider uses its confidential credential. |
| `enterprise-network-api` | **Final protected API resource** exposing delegated scopes such as `status.read`, `diagnostics.read` and `tickets.create`. |

**Three access tokens, two OBO exchanges:**

| Request boundary | Token audience and identity |
|---|---|
| **Application → Runtime** | Entra **delegated access token** issued for `enterprise-agentcore-runtime`, representing James; obtained by the public client using Authorization Code + PKCE. Runtime validates issuer, signature, audience and authorized client. |
| **Runtime → G1 (OBO 1)** | A new Entra delegated token for `enterprise-agentcore-g1`, representing James. The confidential client registered as `enterprise-agentcore-runtime` redeems its own inbound-audience assertion through **AgentCore Identity OBO**. |
| **G1 → Network Operations API (OBO 2)** | A new Entra delegated token for `enterprise-network-api`, still representing James. The confidential client registered as `enterprise-agentcore-g1` redeems the G1-audience assertion through **Gateway outbound OBO**. The API validates the token and applies James's scopes, entitlements and VIC-only data rules. |

**Microsoft's key constraint:** For each OBO request, the incoming `assertion` must have an `aud` claim identifying the **same confidential client** making that exchange. Therefore, the Runtime and G1 resource registrations must also be configured as confidential OBO clients with separately managed credentials, appropriate delegated permissions and consent. **Do not reuse the Runtime token at G1 or the G1 token at the API.**

**Where credentials and permissions live:** The browser holds no client secret. AgentCore Identity manages the confidential credentials for the two OBO hops; Entra grants delegated permissions to the successive resources; Runtime and G1 check inbound tokens; Cedar checks G1 tool access; the Network Operations API independently enforces endpoint, region and field permissions. These registrations **preserve user delegation** and must not be confused with the separate Entra Service Account's **Client Credentials** route.

**Authoritative specifications:** [Microsoft Entra OBO and assertion audience](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) · [AWS AgentCore Identity OBO — Microsoft provider](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html) · [AWS Entra ID integration](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-idp-microsoft.html) · [AWS Gateway target authorization](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-building-adding-targets-authorization.html).

**Remaining implementation detail:** Exact app IDs, delegated scope strings, consent configuration, credential-provider names and Cedar policy JSON are deployment-specific and will be documented with the configuration examples. This does not introduce a custom Chat API or additional application service.
