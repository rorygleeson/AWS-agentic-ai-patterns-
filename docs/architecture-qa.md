# Architecture Q&A

[← Main README](../README.md)

## Why two OBO exchanges?

**First, is the MCP tool the same as the API?** The tool (for example, `get_diagnostics`) is an operation that **G1 exposes over MCP**. G1's **OpenAPI target configuration** maps that tool to the existing REST endpoint (`GET /assets/{id}/diagnostics`). The target is configuration, **not another backend API**. G1 authenticates and authorizes the MCP tool invocation; the Network Operations API separately authenticates and authorizes the resulting REST request.

**Q: Why are there two OAuth 2.0 On-Behalf-Of (OBO) exchanges?**  
**A:** Our design crosses three independently protected resources: **AgentCore Runtime → G1 → Network Operations API**. James first has a token intended for Runtime. AgentCore Identity obtains a **G1-audience** token for the agent's tool call; then G1 obtains a **Network Operations API-audience** token for the final API call. G1 checks tool authorization using Cedar, and the API independently enforces James's regional and endpoint permissions. James still **signs in only once**.

**Q: Does AWS or Microsoft require exactly two OBO exchanges?**  
**A: No.** Two exchanges are **our architectural choice** for these three trust boundaries. [Microsoft's Entra OBO specification](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) requires the incoming token to be issued for the confidential client performing that particular exchange; it does not prescribe how many intermediary services an architecture must have. [AWS AgentCore Identity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html) supports audience-scoped OBO exchange and has a Microsoft Entra provider mode. The two Entra confidential-client registrations and their permitted audiences must be designed consistently with those requirements.

**Q: Why not use just one OBO exchange?**  
**A:** The decisive constraint is **G1's outbound authorization to our Network Operations API**, which is exposed as an **OpenAPI/MCP tool target**. AWS documents these options:

| G1 outbound option | Support and consequence for this PoC |
|---|---|
| **OAuth 2.0 OBO** | **Supported.** API receives a token representing James. **Chosen.** |
| **JWT passthrough** | **Not supported** for our OpenAPI target; also reuses G1's token audience. |
| **OAuth 2.0 Client Credentials** | **Supported**, but the API receives an application identity, not James. |
| **AWS IAM/SigV4** | **Supported**, but our Network Operations API expects Entra OAuth. |

Source: [AWS AgentCore Gateway — outbound authorization support by target type](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html).

If we use **only Runtime → G1 OBO**, G1 would need to forward its token to the API rather than exchange it; that shortcut is not supported by our target type and would blur resource audiences. If we use **only Runtime → API OBO**, we must change or bypass the separately authenticated **G1/Cedar** boundary. **One OBO is possible in other architectures, but it does not preserve all the requirements of this one.**

**Q: Couldn't Cedar replace the second OBO exchange?**  
**A: Cedar can authorize a tool call, but it cannot replace a delegated OAuth token.** A simpler alternative would let G1 verify James with Cedar and call the Network Operations API using **application-only credentials**. That API would then authenticate the application, **not James**. It could no longer enforce James's VIC-only entitlement from a verified delegated identity unless we built another trusted user-context propagation and authorization mechanism. That is a valid architecture for some use cases—similar to our Platform Health pattern—but **not our requirement for user-specific Network Operations API and PostgreSQL row-level restrictions**.

**Q: What tradeoff are we accepting?**  
**A:** We acknowledge that **one exchange or application-only access could be simpler**. We intentionally accept the extra Entra client and token-exchange configuration so **Cedar can restrict tool use at G1 while the final API still knows which human is calling and independently enforces that person's data permissions**. We are not adding a second OBO just because an MCP tool has its own name.

**Q: Is the extra complexity justified by AWS and Microsoft guidance?**  
**A:** **For our chosen trust boundaries, yes—although neither vendor prescribes exactly two OBO exchanges.** We trade additional token-exchange configuration for distinct resource audiences and independent authorization at G1 and the API. [AWS recommends OBO instead of token passthrough for production Gateway integrations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-inbound-auth.html). [Microsoft Entra requires each OBO assertion to be intended for the confidential client redeeming it](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), and [AgentCore Identity documents Microsoft-specific OBO support](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html). Our design follows those documented principles; the precise Entra client registrations, delegated permissions and audience mappings will be specified in the technical appendix.

**Q: Do the service accounts also use OBO?**  
**A: No.** OBO represents a signed-in **human**. Our **Entra Service Account** uses OAuth 2.0 Client Credentials for application-only tokens, while the **AWS Service Account (using IAM Role)** invokes G2 using IAM/SigV4 and obtains a separate application token for the Network Operations API. Their configuration details belong in the [authentication appendix](#appendix-enterprise-ai-application-authentication).

*The three invocation paths are shown in the mobile-readable architecture below. For the seven named callers and their G1/G2 connections, see the [caller-to-Gateway overview](docs/diagrams/aws-agentcore-overview-mobile-v16.svg).*
