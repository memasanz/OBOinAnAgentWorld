# Multi-Tenant External Access — Foundry Agent in Tenant A, Users in Tenant B

This document describes the pattern for letting users in **other Entra
tenants** call a Foundry-hosted agent that operates entirely on **your
tenant's** resources, with **per-partner managed-identity isolation**.

It is the simpler sibling of the OBO docs:
- [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md)
- [`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md)

> **Hard constraints of this design:**
> 1. **No B2B guests.** No external user account is ever created in your tenant.
> 2. **No OBO.** The agent does not access the user's own M365 data
>    (Graph, SharePoint, Exchange, etc.). It only touches resources that live
>    in your tenant.
> 3. **Per-partner data isolation enforced by Azure RBAC**, not just by code.
>    Each partner gets a dedicated Foundry project whose **managed identity has
>    RBAC scoped to only that partner's data slice**. A bug in agent code
>    cannot leak data across partners — Azure IAM physically prevents it.

---

## Why a separate document?

This doc covers a specific scenario: **users in another Entra tenant calling
an agent that only touches your tenant's resources** (no Microsoft Graph, no
SharePoint, no partner-tenant directory). It's split out from the OBO docs
because the auth model is fundamentally different — and simpler.

| | OBO docs (`obo-foundry-apim-graph` / `-sharepoint`) | This doc |
|---|---|---|
| Who the user is | Same tenant as the agent | **Different tenant** from the agent |
| What the agent reads | The user's own M365 data (mail, files, sites) | **Your** Tenant A resources only |
| Auth pattern | OBO token exchange via `apim-obo-middletier` | Validated USER token + per-agent **managed identity** |
| App regs | 3 (web app, middle-tier, Graph permissions) | **1** (`agent-host-webapp`) |
| Cross-partner isolation | n/a (single tenant) | **Azure RBAC** on per-partner data slices |
| User's identity used for | Acting as the user against Graph | Tagging/filtering rows in your data |

Because OBO disappears entirely, so does the middle tier: no token exchange,
no `send-request` to AAD, no Graph permissions, no MCP-OAuth client app reg.

---

## Deployment options at a glance

There are two valid identity models for "users in another organization
calling our agent." Pick based on **whether the partner has Entra** and
**whether you want to involve the partner admin at all**.

| | **Option A — Multi-tenant Entra app reg** | **Option B — Entra External ID (CIAM)** |
|---|---|---|
| Where partner users' accounts live | Their own Entra tenant (Tenant B) | **Your External ID tenant** (a separate Entra tenant you own) |
| Identity provider for the user | Tenant B Entra | Your External ID tenant — federated to Tenant B (Entra), Google, email OTP, etc. |
| Partner admin involvement | **Required**: one-time admin consent in Tenant B | **Not required**: users sign up themselves, or you invite them via email |
| Scales to "many partners" | Limited by ability to chase down each partner admin for consent | Yes — fully self-service onboarding |
| `tid` claim in the user token | Partner's tenant ID | **Always your External ID tenant ID** — partner identity comes from a custom claim (e.g., `extension_partnerId`) or app role |
| Tenant allowlist enforcement | APIM `<issuers>` allowlist (one issuer per partner) | APIM allows the single External ID issuer; partner allowlist is a claim check |
| Works when partner is not on Entra | ❌ no | ✅ yes (federate to Google / email OTP / etc.) |
| Best for | Small number of B2B partners with their own Entra | High-volume / many partners; or partners not on Entra |

The body of this document walks through **Option A** in detail. For Option B
(Entra External ID), see **[`multitenant-external-id.md`](./multitenant-external-id.md)**.
Both options share the same per-agent-MI backend pattern; only the identity
layer (and the APIM `validate-jwt` policy) differs.

---

## Two-layer isolation model

This is the core of the security story. Every backend access is gated by two
independent layers:

| Layer | What it isolates | How it's enforced | Who can break it |
|---|---|---|---|
| **1. Cross-partner** | Partner B's data from Partner C's data | Azure RBAC: per-partner Foundry MI has access **only** to that partner's data slice | Only an Azure RBAC change in Tenant A can break this — not policy, not agent code, not a request |
| **2. Cross-user (within a partner)** | One user's data from another user's, inside the same partner | Agent code / MCP tools filter by `oid` (and `tid`) from validated headers | A bug in agent code/tools — but blast radius is limited to that **one partner's** users |

The first layer is a **hard wall** (IAM); the second is a **soft wall**
(application logic). Together: a worst-case agent bug leaks within one
partner; cross-partner leaks are physically impossible.

---

## Hosting model

- **Your tenant (Tenant A)** hosts a **dedicated Foundry project per partner**
  (one project per partner — agents, prompts, tools, MIs, all isolated).
  Your team manages each project — agent configuration, MCP tool wiring,
  deployments, observability.
- **Each partner has a dedicated data slice** in Tenant A: their own Cosmos
  database (or container), their own AI Search index, their own Storage
  container — whatever the agent needs.
- **Each per-partner Foundry project's managed identity** has Azure RBAC on
  **only that partner's slice**.
- **The partner's end users (in Tenant B)** never touch the Azure portal or
  Foundry Studio. They reach the agent through a **multi-tenant web app**
  hosted in Tenant A that signs them in against *their own* tenant.
- **Partner admins / agent builders have no direct access** to anything in
  your tenant. This is what keeps the design fully compatible with the
  no-B2B-guests rule.

---

## Requirements

| # | Requirement | Owner |
|---|---|---|
| 1 | A **dedicated Foundry project per partner** is provisioned in Tenant A and managed by you on the partner's behalf | You |
| 2 | A **dedicated data slice per partner** (Cosmos DB / container, Search index, Storage container, etc.) is provisioned in Tenant A | You |
| 3 | The **per-partner Foundry project's managed identity** is granted Azure RBAC on **only that partner's data slice** | You |
| 4 | Partner end users access the agent via a **multi-tenant web app** in Tenant A, never via the Azure portal or Foundry Studio | You |
| 5 | Partner admins and agent builders have **no direct Foundry/Azure access** in your tenant | You + Partner agreement |
| 6 | `agent-host-webapp` app reg is multi-tenant (`signInAudience = AzureADMultipleOrgs`) and exposes an `access_as_user` scope so its access token can be the audience APIM validates | You |
| 7 | APIM has a managed identity granted `Azure AI User` (or equivalent) on each per-partner Foundry project so APIM can invoke the agent endpoint | You |
| 8 | **One-time admin consent in each partner tenant** for `agent-host-webapp` | Partner admin |
| 9 | APIM `validate-jwt` policy enforces an explicit **issuer allowlist** (one entry per accepted partner tenant) | You |
| 10 | Agent code / MCP tools enforce per-user separation within the partner's slice using `tid` + `oid` from validated headers | You |

> **Requirement 8 is non-negotiable in Entra.** No OAuth flow issues a user
> token for an app in Tenant B without a Tenant B admin consenting at least
> once. After the one-time consent, the partner admin has no further required
> tasks. Only **one** consent is needed (vs. two in the previous OBO-based
> drafts) because there is no MCP-OAuth client app reg.

### Out of scope by design

- ❌ B2B guest invitations
- ❌ OBO to Microsoft Graph / SharePoint with the user's identity
- ❌ Hosting a single Foundry project that external users sign into directly
- ❌ Putting external users in your Entra security groups
- ❌ Assigning external users to your app roles
- ❌ Reading the user's mail / calendar / OneDrive / partner-tenant directory
- ❌ Sharing a single managed identity across partners (would weaken the
  cross-partner isolation guarantee)

---

## Architecture

```
┌─────────────────────────┐                  ┌──────────────────────────────────────────────────┐
│   Partner tenant (B)    │                  │   Your tenant (A) — hosts everything             │
│                         │                  │                                                  │
│   ┌────────────────┐    │  1. user sign-in │   ┌────────────────────────────────┐             │
│   │ Partner end    │    │  (multi-tenant   │   │ agent-host-webapp              │             │
│   │ user (browser) │────┼──auth-code)──────┼──►│ (multi-tenant web app)         │             │
│   └────────────────┘    │                  │   └──────────────┬─────────────────┘             │
│                         │                  │                  │ POST /chat                    │
│                         │                  │                  │ Authorization: Bearer USER tk │
│                         │                  │                  │ (aud=agent-host-webapp)       │
│                         │                  │                  ▼                               │
│   ┌────────────────┐    │                  │   ┌────────────────────────────────┐             │
│   │ Entra (B)      │    │                  │   │ APIM  /chat  API               │             │
│   │ - issues USER  │    │                  │   │ - validate-jwt (issuer ∈ B)    │             │
│   │   tokens for   │    │                  │   │ - extract tid, oid             │             │
│   │   agent-host-  │    │                  │   │ - lookup tid → Foundry URL     │             │
│   │   webapp       │    │                  │   │ - swap auth → APIM MI token    │             │
│   │ - SP for       │    │                  │   │ - x-user-tid / x-user-oid hdrs │             │
│   │   agent-host-  │    │                  │   │ - rate-limit by tid            │             │
│   │   webapp       │    │                  │   └──────────────┬─────────────────┘             │
│   │   created on   │    │                  │                  │                               │
│   │   one-time     │    │                  │                  ▼                               │
│   │   consent      │    │                  │   ┌────────────────────────────────────────┐     │
│   └────────────────┘    │                  │   │ Foundry project — Partner B (dedicated)│     │
│                         │                  │   │ ┌──────────────────────────────────┐   │     │
│                         │                  │   │ │ Agent + MCP tools                │   │     │
│                         │                  │   │ │ - read x-user-tid / x-user-oid   │   │     │
│                         │                  │   │ │ - filter all queries by oid      │   │     │
│                         │                  │   │ └────────────┬─────────────────────┘   │     │
│                         │                  │   │              │ direct call as           │     │
│                         │                  │   │              │ Foundry-B's MI            │     │
│                         │                  │   └──────────────┼──────────────────────────┘     │
│                         │                  │                  ▼                               │
│                         │                  │   ┌────────────────────────────────┐             │
│                         │                  │   │ Partner B's data slice         │  ◄── RBAC: │
│                         │                  │   │ - Cosmos DB "partnerB-conv"    │   only     │
│                         │                  │   │ - Search index "partnerB-kb"   │   Foundry  │
│                         │                  │   │ - Storage cnt "partnerB-files" │   B's MI   │
│                         │                  │   └────────────────────────────────┘             │
│                         │                  │                                                  │
│                         │                  │   (Partner C has its own Foundry project,        │
│                         │                  │    its own MI, and its own data slice.           │
│                         │                  │    Partner B's MI has NO RBAC on Partner C's     │
│                         │                  │    resources, and vice versa.)                   │
└─────────────────────────┘                  └──────────────────────────────────────────────────┘
```

Key observations:

- **Three things live in Tenant A per partner**: a Foundry project, a data
  slice, and an RBAC binding between the two. Partners are isolated at the
  IAM layer.
- **One multi-tenant app registration** is involved: `agent-host-webapp`. No
  OBO middle tier, no MCP-OAuth client app reg.
- **APIM is in front of `/chat` only.** Its job: validate the inbound USER
  token, enforce the tenant allowlist, route to the right Foundry project, and
  pass the user identity to the agent as headers. APIM authenticates to
  Foundry as its own MI (granted `Azure AI User` on each per-partner project).
- **APIM is NOT in front of the data path.** The agent calls Cosmos / Search /
  etc. directly using its **own** managed identity. There is no centralized
  data API.
- **Authorization is two-layer:**
  1. *Cross-partner* — Azure RBAC. Foundry-B's MI cannot see Partner C's data
     because it has no role assignment on Partner C's resources.
  2. *Cross-user within a partner* — agent/MCP-tool code filters every read
     and write by the validated `oid` from the headers.
- **The partner admin's only required action** is one-time admin consent in
  Tenant B for `agent-host-webapp`. After that, no further partner-side
  action is required.
- **USER tokens carry partner-tenant identity**: `iss = https://login.microsoftonline.com/<tenantB>/v2.0`,
  `tid = <tenantB>`, `oid = <user oid in tenant B>`. APIM enforces the tenant
  allowlist via `<issuers>` and forwards `tid`/`oid` as headers.
- **Partner end users never sign into the Azure portal or Foundry Studio.**
  Their entire experience is the multi-tenant web app.

---

## Internal users (your workforce tenant)

Internal users from your workforce tenant (Tenant A) use the **same path** as
external partners — same web app, same APIM `/chat`, same agent flow.
Mechanically:

- The `<issuers>` allowlist already includes
  `https://login.microsoftonline.com/{{YOUR_TENANT_ID}}/v2.0` alongside the
  partner tenants, so workforce-issued tokens validate the same way.
- Internal users' tokens carry `tid = <yourTenantId>` and a workforce `oid`.
- Treat your tenant ID as just another `PARTNER_FOUNDRY_MAP` key:

  ```json
  {
    "<your-tenant-id>":     "https://eastus.api.azureml.ms/.../internal-project",
    "<partner-tenant-1-id>":"https://eastus.api.azureml.ms/.../partnerB-project",
    "<partner-tenant-2-id>":"https://eastus.api.azureml.ms/.../partnerC-project"
  }
  ```

- The internal Foundry project's MI gets RBAC on the **internal** Cosmos /
  Search / Storage slice — same hard-IAM-wall pattern as for any partner.
  Internal users **cannot** see partner data, and partner Foundry MIs
  **cannot** see internal data.
- Inside the agent, the `oid` filter still applies for per-user separation
  among internal users (one employee can't read another's conversations).

A few things specific to internal users:

- **No admin-consent step.** Your tenant already trusts
  `agent-host-webapp` since it's registered there.
- **Conditional Access for internal users is your standard workforce CA.**
  No External ID layer is involved — MFA, device compliance, and risk
  policies apply directly during sign-in.
- **No partner SP-materialization concern.** Internal users don't need the
  multi-tenant consent dance.
- **Don't put internal data in a partner's slice.** It's tempting to
  collapse "internal users" into one of the partner buckets to avoid
  spinning up an internal slice — don't. Keep internal as its own
  Foundry project + data slice for the same reasons you keep partners
  separate.

---

## App registration

Only one app reg is required for this pattern.

### `agent-host-webapp` — multi-tenant web app + audience

Lives in Tenant A. Consented once per partner tenant.

- **Supported account types:** *Accounts in any organizational directory
  (Multitenant)*. Sets `signInAudience = AzureADMultipleOrgs`.
- **Redirect URI:** the web app's sign-in callback (e.g.,
  `https://contoso-agent-host.azurewebsites.net/.auth/login/aad/callback`
  for App Service Easy Auth, or your own MSAL callback)
- **Expose an API:** add scope `access_as_user`. This makes
  `api://<agent-host-webapp-id>/access_as_user` a valid scope the web app can
  request on the user's behalf, so APIM has a real audience to validate.
  (`Application ID URI` defaults to `api://<app-id>`.)
- **Authentication:** `accessTokenAcceptedVersion = 2` (recommended).
- **API permissions:** `User.Read` on Microsoft Graph (delegated) — minimum
  needed to sign in.
- **Client secret or federated identity credential** for the confidential
  client flow.

> If you previously had `apim-data-api` and `foundry-mcp-client` from earlier
> drafts, they are no longer needed in this model. You can leave them
> dormant, or delete them once nothing else references them.

---

## APIM

Only one API. APIM does **not** sit in front of any data resource.

| API | Path | Purpose | Backend |
|---|---|---|---|
| `chat` | `/chat` | Web app → APIM → Foundry agent endpoint, routed per tenant | Foundry project (one per partner) |

### `validate-jwt` block

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized">
  <!-- Templated tenant — accepts tokens from any consented tenant.
       Per-tenant allowlist is enforced by <issuers> below. -->
  <openid-config url="https://login.microsoftonline.com/organizations/v2.0/.well-known/openid-configuration" />

  <audiences>
    <audience>api://{{AGENT_HOST_WEBAPP_CLIENT_ID}}</audience>
    <audience>{{AGENT_HOST_WEBAPP_CLIENT_ID}}</audience>
  </audiences>

  <!-- TENANT ALLOWLIST. One <issuer> per accepted partner tenant.
       Without this, ANY tenant that consented could call this API. -->
  <issuers>
    <issuer>https://login.microsoftonline.com/{{YOUR_TENANT_ID}}/v2.0</issuer>
    <issuer>https://login.microsoftonline.com/{{PARTNER_TENANT_1_ID}}/v2.0</issuer>
    <!-- add more partner tenants here as they onboard -->
  </issuers>

  <required-claims>
    <claim name="scp" match="any">
      <value>access_as_user</value>
    </claim>
  </required-claims>
</validate-jwt>

<set-variable name="callerTid"
              value="@(context.Principal.Claims.GetValueOrDefault("tid",""))" />
<set-variable name="callerOid"
              value="@(context.Principal.Claims.GetValueOrDefault("oid",""))" />
```

### `/chat` policy — route by `tid`, swap auth, pass identity headers

Store the partner mapping as a JSON Named Value `PARTNER_FOUNDRY_MAP`:

```json
{
  "<partner-tenant-1-guid>": "https://eastus.api.azureml.ms/agents/v1/.../partnerB-project",
  "<partner-tenant-2-guid>": "https://eastus.api.azureml.ms/agents/v1/.../partnerC-project"
}
```

```xml
<inbound>
  <base />
  <!-- shared validate-jwt block; sets callerTid + callerOid -->

  <set-variable name="foundryUrl" value="@{
      var map = JObject.Parse((string)"{{PARTNER_FOUNDRY_MAP}}");
      var tid = (string)context.Variables["callerTid"];
      var url = (string?)map[tid];
      return string.IsNullOrEmpty(url) ? null : url;
  }" />

  <choose>
    <when condition="@(context.Variables["foundryUrl"] == null)">
      <return-response>
        <set-status code="403" reason="Forbidden" />
        <set-body>{"error":"no Foundry project mapped for this tenant"}</set-body>
      </return-response>
    </when>
  </choose>

  <set-backend-service base-url="@((string)context.Variables["foundryUrl"])" />

  <!-- APIM authenticates to Foundry as its own MI. APIM's MI must have
       Azure AI User (or equivalent) on each per-partner Foundry project. -->
  <authentication-managed-identity resource="https://ai.azure.com" />

  <!-- Pass the validated user identity to the agent as trusted headers.
       The agent treats these as the source of truth for who the user is. -->
  <set-header name="x-user-tid" exists-action="override">
    <value>@((string)context.Variables["callerTid"])</value>
  </set-header>
  <set-header name="x-user-oid" exists-action="override">
    <value>@((string)context.Variables["callerOid"])</value>
  </set-header>

  <!-- Optional per-user (or per-tenant) rate limit -->
  <rate-limit-by-key calls="60" renewal-period="60"
                     counter-key="@(
                       (string)context.Variables["callerTid"]
                       + ":" +
                       (string)context.Variables["callerOid"])" />
</inbound>
```

### Optional: log `tid` + `oid` for every call

```xml
<log-to-eventhub logger-id="apim-audit">@{
    return new JObject(
      new JProperty("tid", context.Variables["callerTid"]),
      new JProperty("oid", context.Variables["callerOid"]),
      new JProperty("api", context.Api.Name),
      new JProperty("op",  context.Operation.Name),
      new JProperty("ts",  DateTime.UtcNow)
    ).ToString();
}</log-to-eventhub>
```

---

## Inside the agent — per-user separation using validated headers

The agent runs inside the partner's dedicated Foundry project. It receives
two trusted headers from APIM (`x-user-tid`, `x-user-oid`) and uses its own
managed identity to call the partner's data slice.

> **Why these headers are trusted:** APIM is the only public ingress to
> Foundry. The Foundry project is private (VNET / private endpoint), so
> nothing else can put headers on inbound requests. APIM only sets the
> headers from validated JWT claims — never from anything the client sent.
> The trust boundary is "Foundry only accepts traffic from APIM," enforced by
> network controls.

### Writing a conversation tagged with caller identity (Cosmos)

```python
# Inside an MCP tool or agent code
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

# Pulls the Foundry project's managed identity automatically
cred = DefaultAzureCredential()
cosmos = CosmosClient(COSMOS_URL, credential=cred)
db = cosmos.get_database_client(PARTNER_DB_NAME)            # partner-B-conv
container = db.get_container_client(CONVERSATIONS_CONTAINER)

def save_message(headers, conversation_id, message):
    tid = headers["x-user-tid"]    # from APIM, trusted
    oid = headers["x-user-oid"]    # from APIM, trusted
    container.upsert_item({
        "id": f"{oid}:{conversation_id}",
        "tid": tid,                # always overwrite from header — never trust client body
        "oid": oid,
        "conversationId": conversation_id,
        "message": message,
    })
```

### Reading conversations for the caller only

```python
def list_user_conversations(headers):
    oid = headers["x-user-oid"]
    return list(container.query_items(
        query="SELECT * FROM c WHERE c.oid = @oid",
        parameters=[{"name": "@oid", "value": oid}],
        enable_cross_partition_query=False,
        partition_key=oid,           # if container partitions on /oid
    ))
```

> The container itself is *partner-scoped* (Foundry-B's MI only has RBAC on
> "partner-B-conv"), so cross-partner leakage is impossible regardless of
> what the query says. The `oid` filter is the *intra-partner* user
> separation.

### What to never do

- ❌ Read `tid`/`oid` from the request body (clients can lie)
- ❌ Skip the `oid` filter on the assumption that "the container is for one
  partner anyway" — that protects across partners but not across users in
  the same partner
- ❌ Use `DefaultAzureCredential` configured to fall back to user/dev
  credentials in production (could pick up an unintended identity)

---

## Onboarding a new partner tenant

| # | Owner | Action |
|---|---|---|
| 1 | You | Provision a **dedicated Foundry project** for the partner in Tenant A. Enable system-assigned managed identity (or assign a user-assigned MI dedicated to this partner). |
| 2 | You | Provision the partner's **data slice** in Tenant A — e.g., a Cosmos DB named `partnerB-conv`, an AI Search index `partnerB-kb`, a Storage container `partnerB-files`. |
| 3 | You | Grant the partner Foundry project's MI Azure RBAC on **only those resources** (e.g., `Cosmos DB Built-in Data Contributor` scoped to `partnerB-conv`, `Search Index Data Contributor` on `partnerB-kb`, etc.). Do **not** grant access to other partners' resources. |
| 4 | You | Grant **APIM's** MI `Azure AI User` (or equivalent) on the new Foundry project so APIM can invoke the agent endpoint. |
| 5 | You | Configure the agent and MCP tools to point at the partner's data slice (database name, index name, etc.) — typically via Foundry project environment variables. |
| 6 | You | Add a row to the `PARTNER_FOUNDRY_MAP` Named Value: `"<partnerTenantId>": "<foundry agent endpoint>"`. |
| 7 | You | Add `https://login.microsoftonline.com/<partnerTenantId>/v2.0` to the `<issuers>` allowlist in the `validate-jwt` block. Deploy APIM. |
| 8 | You | Send the partner admin **one** consent URL: `https://login.microsoftonline.com/<partnerTenantId>/adminconsent?client_id=<agent-host-webapp-id>` |
| 9 | Partner | Open URL → review permissions → grant consent. SP for `agent-host-webapp` materializes in Tenant B. |
| 10 | Both | End-to-end smoke test (see *Sanity test* below). |

After onboarding, the partner admin has **no required ongoing actions**.

### Off-boarding a tenant — kill switches

Independent levers, any one of which cuts off the partner:

- **Remove the Foundry project's MI role assignments** on the partner's data
  slice → agent code fails 403 immediately on the next call.
- **Remove their `<issuer>`** from `validate-jwt` → APIM rejects all their
  tokens within seconds of policy deploy.
- **Remove their entry from `PARTNER_FOUNDRY_MAP`** → APIM `/chat` returns
  403 even if the token validates.
- **Delete the partner's Foundry project** → no agent endpoint to call.
- **Delete the partner's data slice** → physically removes their data.
- **Ask their admin to remove the `agent-host-webapp` enterprise app** from
  Tenant B → no new tokens issued.

No cleanup is needed in your directory because no external user records
exist there.

---

## Sanity test (per partner tenant)

A partner-tenant user with a successful end-to-end call should produce a USER
token (decode at https://jwt.ms) with:

- `aud` = `api://<agent-host-webapp>` or the bare GUID
- `iss` = `https://login.microsoftonline.com/<partnerTenantId>/v2.0`
- `tid` = `<partnerTenantId>` (**not** your tenant)
- `oid` = an `oid` from the partner tenant
- `scp` includes `access_as_user`

End-to-end flow check:

1. Partner user signs into the multi-tenant web app → ID token + access
   token issued by Tenant B
2. Browser POSTs to APIM `/chat` with the access token → `validate-jwt`
   passes (issuer in allowlist, audience matches `agent-host-webapp`)
3. APIM looks up `tid` in `PARTNER_FOUNDRY_MAP` → routes to partner's
   Foundry project
4. APIM authenticates to Foundry as its own MI; passes `x-user-tid` and
   `x-user-oid` headers
5. Agent runs; reads headers; calls Cosmos / Search using **its own** MI
6. Cosmos / Search authorize because Foundry-B's MI has RBAC on
   "partner-B-conv" / "partner-B-kb"
7. Agent applies `WHERE c.oid = @callerOid` filter → returns only the
   user's data
8. Response returned to web app → user

If `validate-jwt` fails with `IDX10205` (*Issuer validation failed*), you
forgot to add the partner tenant to `<issuers>`.

If `/chat` returns 403, the partner is missing from `PARTNER_FOUNDRY_MAP`.

If the agent fails with 401/403 calling the data slice, the per-partner
Foundry MI is missing RBAC. Check the role assignments on the partner's
Cosmos DB / Search index / Storage container.

---

## Common pitfalls (multi-tenant specific)

- **Forgetting the `<issuers>` allowlist.** Using `organizations/v2.0` for
  metadata while leaving `<issuers>` empty means *any* tenant in the world
  that consents will pass `validate-jwt`. Always pair the templated metadata
  URL with an explicit allowlist.
- **App not actually multi-tenant.** Partner users get `AADSTS50020` or
  `AADSTS650057` at sign-in. Verify the manifest has
  `"signInAudience": "AzureADMultipleOrgs"`.
- **Sharing a managed identity across partners.** Defeats the cross-partner
  isolation guarantee. Each partner's Foundry project must have its own MI,
  scoped to only that partner's data slice.
- **Granting the per-partner Foundry MI access to a "shared" resource.**
  If you must, scope it tightly (e.g., a specific blob prefix, not the whole
  account) and treat that resource as part of every partner's blast radius.
- **Trusting client-supplied `tid`/`oid` in the request body.** Always read
  these from the validated headers (which originate from JWT claims), and
  *overwrite* anything the client sent before persisting.
- **Skipping the `oid` filter on reads** because "the container is partner-
  scoped." That protects across partners but lets one user in Partner B read
  another Partner B user's data. Both layers matter.
- **Forgetting to add the partner to `PARTNER_FOUNDRY_MAP`.** `validate-jwt`
  passes (issuer is in the allowlist) but `/chat` returns 403.
- **APIM managed identity not granted Foundry RBAC.** `/chat` validates the
  user token, picks the right Foundry URL, then fails calling Foundry with
  401/403. Confirm APIM's MI has `Azure AI User` on every per-partner Foundry
  project.
- **Using `DefaultAzureCredential` with broad fallbacks in agent code.** In
  Foundry, prefer the explicit Foundry-managed-identity credential to avoid
  silently picking up another identity (e.g., a developer's az login during
  local testing).
- **Wishing OBO was here.** If a use case appears that needs the user's own
  M365 data (their mail, their files), this pattern can't satisfy it. Add a
  separate `/graph` API following the OBO docs — that API uses
  `apim-obo-middletier` (with secret + Graph permissions) alongside the
  no-OBO pieces here. The two patterns coexist in the same APIM instance.

---

## See also

- [`README.md`](./README.md) — auth flow background, sequence diagram,
  Microsoft Entra Agent ID references
- [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) — single-tenant
  OBO to Microsoft Graph (different pattern; use when agent needs user's own
  M365 data)
- [`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md) —
  single-tenant OBO to SharePoint REST
- Microsoft Learn — [Tenancy in Microsoft Entra ID](https://learn.microsoft.com/entra/identity-platform/single-and-multi-tenant-apps)
- Microsoft Learn — [How to convert an app to be multi-tenant](https://learn.microsoft.com/entra/identity-platform/howto-convert-app-to-be-multi-tenant)
- Microsoft Learn — [Managed identities in APIM](https://learn.microsoft.com/azure/api-management/api-management-howto-use-managed-service-identity)
- Microsoft Learn — [Azure AI Foundry managed identity](https://learn.microsoft.com/azure/ai-foundry/concepts/managed-identity)
