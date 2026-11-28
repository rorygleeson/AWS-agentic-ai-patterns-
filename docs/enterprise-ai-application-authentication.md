# Enterprise AI Application authentication

[← Main README](../README.md)

## Enterprise AI Application authentication

*Implementation deep dive — to be completed when we focus on the frontend/backend authentication design.*

The **Enterprise AI Application** is a web application that invokes **AgentCore Runtime's managed `InvokeAgentRuntime` API** directly using Entra OAuth. There is no custom Chat API, application API Gateway or forwarding Lambda in the proposed design.

This appendix will document, with a worked James example:

1. Entra sign-in using Authorization Code with PKCE, including access tokens and the browser session.
2. Browser-to-Runtime OAuth invocation, token audience, CORS and session behaviour.
3. Runtime's inbound JWT validation and AgentCore Identity's confidential credential provider.
4. Runtime OBO for a **G1-audience** token, then G1 OBO for a **Network Operations API-audience** token, preserving James's identity at each step.
5. Negative tests for wrong audiences, unauthorized callers and regional restrictions.

**Status:** The architectural boundaries are agreed; the precise token configuration and integration tests will be validated during implementation. This placeholder does not introduce a new application, gateway or authentication pattern.
