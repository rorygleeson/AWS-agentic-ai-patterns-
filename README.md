# AWS Agentic AI Patterns

A practical AWS proof of concept exploring how AI agents securely discover and invoke downstream APIs using Amazon Bedrock AgentCore.

## Architecture

**Web app → Amazon API Gateway → AgentCore Runtime (Strands + Bedrock) → AgentCore Gateway → MCP tools → Network REST API**

The solution uses a fictional Network API with two operations: read service status and request a diagnostic.

**Why two Gateways?** To **isolate two different trust and authorisation models**: (1) a Gateway invoking AWS APIs as a service identity and (2) a Gateway investigating delegated user access via OAuth/OBO. Separate Gateways give each model its own inbound authoriser configuration, execution role, policies, credentials and audit boundary. This makes the security differences testable without sharing the same Gateway role or policy surface. AWS supports different outbound credentials per target on a single Gateway, so **two are a deliberate security-isolation and learning choice, not an AWS requirement**.

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
