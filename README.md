# AWS Agentic AI Patterns

A practical, security-focused AWS proof of concept demonstrating how agents connect to enterprise systems while preserving identity and enforcing authorisation.

## Architecture

**Interactive:** Web chat → Microsoft Entra ID → API Gateway → thin request adapter → AgentCore Runtime U (JWT) → AgentCore Gateway G1 (JWT/Cedar) → MCP tools → downstream services.

**Scheduled:** EventBridge Scheduler → IAM Lambda → AgentCore Runtime M (IAM) → AgentCore Gateway G2 (IAM) → read-only network health summary.

Two Runtimes and two Gateways separate **human JWT** and **machine IAM** inbound authentication. Each Gateway target independently configures its downstream credentials (OBO, 3LO, 2LO or IAM). See [ADR-001](docs/architecture-decisions.md#adr-001--separate-human-jwt-and-automation-iam-gateways) for the decision rationale.

The synthetic demonstration includes five named human roles (Sarah, James, Emma, David and Oliver), one Network Health Reporter service, a Knowledge Base, Network and Ticket APIs, an independent vendor API, a Platform Health API, and RDS PostgreSQL behind the internal APIs.

## Technical documentation

- [Architecture and Gateway mapping](docs/two-gateway-design.md)
- [Users and permissions](docs/identities-and-permissions.md)
- [Downstream systems and MCP tools](docs/tool-and-api-inventory.md)
- [2LO, 3LO and OBO notes](docs/oauth-token-flows.md)
- [Architecture decisions](docs/architecture-decisions.md)

**Status:** Architecture design under reconciliation. No infrastructure has been deployed or identity flows proven yet.
