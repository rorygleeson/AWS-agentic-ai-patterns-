# AWS Agentic AI Patterns

A practical AWS proof of concept exploring how AI agents securely discover and invoke downstream APIs using Amazon Bedrock AgentCore.

## Architecture

**Web app → Amazon API Gateway → AgentCore Runtime (Strands + Bedrock) → AgentCore Gateway → MCP tools → Network REST API**

The solution uses a fictional Network API with two operations: read service status and request a diagnostic. Two AgentCore Gateways demonstrate different integration and identity patterns:

| | Gateway 1 | Gateway 2 |
|---|---|---|
| API integration | Native API Gateway REST stage target | OpenAPI specification target |
| Downstream access | IAM/SigV4 application identity | OAuth and delegated OBO experiment |

The design also explores Microsoft Entra ID sign-in, interactive and machine-to-machine access, user-specific tool permissions, and Cedar policy-based authorisation.

## Architecture notes

- [Two-Gateway design](docs/two-gateway-design.md)
- [Identity and permissions](docs/identities-and-permissions.md)
- [2LO, 3LO and OBO notes](docs/oauth-token-flows.md)
- [API and tool inventory](docs/tool-and-api-inventory.md)

**Project status:** Design in progress. Features and identity propagation are not yet validated in a deployed environment.
