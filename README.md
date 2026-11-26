# AWS Agentic AI Patterns

A practical AWS proof of concept exploring how AI agents securely discover and invoke downstream APIs using Amazon Bedrock AgentCore.

## Architecture

**Web app → Amazon API Gateway → AgentCore Runtime (Strands + Bedrock) → AgentCore Gateway → MCP tools → Network REST API**

The solution uses a fictional Network API with two operations: read service status and request a diagnostic.

Two AgentCore Gateways separate the service-identity and delegated-identity integration paths. Their selection and trade-offs are documented in [ADR-001](docs/architecture-decisions.md#adr-001--use-two-agentcore-gateways-for-separate-security-boundaries).

| | Gateway 1 | Gateway 2 |
|---|---|---|
| API integration | Native API Gateway REST stage target | OpenAPI specification target |
| Outbound authentication | IAM/SigV4 application identity | OAuth token exchange / OBO experiment |
| Identity at the API | Gateway/service principal | Delegated end-user identity (to be verified) |

Microsoft Entra ID supplies interactive (3LO) and machine-to-machine (2LO) identities. We test **User A: read + diagnostics**, **User B: read only**, and **machine client: read only**, with Cedar-based policy evaluation. The OAuth/OpenAPI path is intended to prove actual on-behalf-of (OBO) delegation; OAuth alone does not guarantee OBO.

## Architecture notes

- [Two-Gateway design](docs/two-gateway-design.md)
- [Identity and permissions](docs/identities-and-permissions.md)
- [2LO, 3LO and OBO notes](docs/oauth-token-flows.md)
- [API and tool inventory](docs/tool-and-api-inventory.md)

**Project status:** Design in progress. Features and identity propagation are not yet validated in a deployed environment.
