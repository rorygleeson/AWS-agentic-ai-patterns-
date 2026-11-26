# AWS Agentic AI Patterns

A practical, security-focused AWS proof of concept exploring how identity, authorisation and API integration work across an AI agent architecture.

**Reference flow:** Web app → Amazon API Gateway → Amazon Bedrock AgentCore Runtime (Strands) → AgentCore Gateway → MCP tools → downstream REST APIs.

The project will compare agreed patterns: interactive and machine identities, service-account access versus end-to-end on-behalf-of (OBO), REST API versus OpenAPI Gateway targets, tool-level restrictions, and Cedar-based policy evaluation. Two Gateways are included for learning and comparison.

## Project status

**Stage 0 — Project foundation.** Documentation and design come before implementation. No deployed infrastructure yet.

➡️ **[Project board / master TODO](TODO.md)** — authoritative stages, tasks, progress and next action.

**Architecture notes:** [2LO, 3LO and OBO](docs/oauth-token-flows.md) · [Identity/permissions matrix](docs/identities-and-permissions.md) · [Two-Gateway design](docs/two-gateway-design.md).

## How we work

- One master `TODO.md` holds **all known planned work**, including documentation, architecture, implementation, tests, security, cost controls and teardown.
- Every task has a stable ID and a checkbox; we check it only when evidence confirms completion.
- We finish or explicitly defer tasks before progressing; new stages can be appended as we learn.
- Design decisions live in a standalone ADR; detailed patterns and Q&A will be separate documents.
- Never commit credentials, tokens, secrets or real user data.

**Status:** Planning. The architecture is a proposed PoC, not an implemented or validated system.
