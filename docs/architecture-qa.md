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


## Alternative — VPC PrivateLink and a Gateway with no inbound authentication

**Q: Does AgentCore Gateway have to use OAuth/JWT or IAM authentication?**  
**A: No.** AgentCore Gateway supports inbound authorizer type `NONE`, which performs **no authentication or authorization at the Gateway boundary**. This is supported by AWS, but **not selected for our enterprise PoC**: our design retains **G1 = Entra JWT** and **G2 = IAM/SigV4**, with Cedar tool authorization where appropriate.

**Q: Could a `NONE` Gateway still be limited to approved VPCs?**  
**A: Yes.** AWS PrivateLink provides an **interface VPC endpoint** for AgentCore Gateway. An **AWS Organizations Resource Control Policy (RCP)** can deny `bedrock-agentcore:InvokeGateway` when `aws:SourceVpc` or `aws:SourceVpce` is not approved. AWS explicitly says these RCPs apply to **all Gateway inbound types, including `NONE`**. The RCP is attached at the organization root, OU or account—not directly to the Gateway—and **sets a deny perimeter; it never grants access**. A resource-based policy can also be attached to an individual supported AgentCore resource. A VPC endpoint policy and endpoint security groups add further restrictions.

**Q: Does an approved VPC make an unauthenticated Gateway secure for sensitive tools?**  
**A: Not by itself.** A VPC restriction answers **“Where did this request come from?”**, not **“Who is calling?”** With `NONE`, a workload that can reach Gateway through an approved VPC endpoint may invoke its tools unless an additional trusted control blocks it. Sensitive tools therefore need suitable downstream authorization and/or an authentication/interceptor layer.

**Important implementation consideration:** A strict VPC-only RCP will also block an AgentCore Runtime agent that invokes Gateway **outside the approved VPC path**. Runtime must be configured to reach Gateway through an approved interface VPC endpoint, or the perimeter design must explicitly account for that route. Test AWS-service/forward-access-session exceptions and CloudTrail data-event visibility before enforcing an RCP broadly.

**Decision for this PoC:** Recognise **PrivateLink + RCP network restriction** as an optional additional enterprise control. Recognise **`NONE` + restricted network access** as a technically supported alternative, **not** a sixth access pattern and **not** a replacement for G1/G2 authentication in our chosen design.

**AWS references:**
- [Gateway inbound authorization, including `NONE` and its security considerations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-inbound-auth.html)
- [Resource control policies for AgentCore — VPC restriction and `NONE` applicability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-control-policies.html)
- [AgentCore PrivateLink interface VPC endpoints and endpoint policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/vpc-interface-endpoints.html)
- [AgentCore resource-based policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
