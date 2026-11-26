# Identity Flow Notes — 2LO, 3LO and OBO

**Purpose:** Short reference notes for understanding our [PoC architecture](two-gateway-design.md). This is **not** a general OAuth tutorial. Actual token handling will be verified during implementation.

## Terms in our architecture

| Term | What it means **in this PoC** | Where we'll demonstrate it |
|---|---|---|
| **3LO (interactive)** | User A or B signs in with Entra ID using authorisation code + PKCE. The browser obtains an **access token** for the entry API. | Browser → API Gateway |
| **2LO (machine-to-machine)** | Our read-only automation client gets an **application access token** using client credentials. No human user is represented. | Machine Client → secured entry/API path |
| **OBO (on-behalf-of)** | A trusted tier exchanges a validated user access token for a **new downstream API access token** representing delegated user access. | Runtime/Gateway identity path → Network API (Gateway 2 experiment) |
| **Service identity** | Gateway or backend invokes the downstream API as an application instead of the original user. | Gateway 1 → Network API with IAM/SigV4 |

## Why OBO matters here

The browser's token is intended for the **entry API**. It should not simply be replayed to the Network API. In a genuine OBO flow, the downstream token has a **different audience** and carries appropriate delegated user context.

- **User A:** read status and request diagnostics.
- **User B:** read status only.
- **Machine client:** read status only.
- **Service-account variant:** the Network API sees the application identity. A trusted authorisation layer must enforce User A/User B restrictions before invoking it.

**Important distinction:** 3LO establishes interactive delegation; it does **not** automatically perform OBO. An OpenAPI target supporting OAuth also does **not** automatically prove OBO.

## Our two-Gateway comparison

| | Gateway 1 | Gateway 2 |
|---|---|---|
| Integration | Native API Gateway REST stage target | OpenAPI specification target |
| Outbound auth | IAM/SigV4 application identity | OAuth, with OBO exchange to be proven |
| What to inspect | Which application identity the downstream API trusts | Whether an actual user-delegated, audience-correct token arrives |
| Expected result | Works with enforced upstream user restrictions | Downstream API can make a delegated user permission decision **if OBO is supported/configured** |

## Token checks we'll demonstrate

- **Access token, not ID token**, authorises API calls.
- Validate **signature, issuer, audience, expiry and permissions** at the receiving API.
- **2LO token:** app-only identity, no human user.
- **OBO tokens:** distinct entry and downstream audiences; verify delegated claims rather than trusting a prompt or arbitrary header.
- No bearer tokens, client secrets or refresh tokens in logs or GitHub.

## What must be proven in the PoC

1. Capture redacted claims showing who each token represents and its intended audience.
2. Trace which component validates each token and performs any exchange.
3. Demonstrate User B's diagnostic call is denied in both Gateway patterns.
4. Confirm the exact Entra + AgentCore OBO mechanism; document any unsupported path honestly.

**Official implementation references:** [Microsoft Entra OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) · [AWS AgentCore OBO exchange](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html) · [AWS Gateway outbound authorisation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html).

See [Identity & Permissions](identities-and-permissions.md) and [Master TODO](../TODO.md).
