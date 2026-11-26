# AWS Agentic AI Patterns

A practical, security-focused AWS proof of concept demonstrating how agents connect to enterprise systems while preserving identity and enforcing authorisation.

## Architecture

**Interactive:** Web chat → Microsoft Entra ID → API Gateway → thin request adapter → AgentCore Runtime U (JWT) → AgentCore Gateway G1 (JWT/Cedar) → MCP tools → downstream services.

**Scheduled:** EventBridge Scheduler → IAM Lambda → AgentCore Runtime M (IAM) → AgentCore Gateway G2 (IAM) → read-only network health summary.

Two Runtimes and two Gateways separate **human JWT** and **machine IAM** inbound authentication. Each Gateway target independently configures its downstream credentials (OBO, 3LO, 2LO or IAM). See [ADR-001](docs/architecture-decisions.md#adr-001--separate-human-jwt-and-automation-iam-gateways) for the decision rationale.

The synthetic demonstration includes five named human roles (Sarah, James, Emma, David and Oliver), one Network Health Reporter service, a Knowledge Base, Network and Ticket APIs, an independent vendor API, a Platform Health API, and RDS PostgreSQL behind the internal APIs.

## Business use cases

Six synthetic business journeys give each security and integration pattern a concrete purpose.

| ID | Persona | Use case |
|---|---|---|
| UC01 | **Emma — Service Desk** | Check basic asset status and retrieve standard outage procedures |
| UC02 | **James — VIC Engineer** | Diagnose Victorian faults and escalate through authorised tickets |
| UC03 | **Sarah — National Admin** | Review national faults and create confirmed, authorised tickets |
| UC04 | **David — External Analyst** | Connect a separate NetworkMonitor account and retrieve his vendor alerts |
| UC05 | **Oliver — Platform Operations** | View platform health through a native REST API integration |
| UC06 | **Network Health Reporter — automation** | Generate a scheduled, read-only aggregate health summary without human sign-in |

## Architecture patterns

Seven reusable patterns explain **how** the journeys are implemented and secured.

| ID | Pattern | Demonstrated by |
|---|---|---|
| P01 | **Corporate SSO + JWT** | Entra sign-in and validated access through API Gateway and Runtime U |
| P02 | **OAuth On-Behalf-Of (OBO)** | James accesses Network/Ticket APIs under delegated identity |
| P03 | **IAM + OAuth 2LO** | Scheduled reporter uses IAM in AWS and a scoped downstream application token |
| P04 | **Third-party OAuth 3LO** | David separately authorises the Cognito-backed vendor account |
| P05 | **MCP Gateway + Cedar** | Tool- and parameter-level access decisions independent of the model |
| P06 | **REST APIs + PostgreSQL RLS** | James receives VIC rows; Sarah receives authorised national data |
| P07 | **Shared RAG retrieval** | Bedrock Knowledge Base provides approved procedure excerpts and citations |

The design exposes **eight MCP tools** across the two Gateways. [See the complete tool and downstream API catalogue](docs/tool-and-api-inventory.md).

## Technical documentation

- [Architecture and Gateway mapping](docs/two-gateway-design.md)
- [Users and permissions](docs/identities-and-permissions.md)
- [Downstream systems and MCP tools](docs/tool-and-api-inventory.md)
- [2LO, 3LO and OBO notes](docs/oauth-token-flows.md)
- [Architecture decisions](docs/architecture-decisions.md)

**Status:** Architecture design under reconciliation. No infrastructure has been deployed or identity flows proven yet.
