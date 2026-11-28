# AgentCore components in context

[← Main README](../README.md)

## AgentCore components in context

AgentCore provides building blocks for deploying agents and connecting them to enterprise systems. They serve different purposes:

- **AgentCore Runtime:** Hosts and executes the AI agent. The agent decides which tool to request; Runtime does not replace the downstream system's permission checks.
- **AgentCore Memory:** Can store and retrieve short-term conversation context and longer-term information across sessions. It must be configured and used by the agent; hosting an agent in Runtime does not automatically give it persistent memory.
- **AWS Agent Registry:** A searchable catalogue in which teams can publish and discover agents, MCP servers, tools and related resources. A registry entry does not itself grant permission to invoke a tool.
- **AgentCore Gateway:** Presents approved APIs, functions and services as agent-accessible tools, and provides an authenticated entry point. Its configured inbound authentication and tool permissions are separate from the downstream API's controls.
- **MCP (Model Context Protocol):** The protocol through which an agent can discover tools and invoke them via Gateway. MCP standardises the tool interface; it does **not** replace OAuth, AWS IAM or business authorization.

**How they fit together here:** Human requests enter the **Enterprise AI Application**, then **Runtime** and **G1**. The **Entra Service Account** also invokes Runtime and G1 for automated AI analysis. The **AWS Service Account (using IAM Role)** calls a known tool on **G2** directly, without Runtime. Each final system applies its own permissions. **Memory and Registry are not required for these authentication flows.**

**AWS references:** [AgentCore overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html) · [Gateway and MCP](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using.html) · [AWS Agent Registry](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry.html).
