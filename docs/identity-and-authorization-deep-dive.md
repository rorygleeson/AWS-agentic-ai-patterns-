# Identity and authorization — detailed reference

[← Main README](../README.md)

## Deep dives — identity and authorization

> **Draft appendix — pending reconciliation with the simplified architecture.** The main sections above define the proposed PoC baseline: the Enterprise AI Application invokes AgentCore Runtime directly, without a custom Chat API or forwarding Lambda. Detailed Entra registrations, token names and OBO walkthroughs below were written for an earlier design and **must not be treated as validated implementation instructions** until revised and integration-tested.


The sections below are optional implementation detail. The main walkthrough above is complete without them. The linked [architecture baseline](docs/architecture-baseline.md) and [architecture decisions](docs/architecture-decisions.md) follow the same three invocation paths.

### Deep dive: Entra ID groups, app roles, OAuth scopes and OBO

#### 1. Start with the business problem

James and Emma both sign in to **our Enterprise AI Application**. James is a VIC engineer; Emma works on the Service Desk. Both may have an OAuth token that includes `diagnostics.read`, yet **James may retrieve VIC diagnostics and Emma may not retrieve any diagnostics**.

That is not contradictory. The token's scope is **only one check**:

- **May this person invoke the AI diagnostics tool?**
  - **Mechanism:** **AI Backend app role**, enforced by AgentCore G1 **Cedar**
  - **Our example:** James allowed; Emma denied

- **May the AI client request the Network Operations API diagnostics operation for a signed-in person?**
  - **Mechanism:** **Delegated OAuth scope**, enforced by API Gateway
  - **Our example:** `diagnostics.read` is required

- **May this person access diagnostics for this asset and region?**
  - **Mechanism:** **Entra ID group** membership, mapped by Network Operations API Lambda to business entitlements
  - **Our example:** James VIC allowed; NSW denied; Emma denied

- **Which database rows may the request return or change?**
  - **Mechanism:** PostgreSQL **row-level security (RLS)**
  - **Our example:** James's request cannot retrieve NSW rows


**The key rule:** An authenticated user and a valid OAuth scope are not sufficient to access every API operation or record. The final Network Operations API must still enforce the individual's rights.

#### 2. Four concepts, explained simply

- **Entra ID group**
  - **Plain-English meaning:** A named collection of users, such as the VIC engineering team. **We use ordinary Entra ID groups**, not an API Gateway-specific group.
  - **Where we configure it:** Entra ID → Groups

- **App role**
  - **Plain-English meaning:** A named permission **within a particular application**, such as `Network.Engineer`.
  - **Where we configure it:** App registration → App roles; assign through Enterprise Applications

- **OAuth delegated scope**
  - **Plain-English meaning:** Permission for a **client application to request an operation on behalf of a signed-in person**, such as `diagnostics.read`.
  - **Where we configure it:** Network Operations API app registration → Expose an API; grant delegated API permission/consent to the middle tier

- **OBO (On-Behalf-Of)**
  - **Plain-English meaning:** A token exchange: our confidential AI backend asks Entra for a **new Network Operations API token representing the same signed-in person**.
  - **Where we configure it:** AI Backend confidential client + AgentCore Identity OAuth credential provider


**These do not automatically map to each other.** Adding James to an Entra ID group does not automatically grant an app role, make him an OAuth client, or issue an OBO token. Granting `diagnostics.read` does not turn Emma into an engineer.

**Entra ID group names versus IDs:** We write names such as `Network-Engineers-VIC` because people can read them. Entra also assigns each group a unique **Object ID**. Our Network Operations API authorizes based on the **verified Object ID**, not a name supplied in an agent prompt or HTTP request.

#### 3. Exactly what we create and assign in Microsoft Entra

**A. Human Entra ID groups — three in total**

- **`Network-Admins`**
  - **Member:** Sarah
  - **What our Network Operations API rules grant:** Status, diagnostics and ticket creation for all regions

- **`Network-Engineers-VIC`**
  - **Member:** James
  - **What our Network Operations API rules grant:** Status, diagnostics and ticket creation for VIC only

- **`Network-ServiceDesk`**
  - **Member:** Emma
  - **What our Network Operations API rules grant:** Basic asset status for all regions; **no** diagnostics or tickets


These are proposed names for **normal Entra ID groups**. **There is no special Entra ID group registered to AWS API Gateway, and no group that is enabled for OBO.** David and Oliver need **no Entra ID group for the Network Operations API** because their permitted tools point to different systems. Neither non-human caller uses these human Entra ID groups.

**B. Three Entra app registrations for the human flow**

- **`enterprise-ai-web`**
  - **Represents:** Browser chat client
  - **What we define or configure:** Interactive authorization code + PKCE; permission to call the AI Backend's `chat.invoke` delegated scope

- **`enterprise-ai-backend`**
  - **Represents:** Protected Chat API (`POST /ai/chat`) **and** confidential client for OBO
  - **What we define or configure:** Expose `chat.invoke`; define the human app roles below; configure the **same backend application ID** for the confidential OBO client

- **`enterprise-network-api`**
  - **Represents:** Protected Network Operations API
  - **What we define or configure:** Expose delegated scopes `status.read`, `diagnostics.read`, `tickets.write`; define app-only role `HealthSummary.Read`; configure appropriate group claims for this API if using token membership

**Separate Entra automation registrations (not part of those three human-flow registrations):** `entra-automation-client` is the non-human client. It is granted `Automation.Runtime.Invoke` on `enterprise-ai-backend`, `Automation.Summary.Invoke` on a **distinct** `enterprise-agentcore-g1` resource registration, and `HealthSummary.Read` on `enterprise-network-api`. Runtime, G1 and the final API each receive a separately audience-scoped Client Credentials token. The existing AWS scheduled reporter uses its own client registration `network-health-reporter` only for G2's **outbound** Network Operations API token.


**Why must the backend OBO client match Token A's audience?** Microsoft permits the middle tier to redeem an incoming token only when the assertion was issued **for that confidential middle-tier application**. Using some other client ID to exchange a Chat API token is not our design and can fail token exchange.

**C. Human AI-application roles — assigned directly in this PoC**

- **Sarah**
  - **Chat API app role in Token A:** `Network.Admin`
  - **Entra ID group for the Network Operations API:** `Network-Admins`

- **James**
  - **Chat API app role in Token A:** `Network.Engineer`
  - **Entra ID group for the Network Operations API:** `Network-Engineers-VIC`

- **Emma**
  - **Chat API app role in Token A:** `Network.ServiceDesk`
  - **Entra ID group for the Network Operations API:** `Network-ServiceDesk`

- **David**
  - **Chat API app role in Token A:** `Vendor.Analyst`
  - **Entra ID group for the Network Operations API:** None

- **Oliver**
  - **Chat API app role in Token A:** `Platform.Analyst`
  - **Entra ID group for the Network Operations API:** None


We assign these human **app roles directly to users** in the AI Backend's Enterprise Application. **Separately**, we maintain their Entra ID group memberships. This keeps the two concepts easy to demonstrate, including in an Entra Free learning tenant.

**Could we assign an Entra ID group to an app role instead? Yes.** Microsoft supports assigning groups to an application's user-facing roles, allowing assigned members to receive role claims. That is an alternative enterprise pattern; we are **not** using it in this PoC. We do not implicitly turn Entra ID group membership into an app role or scope.

**D. Delegated permissions and consent for OBO**

The Network Operations API defines the three **delegated scopes**. The confidential `enterprise-ai-backend` client is configured with the **Network Operations API delegated permissions** it needs, and the required user/admin consent is granted. This is **client/API authorization**, not an Entra ID group assignment. The Network Operations API's HTTP methods each require the appropriate scope.

#### 4. Token A and Token B: why they are different

A person signs in **once**, but each protected API expects a token intended for **that API**.

**Token A — AI Backend**

- **Who obtains it?:** The browser client following interactive Entra sign-in
- **Issued by:** Microsoft Entra ID
- **Intended audience (`aud`):** `enterprise-ai-backend`
- **Example delegated scope (`scp`):** `chat.invoke`
- **User:** James (`tid`, `oid`)
- **Additional claims:** AI Backend-specific user roles may appear in `roles`
- **Who validates it?:** Chat API and configured JWT-based AgentCore boundaries

**Token B — Network Operations API**

- **Who obtains it?:** The **confidential AI backend** using AgentCore Identity **OBO**
- **Issued by:** Microsoft Entra ID
- **Intended audience (`aud`):** `enterprise-network-api`
- **Example delegated scope (`scp`):** `diagnostics.read`
- **User:** **The same James** (`tid`, `oid`)
- **Additional claims:** Group claims can appear **if configured for the Network Operations API**; no automatic copying from Token A
- **Who validates it?:** Network Operations API Gateway authorizer and its backend


**Why not send Token A to every system?** Because the Network Operations API must reject a token addressed to the Chat API. Accepting an unrelated token audience creates a security boundary failure. OBO obtains a properly addressed token without another interactive sign-in.

**What is inside a token?** An Entra access token is a **signed** message. `aud` identifies the intended API; `scp` carries delegated scopes; `roles` carries application roles or user app roles for that token's resource; `tid` and `oid` identify the Entra tenant and user. **Not every token contains every claim.** We must explicitly configure and verify the claims each receiving service requires.

#### 5. OBO: exactly who does it and what enables it

**OBO is not a feature switched on for particular Entra ID groups.** Entra ID is the **token issuer**. Our **confidential AI backend**, through its configured **AgentCore Identity OAuth credential provider**, is the **requesting client**.

1. **James signs in** to our Enterprise AI Application using Entra (authorization code with PKCE), and the browser receives **Token A**, intended for the AI Backend.
2. The chat backend and AgentCore JWT ingress **validate Token A**. AgentCore G1 Cedar examines James's assigned AI app role when he requests the diagnostics tool.
3. **AgentCore Identity's Microsoft Entra OAuth 2.0 OBO provider** uses James's incoming Token A, the confidential backend's client ID and credential, and the configured delegated Network Operations API permission to request a new access token from **Entra's token endpoint**.
4. Entra verifies the middle-tier identity, Token A audience and the relevant **delegated permission/consent**. If these conditions are not met, **no Token B is issued**.
5. Entra issues **Token B** for the Network Operations API, **still identifying James** and containing the granted Network Operations API delegated scope.
6. AgentCore's **OpenAPI target** calls the Network REST endpoint with **Token B**; the Network Operations API validates it and enforces James's business rights.

**Who can participate?** Sarah, James and Emma are human users and can all be represented by OBO tokens **if the application setup and delegated permissions allow the exchange**. Group membership decides what they may **do afterward**. David and Oliver also have human identities, but we give them **no Network Operations API authorization** in this PoC. The two service accounts (M2M) have **no human user token**, so they use **Client Credentials**, not OBO.

**Who does not request OBO?** Not the Entra ID group; not API Gateway simply because it hosts the endpoint; not RDS. AgentCore Identity performs the configured exchange for the confidential middle tier.

#### 6. How the Network Operations API enforces endpoint and regional access

The **Network Operations API** is one REST API backed by one RDS PostgreSQL database. Its four endpoints are:

- **`GET /assets/{id}/status`**
  - **OAuth/API permission required:** Delegated `status.read`
  - **Additional Network Operations API rules:** Sarah all regions; James VIC; Emma nationwide **basic fields only**

- **`GET /assets/{id}/diagnostics`**
  - **OAuth/API permission required:** Delegated `diagnostics.read`
  - **Additional Network Operations API rules:** Sarah all regions; James VIC; Emma **denied**

- **`POST /tickets`**
  - **OAuth/API permission required:** Delegated `tickets.write`
  - **Additional Network Operations API rules:** Sarah all regions; James VIC; Emma **denied**

- **`GET /network/health-summary`**
  - **OAuth/API permission required:** **App-only** `roles: [HealthSummary.Read]`, not a delegated `scp`
  - **Additional Network Operations API rules:** **Only** the two approved non-human Entra application identities; health summary only


**Where does each check happen?**

1. **API Gateway REST JWT authorizer** validates the Entra signature, tenant issuer, **Network Operations API audience**, expiry and required delegated **method scope** for human endpoints. The app-only summary method deliberately has **no `scp` requirement**, because Client Credentials tokens use `roles`.
2. **The Network Operations API Lambda** obtains the **verified** user `tid/oid` and Entra ID group Object IDs from correctly issued Network Operations API claims, or an authorized membership lookup. It determines permissions from the RDS **`group_entitlements`** table. **Entra is the source of truth for group membership; RDS is the source for our business mappings.**
3. Lambda checks the actual asset's **trusted region**, permitted operation and, for Emma, **permitted response fields**. It rejects a prompt-supplied user ID, region or group claim as an authorization source.
4. Lambda establishes trustworthy **per-request database context**. PostgreSQL **RLS** filters permitted rows. RLS does **not** read an OAuth token by itself, and RLS does **not** redact columns; the API shapes Emma's basic response.
5. For `POST /tickets`, Lambda checks the caller's ticket-creation permission and verified regional entitlement before writing to the database.

**Same OAuth scope, different person:**

- **James**
  - **Example delegated scope:** `diagnostics.read`
  - **Attempt:** VIC diagnostics
  - **Result:** **Allow**

- **James**
  - **Example delegated scope:** `diagnostics.read`
  - **Attempt:** NSW diagnostics
  - **Result:** **Deny** — outside VIC

- **Emma**
  - **Example delegated scope:** `diagnostics.read` (even if her authorized client obtained it)
  - **Attempt:** Any diagnostics
  - **Result:** **Deny** — Service Desk has no diagnostics entitlement

- **Emma**
  - **Example delegated scope:** `status.read`
  - **Attempt:** Basic NSW status
  - **Result:** **Allow** — basic fields only


**Why isn't OAuth enough?** OAuth scopes describe what the client may request on a user's behalf, not every individual business rule. An endpoint that accepts James's `diagnostics.read` token must still deny James's NSW asset and Emma's diagnostics request.

**Group membership details:** The Network Operations API's Entra token can be configured to carry **Entra ID group Object IDs**. Where group claims are omitted or indicate overage, a trusted Microsoft Graph membership lookup or another authorized entitlement lookup is required. If membership cannot be verified, **deny**. Do not rely on mutable group display names or data supplied in the model's arguments.

**Our selected enforcement option:** Use the existing Network Operations API Lambda for group/operation/region/field rules and RDS RLS for rows. We are **not** creating an extra API-level Cedar engine or separate Lambda authorizer for this PoC. The Lambda uses **generic permission checks** driven by protected entitlement data; no hardcoded if-statements naming James or Emma.

| Other valid option | Why we did not choose it here |
|---|---|
| Separate API Gateway **Lambda authorizer** | Can centralize endpoint checks before the business handler, but adds another function and does not replace record/field checks |
| **Amazon Verified Permissions (Cedar)** for the API | Useful when many APIs share policy; adds a policy store and another authorization boundary beyond the AgentCore Cedar demonstration |
| API Gateway **OAuth scopes alone** | Cannot enforce VIC versus NSW or Service Desk's restricted fields |

#### 7. Where AgentCore Cedar fits — separate from API authorization

G1's **Cedar** policies authorize **MCP tool invocation** from **Token A** identity/app-role claims. For example:

- James's `Network.Engineer` role can permit the diagnostics tool. **It cannot authorize NSW data**—the Network Operations API and RLS still deny that.
- Emma's `Network.ServiceDesk` role permits the basic-status tool but **denies the diagnostics and ticket tools** before calling the API.
- David's `Vendor.Analyst` role permits the vendor tool, but does **not** replace David's independent vendor OAuth authorization.
- Oliver's `Platform.Analyst` role permits the Platform Health tool **before** Gateway uses its **shared AWS IAM role** to invoke the IAM-protected final API.
- The Entra Service Account's **application** role permits its **summary tool only**, never human tools.
- G2 uses an **IAM-principal policy** for the AWS Service Account (using IAM Role) role and exposes **health summary only**.

**Why Cedar is important for Oliver:** Platform Health validates an **AWS SigV4-signed service request** from Gateway. It does not receive Oliver's human OAuth token as its authenticated principal. Without an earlier trusted **human-specific authorization decision**, any person able to trigger that shared service role could potentially use the tool. We choose Cedar for that check.

**Is Cedar mandatory for Network Operations API OBO? No.** Because the Network Operations API identifies James and independently enforces his rights, Cedar is additional **tool-level protection**. But it does not replace the downstream API.

**Exact claims matter:** For Entra v2 access tokens, the calling application ID is commonly `azp` (v1 `appid`). AgentCore JWT `allowedClients` checks a claim named **`client_id`**, which is **not automatically the same claim**. Our G1 Cedar policy checks the verified **`azp`** and required business/app role. Do **not** assume an AWS JWT allow-list of client IDs automatically maps to Entra's claim names. Entra JWT `roles` arrays are exposed to AgentCore Cedar through **string-valued tags**; policies must check complete role values and deny missing/wrong-type claims. We do **not** require human `chat.invoke` globally at G1, because the valid Entra app-only automation token does not have human delegated `scp`.

#### 8. Service accounts (M2M): Client Credentials, not OBO

Our PoC has **two service accounts (M2M) with different inbound authentication**, but both eventually use **Entra application permissions** for the same Network Operations API summary endpoint.

- **AWS Service Account (using IAM Role)**
  - **AgentCore inbound identity:** Scheduled Lambda → G2 **AWS_IAM/SigV4** directly (no Runtime)
  - **Network Operations API outbound credential:** AgentCore Identity OAuth **Client Credentials** for dedicated Entra app `network-health-reporter`
  - **What the final API sees:** AWS Service Account (using IAM Role)'s Entra **application ID**, `HealthSummary.Read`, no human

- **Entra Service Account**
  - **AgentCore inbound identity:** Separate Entra service principal → application JWT → JWT-protected Runtime → separately scoped application JWT → G1 **CUSTOM_JWT**
  - **Network Operations API outbound credential:** Separate target's **Client Credentials** for `entra-automation-client`
  - **What the final API sees:** Entra Service Account's own Entra **application ID**, `HealthSummary.Read`, no human


**No Entra ID group or OBO is involved.** `HealthSummary.Read` is an Entra **application app role** defined on `enterprise-network-api` and granted separately to the two service principals. The Network Operations API checks each application's verified client identity and its role—not just the signature of some Entra-issued JWT.

**G2 IAM inbound does not magically become OAuth outbound.** G2's Network Operations API **OpenAPI target** has an independent **AgentCore Identity OAuth credential provider** configured as an Entra **Client Credentials client** (tenant/token endpoint, client ID, credential, Network Operations API `/.default` resource scope). AgentCore Identity requests the **Network Operations API-audience** app token. The confidential client secret is secured through **AWS Secrets Manager**; AgentCore Identity manages OAuth tokens through its encrypted **token vault**. The Gateway's IAM role is **not** the application secret, and neither is a human identity.

**Why two gateways?** An AgentCore Gateway is configured with **one inbound authorizer type**. G1 accepts Entra JWTs; G2 accepts IAM/SigV4. We deliberately keep both to teach two **service-account ingress** methods. A single G1 JWT Gateway could serve the AWS Service Account (using IAM Role) if the AWS Service Account (using IAM Role) instead obtained an Entra app-only JWT; G2 is a **chosen learning pattern**, not a universal requirement for service accounts. Each Gateway can have multiple targets with independent **outbound** authentication.

#### 9. FAQs — common IAM misunderstandings

| Question | Short answer |
|---|---|
| **Is an Entra ID group tied to API Gateway?** | **No.** It is an ordinary Entra ID group. The Network Operations API Lambda maps its verified Object ID to business rights. |
| **Which Entra ID groups support OBO?** | **None are OBO-enabled.** Entra issues tokens to a correctly configured confidential client's delegated exchange, representing an authenticated human. |
| **Can Emma obtain an OBO token?** | **Yes, if the client permissions and consent allow it.** It does not grant her diagnostics business rights. |
| **Do Entra ID group memberships become app roles automatically?** | **No.** We assign roles separately. Assigning a group to an app role is an alternative configuration we chose not to use. |
| **Do app roles automatically grant OAuth scopes?** | **No.** App roles and delegated scopes are defined and assigned separately, and may belong to different API registrations. |
| **Does API Gateway need the Entra ID group's name?** | **No.** The Network Operations API's authorization logic uses the verified **Object ID** and RDS mappings. |
| **Why not use Token A for Network Operations API?** | **Wrong audience.** Token B is issued specifically for Network Operations API via OBO. |
| **Does RLS read James's token?** | **No.** Lambda converts verified identity/entitlements into database context; RLS restricts rows. |
| **Can Cedar alone protect the Network Operations API?** | **No.** The final API must enforce its own rights, including when reached by another permitted client. |
| **Can the AWS Service Account (using IAM Role) use OBO?** | **No human is signed in.** It uses Client Credentials. |
| **Can a valid scope still be denied?** | **Yes.** Emma's business rights or James's regional restriction can deny the operation. |

**Sources:** [Microsoft OBO](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) · [Entra app roles and Entra ID groups](https://learn.microsoft.com/en-us/entra/identity-platform/howto-add-app-roles-in-apps) · [Entra access token claims](https://learn.microsoft.com/en-us/entra/identity-platform/access-token-claims-reference) · [AWS AgentCore OBO](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html) · [AWS Gateway outbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html) · [API Gateway REST JWT authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-jwt-authorizer.html).


### How OAuth connects to PostgreSQL RLS

**RLS does not read an OAuth token. The Network Operations API connects the user's OAuth identity to the database rules.**

Imagine James asks our Enterprise AI Application: **“Show me diagnostics for asset 123.”** That asset is in NSW, but James is a VIC engineer.

1. **James signs in** using Microsoft Entra ID.
2. **OBO obtains a new OAuth access token for the Network Operations API**, still representing James. It is not a generic token for the AI agent.
3. **The Network Operations API validates the token** and identifies James using trusted token claims. It then verifies his `Network-Engineers-VIC` membership with Entra claims or an authorized identity lookup.
4. **The Network Operations API establishes trusted database context for this request:** James may access **VIC** data, not NSW. It does not accept a region merely because the user or AI agent supplied it in a prompt.
5. **PostgreSQL applies its RLS policy** to the database query using that verified context. It excludes any row outside James's permitted region.
6. **The API returns only authorized data** to the agent, which uses it to answer James.

- **Sarah**
  - **Verified access from Entra and API rules:** National Admin, all-region entitlement
  - **What RLS allows:** Rows in all permitted regions

- **James**
  - **Verified access from Entra and API rules:** Engineer, VIC-only entitlement
  - **What RLS allows:** **VIC rows only**

- **Emma**
  - **Verified access from Entra and API rules:** Service Desk, nationwide basic status
  - **What RLS allows:** Status rows nationally; **the API** removes sensitive fields

- **Both approved automation callers**
  - **Verified access from Entra and API rules:** App-only OAuth permission for health summary
  - **What RLS allows:** **Not a human regional-RLS example**; the API exposes only a health-summary response


**The crucial distinction:** OAuth/OBO tells the Network Operations API **who is calling**. Verified Entra groups and our API rules determine **which data that caller may access**. The API conveys that trusted restriction to PostgreSQL, and **RLS enforces it against actual database rows**.

This is a deliberate additional security boundary. An AI agent asking for NSW data cannot override James's VIC entitlement. **RLS filters rows, not individual columns**: Emma's basic-only response is also restricted by the Network Operations API. The database context must be isolated for each request, and the API's database role must not bypass RLS.

### Authentication is not the same as authorization

- **Authentication** establishes the identity of the user, application or AWS role.
- **Authorization** decides which tool, operation, region, record or field that identity may access.
- **OBO preserves user identity; it does not automatically grant all data.** The Network Operations API still enforces each user's permissions.
- **Cedar** (rules controlling which tools an agent may call) is used for restricted AgentCore tool access. **PostgreSQL RLS** enforces appropriate row restrictions behind the enterprise APIs. Shared RAG does not require a special per-user Cedar restriction.

### Two Gateway authentication boundaries

The architecture has two Gateway inbound authentication methods, but **three invocation paths**. **G1 is not a public alternative to our Chat API.** The JWT-protected Runtime reaches G1 over a **private AgentCore Gateway VPC endpoint (PrivateLink)**. G1's resource/organization policy restricts requests to the approved VPC endpoint (`aws:SourceVpce`), in addition to JWT validation and Cedar authorization; otherwise a caller could bypass the Enterprise AI Application by calling G1 directly with a token. G2 is restricted to the approved AWS Lambda execution role. JWT-protected Runtime ingress admits only the trusted human backend flow and the explicitly approved Entra automation client.

- **G1 — JWT Gateway:** accepts authorized tool calls from Runtime under **human-delegated or Entra application-only credentials**. Human access requires the applicable verified delegated identity and business role. The automation case uses the approved application's **`Automation.Summary.Invoke`** role/client identity and is restricted to the health-summary tool. Runtime must acquire this application-only G1 credential explicitly and never use it for human requests.
- **G2 — IAM Gateway:** accepts IAM/SigV4 tool requests directly from the authorized AWS Service Account (using IAM Role) scheduled Lambda.

**Why is G2 necessary in *our chosen design*?** The AWS Service Account (using IAM Role) is a scheduled AWS workload. Its Lambda uses **an AWS IAM execution role**, and we deliberately want the AWS Service Account (using IAM Role) to authenticate **into AgentCore Gateway using IAM/SigV4**—without pretending to be a human or obtaining an Entra JWT just to enter Gateway. G1 accepts **Entra OAuth JWTs**, from authorized humans and the separately registered Entra Service Account. AgentCore Gateway has **one inbound authorizer type per Gateway**; therefore **these two chosen inbound methods cannot share a single Gateway**. G2 is required *because we chose IAM inbound for this AWS workload*, not because all service accounts require their own Gateway.

**The useful cross-boundary lesson:** AWS Service Account (using IAM Role)'s Lambda → **G2 (IAM/SigV4 inbound)** → a restricted health-summary OpenAPI target → **Network Operations API (Entra OAuth 2.0 Client Credentials outbound)** → RDS. The final Network Operations API does not accept the AWS Service Account (using IAM Role)'s AWS role as an OAuth token. G2 uses the AWS Service Account (using IAM Role)'s Entra application identity to obtain a separate Network Operations API token; the API authorizes **health summary only**.

#### Exactly what we configure in AWS and Entra

**Yes: G2's Network Operations API target is configured to use an OAuth client.** That client is an **Entra application/service principal** we register, and AgentCore Identity knows **which Entra tenant and OAuth token endpoint to contact**. We are not writing our own OAuth implementation.

**In Microsoft Entra ID:**

1. Register `network-health-reporter` as a **confidential client application**. Entra gives it an **Application (client) ID**; create a **client secret**.
2. In the separately registered `enterprise-network-api`, define an **application permission/app role** called `HealthSummary.Read`. Assign this permission to `network-health-reporter` and grant the required administrator consent. This grants the application permission to request a summary, **not** a human user's permissions.
3. Record our **Entra tenant ID**, the reporter application's **client ID**, and the **Network Operations API's Application ID URI** (for example, `api://<network-api-app-id>`).

**In AWS AgentCore Identity (outbound OAuth configuration):**

1. Create an **OAuth 2.0 credential provider**, choosing **Microsoft** as the provider. Supply the **Entra tenant**, **reporter client ID**, and the reporter **client secret** or a reference to that secret in **AWS Secrets Manager**.
2. AgentCore Identity uses the Microsoft tenant/provider configuration to locate Entra's **OAuth token endpoint**, such as `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`. This is the **connection to Entra ID**; it is not a network connection configured on the AWS IAM role.
3. On **G2's Network Operations OpenAPI target**, choose **Outbound authentication = OAuth**, select that **credential provider**, and set **Grant type = Client Credentials**. Request the Network Operations API resource's `/.default` scope (example: `api://<network-api-app-id>/.default`), which asks Entra for the **application permissions already granted** to this client.
4. Give G2's service role the documented permissions to use that credential provider and its underlying secret. **G2 is not configured with a human JWT for inbound access**—its inbound authentication stays **AWS IAM/SigV4**.

**At runtime:** A signed IAM request arrives at G2 → G2 chooses its health-summary target → **AgentCore Identity** uses the configured Entra **client ID + secret** to ask Entra for an access token → Entra returns a token addressed to `enterprise-network-api` with the approved app role → G2 calls the Network Operations API with `Authorization: Bearer <access-token>`. The API validates the token and authorizes the **summary endpoint only**.

**Where is what stored?** The **client secret** is secured through **Secrets Manager**; the **OAuth provider configuration** belongs to **AgentCore Identity**; issued **access tokens** are managed in AgentCore Identity's encrypted **token vault**. They are **three different things**.

### Two ways to connect REST APIs to AgentCore tools

**First, what is an AgentCore Gateway target?** A **target** is the Gateway's configuration for connecting an AI agent or an authorized automated client to a backend system. It tells Gateway **where the system is, which operations to expose as tools, and how Gateway authenticates when calling it**. A target is **not a new API or another running service**.

**Model Context Protocol (MCP)** is the standard interface an agent or approved automated client uses to discover and call those tools.

Think of the terms this way:

- **End system:** Our Network Operations API.
- **Endpoint:** One operation on that API, such as `GET /assets/{id}/diagnostics`.
- **AgentCore Gateway target:** The configured connection to that API, including its operation definitions and outbound authentication.
- **MCP tool:** The action an agent or approved automated client invokes, such as `get_diagnostics`. Gateway translates the tool call into the REST endpoint request.

**One Gateway can contain several targets, and one target can expose several tools.** Our **one Network Operations API** has four endpoints, but it is connected through **three Gateway target configurations**, because the human users and two service accounts have distinct outbound OAuth credentials:

- **G1 human OpenAPI target:** Exposes **status, diagnostics and create-ticket** tools. Gateway uses **OBO** to call the Network Operations API as the signed-in person.
- **G1 Entra Automation OpenAPI target:** Exposes **health summary only** for the verified Entra Service Account JWT. Gateway uses that application's **Client Credentials** for the Network Operations API.
- **G2 AWS Service Account (using IAM Role) OpenAPI target:** Exposes **health summary only** for the authorized AWS Lambda IAM execution role. Gateway uses the AWS Service Account (using IAM Role)'s **separate Client Credentials** for the Network Operations API.

**Three target configurations do not mean three copies of the API.** All three targets call the **same Network Operations API** and RDS database. We keep the human and two service-account targets separate; **Cedar tool authorization denies** both service accounts from invoking human operations. Target registration alone does not guarantee per-caller tool discovery filtering.

**Both integration approaches connect to REST APIs.** REST APIs can use OAuth or IAM; the distinction here is the type of **AgentCore Gateway target** and its supported outbound credentials.

**Native AWS API Gateway stage target**

- **Our example:** Platform Health API
- **What we give AgentCore:** An existing AWS API Gateway REST API stage
- **Outbound authentication AgentCore supports:** **AWS IAM/SigV4** or API key; **no OAuth option** for this target type
- **Identity the final API receives in our PoC:** Gateway's **AWS IAM service role**

**OpenAPI schema target**

- **Our example:** Network Operations API
- **What we give AgentCore:** An **OpenAPI document** listing the REST URLs, operations and parameters
- **Outbound authentication AgentCore supports:** **OAuth**, including OBO for a person and Client Credentials for a service
- **Identity the final API receives in our PoC:** A **user-delegated Entra token** or the service account's Entra token


**Why does Network Operations need the OpenAPI option?** Its REST endpoints must identify the human caller to enforce regional permissions. When James requests VIC diagnostics:

1. AgentCore discovers the diagnostics tool from the **OpenAPI definition**.
2. AgentCore's configured **OAuth credential provider** uses OBO to obtain an Entra token **for the Network Operations API**, still representing James.
3. Gateway calls the REST endpoint with that token. **The Network Operations API validates it** and enforces James's VIC-only permissions.

The native API Gateway stage target **cannot perform step 2 using OAuth outbound authentication**. This is a **limitation of that AgentCore target type**, **not of REST APIs or AWS API Gateway itself**. The actual REST endpoint may still be hosted on AWS API Gateway.

**Why use the native target for Platform Health?** That API is deliberately protected by **AWS IAM**, so Gateway can import the REST stage and sign requests with **its own service role (SigV4)**. Oliver must first be authorized to use the tool; the final API authenticates the service role, **not Oliver's Entra token**.

**Important:** The OpenAPI document **does not authenticate anyone or create another service**. It describes the existing REST API. AgentCore's credential provider obtains the OAuth token, and the downstream API validates and authorizes it. See AWS's [Gateway outbound-authentication matrix](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html) and [native API Gateway target limitation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html).

These are our **core integration/authentication patterns**. Detailed session handling, Cedar policies, database controls, logging and implementation decisions belong in the supporting architecture documents—not in the introductory access tables.
