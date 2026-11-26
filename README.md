# Enterprise AI Agent — AWS AgentCore PoC

This proof of concept uses fictional enterprise systems and users to demonstrate how an AI agent can access different systems with different permissions.

## 1. End systems

| # | End system | Description | High-level access rule |
|---|---|---|---|
| **1** | **Knowledge Base (RAG)** | Retrieves information from shared troubleshooting documents using Amazon Bedrock Knowledge Bases. | **All human users who successfully sign in to the AI chat app** can search the same approved knowledge. |
| **2** | **Network Operations API** | Enterprise REST API that retrieves network asset status, diagnostics and aggregate health information from **Amazon RDS PostgreSQL**. | **Only authorized users.** Access depends on the user's permissions and region. The service account has separate aggregate-only access. |
| **3** | **Ticket Management API** | Enterprise REST API that reads and creates support tickets stored in **Amazon RDS PostgreSQL**. | **Only authorized users.** Permissions determine which tickets they can read or create and in which regions. |
| **4** | **NetworkMonitor** | Simulated third-party SaaS returning monitoring alerts belonging to an individual vendor user. | **Only users granted tool access who separately authorize their vendor account.** Each user sees only their permitted alerts. |
| **5** | **Platform Health API** | AWS API Gateway REST API returning synthetic AI platform-health information. | **Only users explicitly granted access to this tool.** The downstream API authenticates the Gateway's IAM role. |

### Supporting database — Amazon RDS PostgreSQL

- Stores network assets, diagnostics, tickets and access entitlements.
- Is accessed **only through the Network and Ticket application APIs**, never directly by the AI agent.
- Uses database permissions and **row-level security (RLS)** to restrict records.

## 2. Users and their access

The application has **five human users** and **one non-human service account**.

| User | Role | Systems they can access | Exact permissions |
|---|---|---|---|
| **Sarah** | National Admin | **RAG, Network API, Ticket API** | Shared knowledge; network status and diagnostics in **all regions**; read/create tickets nationally. |
| **James** | VIC Engineer | **RAG, Network API, Ticket API** | Shared knowledge; network status, diagnostics and tickets for **VIC only**. |
| **Emma** | Service Desk | **RAG, Network API** | Shared knowledge; **basic asset status in all regions**. No detailed diagnostics or sensitive fields. |
| **David** | External Analyst | **RAG, NetworkMonitor** | Shared knowledge; **his own third-party alerts**, after separate vendor authorization. |
| **Oliver** | Platform Analyst | **RAG, Platform Health API** | Shared knowledge; **read-only platform-health information**. |
| **Network Health Reporter** | **M2M Service Account** | **Network API** | **Aggregate network-health summary only**, using its own application identity. No individual asset access. |

### Explicit restrictions

| User | Access that must be denied |
|---|---|
| **Sarah** | NetworkMonitor alerts and Platform Health API. |
| **James** | NSW/other-region records, NetworkMonitor and Platform Health API. |
| **Emma** | Detailed diagnostics, sensitive asset fields, Ticket API, NetworkMonitor and Platform Health API. |
| **David** | Internal Network API, Ticket API and Platform Health API. |
| **Oliver** | Network API, Ticket API and NetworkMonitor. |
| **Network Health Reporter** | RAG, individual asset records, diagnostics, tickets, NetworkMonitor, Platform Health and human impersonation. |

**Default rule:** A successful sign-in allows a human to use the AI chat app and shared RAG. Every other system requires its own authorization. If access is not explicitly listed above, it is denied. Ticket creation additionally requires explicit confirmation.
