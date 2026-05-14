# OBO Flow: Multi-Tenant External Access (No B2B Guests)

This document extends the single-tenant pattern in
[`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) /
[`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md) to allow
users in **other Entra tenants** to call your APIM-fronted API — without
inviting them as B2B guests in your tenant.

> **Hard constraint of this design:** no external user account is ever created
> in your tenant. External users sign in entirely against their own home tenant.
> All authorization decisions happen on your side using claims (`tid`, `oid`)
> from the inbound user token.

---

## Why a separate document?

The single-tenant docs assume one Entra tenant owns everything: the middle-tier
app, the Foundry project, and the users. In the external-access model:

- **Your tenant (Tenant A)** owns: `apim-obo-middletier`, APIM, the downstream
  API integration, the multi-tenant web app (`agent-host-webapp`), the
  `foundry-mcp-client` app registration, **and a dedicated Foundry project per
  partner** that you manage on the partner's behalf.
- **Each partner tenant (Tenant B, C, …)** owns only its own users. No Azure
  resources, no app registrations, no Foundry project on the partner side.
- **Tokens crossing the boundary:** USER tokens issued by the partner tenant
  (for the multi-tenant web app and for the MCP client) flowing into resources
  hosted in Tenant A.

This changes the app-registration audience, the `validate-jwt` policy, and the
authorization model. Everything else (OBO mechanics, Graph/SP permissions,
caching) is identical to the single-tenant doc.

---

## Requirements

### Inherits from the single-tenant doc

All requirements from `obo-foundry-apim-graph.md` (or `-sharepoint.md`) still
apply. This document adds requirements **on top**.

### Hosting model

The partner does **not** stand up their own Azure resources. The pattern is:

- **Your tenant (Tenant A)** hosts a **dedicated Foundry project per partner**
  (one project per partner, so agents, prompts, tools, and data are isolated).
  You manage the project on the partner's behalf — agent configuration, MCP
  tool wiring, deployments, observability are all done by your team.
- **The partner's end users (in Tenant B)** never touch the Azure portal or
  Foundry Studio. They reach the agent through a **multi-tenant web app**
  hosted in Tenant A that signs them in against *their own* tenant.
- **Partner admins / agent builders have no direct access** to the Foundry
  project. This is what keeps the design fully compatible with the
  no-B2B-guests rule (Azure RBAC on the Foundry resource would require a
  cross-tenant principal in your tenant, which we are not creating).

If a partner needs to self-serve agent development, this design does not fit —
that scenario requires either guest accounts, B2B direct connect, or putting
the Foundry project in the partner's own tenant. None of those are documented
here.

### External-access requirements

| # | Requirement | Owner |
|---|---|---|
| 1 | A **dedicated Foundry project per partner** is provisioned in your tenant (Tenant A) and managed by you on the partner's behalf | You |
| 2 | Partner end users access the agent via a **multi-tenant web app** in Tenant A, never via the Azure portal or Foundry Studio | You |
| 3 | Partner admins and agent builders have **no direct Foundry/Azure access** in your tenant | You + Partner agreement |
| 4 | `apim-obo-middletier` must be multi-tenant (`signInAudience = AzureADMultipleOrgs`) | You |
| 5 | The web app's app registration (`agent-host-webapp`) must be multi-tenant | You |
| 6 | The Foundry MCP client app registration (`foundry-mcp-client`) must be multi-tenant | You |
| 7 | Delegated Graph (or SharePoint) permissions on the middle-tier app must be admin-consented in your tenant | You |
| 8 | **One-time admin consent in each partner tenant** for `agent-host-webapp` and `foundry-mcp-client` (consenting to `foundry-mcp-client` cascades a service principal for `apim-obo-middletier` into the partner tenant) | Partner admin |
| 9 | APIM `validate-jwt` policy enforces an explicit issuer allowlist (one entry per accepted partner tenant) | You |
| 10 | Per-user authorization is enforced on your side using `tid` + `oid` claims (tenant allowlist and/or oid allowlist) | You |

> **Requirement 8 is non-negotiable in Entra.** There is no OAuth flow that
> issues a user token for an app in tenant B without an admin in tenant B
> consenting at least once. Microsoft Graph itself works the same way. After
> the one-time consent (two clicks — one per app), the partner admin has no
> further required tasks.

### Out of scope by design

- ❌ B2B guest invitations (no guest user objects in your tenant)
- ❌ Hosting a single Foundry project that external users sign into directly
  (would require Azure RBAC on the Foundry resource → guest accounts)
- ❌ Putting external users in **your** Entra security groups
- ❌ Assigning external users to **your** app roles (assignment happens in
  *their* tenant; if you don't trust their admin to do it, don't use app roles)

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
│                         │                  │                  ▼                               │
│   ┌────────────────┐    │                  │   ┌────────────────────────────────┐             │
│   │ Entra (B)      │    │                  │   │ APIM  /chat  API               │             │
│   │ - issues USER  │    │                  │   │ - validate-jwt (issuer ∈ B)    │             │
│   │   tokens for   │    │                  │   │ - lookup tid → Foundry URL     │             │
│   │   apps in (A)  │    │                  │   │ - swap auth → APIM MI token    │             │
│   │ - SPs created  │    │                  │   │ - x-user-tid / x-user-oid hdrs │             │
│   │   on consent:  │    │                  │   │ - rate-limit by tid            │             │
│   │   • agent-host │    │                  │   └──────────────┬─────────────────┘             │
│   │   • foundry-   │    │                  │                  │                               │
│   │     mcp-client │◄───┼──auth-code via───┤                  ▼                               │
│   │   • apim-obo-  │    │  APIM Cred Mgr   │   ┌────────────────────────────────┐             │
│   │     middletier │    │  (foundry-mcp-   │   │ Foundry project (per partner)  │             │
│   │     (cascaded) │    │   client)        │   │ - dedicated agent              │             │
│   └────────────────┘    │                  │   │ - MCP tool: OAuth Identity     │             │
│                         │  2. MCP tool     │   │   Passthrough                  │             │
│                         │     auth-code    │   └──────────────┬─────────────────┘             │
│                         │                  │                  │ Bearer USER token             │
│                         │                  │                  │ for middle-tier audience      │
│                         │  3. USER token   │                  ▼                               │
│                         │  (aud=middletier,│   ┌────────────────────────────────┐             │
│                         │   tid=B, oid=B)  │   │ APIM  /graph  API              │             │
│                         │ ────────────────►│   │ - same validate-jwt block      │             │
│                         │                  │   │ - OBO send-request to AAD as   │             │
│                         │                  │   │   apim-obo-middletier          │             │
│                         │                  │   └──────────────┬─────────────────┘             │
│                         │                  │                  │ Graph token (for B            │
│                         │                  │                  │ user; aud=Graph)              │
│                         │                  │                  ▼                               │
│                         │                  │   Microsoft Graph / SharePoint REST              │
│                         │                  │   → returns PARTNER-TENANT data                  │
└─────────────────────────┘                  └──────────────────────────────────────────────────┘
```

Key observations:

- **All Azure resources and all three app registrations live in Tenant A.**
  Tenant B contributes only its users.
- **Three multi-tenant app registrations** are involved, all owned by you:
  - `agent-host-webapp` — the web app the user signs into (phase 1)
  - `foundry-mcp-client` — the OAuth client the MCP tool uses (phase 2)
  - `apim-obo-middletier` — the OBO middle tier (audience of the USER token)
- **Two APIM APIs in the same instance**, both protected by the same
  `validate-jwt` block (same audiences, same issuer allowlist):
  - `/chat` — front of every Foundry project; routes by `tid` claim
  - `/graph` (or `/sharepoint`) — performs the OBO exchange to the downstream API
- **Per-partner Foundry project isolation.** The `/chat` API uses a Named
  Value mapping `tid → Foundry agent URL` to send each partner's traffic to
  their dedicated project. APIM authenticates to Foundry using its **managed
  identity**; the user identity travels in `x-user-tid` / `x-user-oid`
  headers so the agent can act on the user's behalf.
- **The partner admin's only required action** is one-time admin consent in
  Tenant B for `agent-host-webapp` and `foundry-mcp-client`. Consenting to
  `foundry-mcp-client` cascades a service principal for `apim-obo-middletier`
  into Tenant B because `foundry-mcp-client` declares a delegated permission on
  it. After that, no further partner-side action is required.
- **USER tokens carry partner-tenant identity**: `iss = https://login.microsoftonline.com/<tenantB>/v2.0`,
  `tid = <tenantB>`, `oid = <user oid in tenant B>`. APIM enforces a tenant
  allowlist via `<issuers>` in `validate-jwt`.
- **OBO returns partner-tenant Graph data.** The middle-tier app exchanges the
  USER token for a Graph token whose subject is the partner-tenant user, so
  Graph returns *their* tenant's data — never yours.
- **Partner end users never sign into the Azure portal or Foundry Studio.**
  Their entire experience is the multi-tenant web app. Foundry project
  management is performed by you, signed in with your own Tenant A credentials.

---

## App registration changes

Three multi-tenant app registrations live in your tenant. The middle-tier
already exists from the single-tenant doc; the other two are new for this
pattern.

### 1. `apim-obo-middletier` — make it multi-tenant

In Entra → App registrations → `apim-obo-middletier` → **Authentication**:

- **Supported account types:** *Accounts in any organizational directory
  (Any Microsoft Entra ID tenant — Multitenant)*

This sets `signInAudience = AzureADMultipleOrgs` in the manifest.

> **Do not** include "personal Microsoft accounts" unless you have a specific
> consumer-account use case — adds attack surface for no benefit here.

### 2. `agent-host-webapp` — multi-tenant web app

This is the only thing partner end users sign into. Lives in Tenant A; consented
in each Tenant B.

- **Supported account types:** Multitenant
- **Redirect URI:** the web app's sign-in callback (e.g.,
  `https://contoso-agent-host.azurewebsites.net/.auth/login/aad/callback`
  if you use App Service Easy Auth, or your own MSAL callback)
- **API permissions:** `User.Read` on Microsoft Graph (delegated) — the bare
  minimum to sign in. The web app does *not* need any permission on
  `apim-obo-middletier`; the OBO flow is initiated later by the MCP tool, not
  by the web app.
- **Client secret or federated identity credential** for confidential client
  flow.

### 3. `foundry-mcp-client` — multi-tenant MCP OAuth client

This is the OAuth client APIM Credential Manager uses to acquire USER tokens
for the middle-tier audience.

- **Supported account types:** Multitenant
- **Redirect URI:** the APIM Credential Manager broker URL —
  `https://global.consent.azure-apim.net/redirect/<your-apim-credmgr-guid>-<connection-name>`
- **API permissions:** delegated `access_as_user` on `apim-obo-middletier`,
  admin-consented in your tenant
- **Client secret or FIC**

> Because all three app regs live in **your** tenant, you do not ship any
> client secret to the partner. The partner admin's only action is consenting
> via URL.

---

## APIM APIs

You will operate **two APIs** in the same APIM instance:

| API | Path | Purpose | Backend |
|---|---|---|---|
| `chat` | `/chat` | Web app → APIM → Foundry agent endpoint, routed per tenant | Foundry project (one per partner) |
| `graph` (or `sharepoint`) | `/graph` | Foundry MCP tool → APIM → downstream API (OBO exchange) | Microsoft Graph / SharePoint REST |

Both APIs share the **same** `validate-jwt` block (same audiences, same issuer
allowlist) so a single tenant allowlist controls both surfaces. Removing a
partner from the allowlist immediately blocks them on both APIs.

### Shared `validate-jwt` (used by both APIs)

This replaces the single-tenant `validate-jwt`. It accepts USER tokens from
any tenant in the allowlist, in either v1 or v2 format.

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized">
  <!-- Templated tenant — accepts tokens from any consented tenant.
       Per-tenant allowlist is enforced by <issuers> below. -->
  <openid-config url="https://login.microsoftonline.com/organizations/v2.0/.well-known/openid-configuration" />

  <audiences>
    <audience>api://{{APIM_OBO_MIDDLETIER_CLIENT_ID}}</audience>
    <audience>{{APIM_OBO_MIDDLETIER_CLIENT_ID}}</audience>
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

### `chat` API policy — route to per-partner Foundry project

The web app POSTs the user's chat request here with the USER token. APIM picks
the correct Foundry project based on `tid` and forwards using its own
**managed identity** (Foundry RBAC in Tenant A grants `Azure AI User` to
APIM's MI on each partner project).

Store the partner mapping as a JSON Named Value
`PARTNER_FOUNDRY_MAP`, e.g.:

```json
{
  "<partner-tenant-1-guid>": "https://eastus.api.azureml.ms/agents/v1/.../partnerB-project",
  "<partner-tenant-2-guid>": "https://eastus.api.azureml.ms/agents/v1/.../partnerC-project"
}
```

Policy:

```xml
<inbound>
  <base />

  <!-- Includes the shared validate-jwt block above; sets callerTid / callerOid -->

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

  <!-- Replace inbound user auth with APIM's managed-identity token for Foundry.
       The user identity continues to flow as headers (below) so the agent and
       MCP tool can act on the user's behalf downstream. -->
  <authentication-managed-identity resource="https://ai.azure.com" />

  <!-- Pass user identity to the agent runtime so the MCP tool's OAuth
       Identity Passthrough connection uses the right user. -->
  <set-header name="x-user-tid" exists-action="override">
    <value>@((string)context.Variables["callerTid"])</value>
  </set-header>
  <set-header name="x-user-oid" exists-action="override">
    <value>@((string)context.Variables["callerOid"])</value>
  </set-header>

  <!-- Per-partner throttling, keyed on tid. -->
  <rate-limit-by-key calls="60" renewal-period="60"
                     counter-key="@((string)context.Variables["callerTid"])" />
</inbound>
```

> The exact `resource` URI for `<authentication-managed-identity>` depends on
> which Foundry agent endpoint you're hitting (Azure ML data plane vs. the
> newer Foundry control-plane endpoints). Verify against the Foundry SDK
> reference for the agent endpoint you target.

### `graph` API policy — unchanged structure, shared `validate-jwt`

The Graph (or SharePoint) API keeps the OBO `send-request` from the
single-tenant doc. The only change: it uses the **same** shared `validate-jwt`
block above so the tenant allowlist is enforced identically.

### Optional: enforce a per-user `oid` allowlist (either or both APIs)

If you want fine-grained control over *which* users in an allowed tenant may
call — without trusting partner admins to manage anything — add this after
`validate-jwt`. Store the allowed `oid`s in a Named Value (CSV) or, for many,
in Cosmos / blob / Key Vault.

```xml
<choose>
  <when condition="@{
      var allowed = ((string)"{{ALLOWED_OIDS_CSV}}").Split(',');
      return !allowed.Contains((string)context.Variables["callerOid"]);
  }">
    <return-response>
      <set-status code="403" reason="Forbidden" />
      <set-body>{"error":"user not authorized"}</set-body>
    </return-response>
  </when>
</choose>
```

### Optional: log `tid` + `oid` for every call

Highly recommended. Audit record per call without ever knowing the user's name.

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

## Onboarding a new partner tenant

A repeatable runbook. Steps marked **You** are on the API publisher side;
**Partner** steps are done by an admin in the partner tenant. All "You" steps
happen in **your** tenant (Tenant A).

| # | Owner | Action |
|---|---|---|
| 1 | You | Provision a **dedicated Foundry project** for the partner in Tenant A. Configure the agent and the MCP tool (OAuth Identity Passthrough → `foundry-mcp-client` connection). |
| 2 | You | Grant APIM's managed identity `Azure AI User` (or equivalent) RBAC on the new Foundry project so APIM can invoke the agent. |
| 3 | You | Add a row to the `PARTNER_FOUNDRY_MAP` Named Value: `"<partnerTenantId>": "<foundry agent endpoint>"`. |
| 4 | You | Add `https://login.microsoftonline.com/<partnerTenantId>/v2.0` to the `<issuers>` allowlist in the shared `validate-jwt` block. Deploy both APIM APIs. |
| 5 | You | Send the partner admin **two** consent URLs: <br/>• `https://login.microsoftonline.com/<partnerTenantId>/adminconsent?client_id=<agent-host-webapp-id>` <br/>• `https://login.microsoftonline.com/<partnerTenantId>/adminconsent?client_id=<foundry-mcp-client-id>` |
| 6 | Partner | Open both URLs → review permissions → grant consent. SPs for `agent-host-webapp` and `foundry-mcp-client` materialize in their tenant. The `foundry-mcp-client` consent **cascades** an SP for `apim-obo-middletier` because of the declared delegated permission. |
| 7 | You | (If using oid allowlist) Receive list of `oid` GUIDs from partner; add to APIM Named Value. |
| 8 | Both | End-to-end smoke test (see *Sanity test* below). |

After onboarding, the partner admin has **no required ongoing actions**. User
add/remove inside their tenant flows through their own IdP normally; you only
update the oid allowlist (if used) or rotate the Foundry mapping.

### Off-boarding a tenant — kill switch

Three independent levers, any one of which cuts off the partner:

- **Remove their `<issuer>`** from the shared `validate-jwt` block →
  both `/chat` and `/graph` reject their tokens within seconds of policy deploy.
- **Remove their entry from `PARTNER_FOUNDRY_MAP`** → `/chat` returns 403 even
  if the token is otherwise valid.
- **Ask their admin to remove the enterprise apps** (`agent-host-webapp`,
  `foundry-mcp-client`) from their tenant → no new tokens can be issued at all.

You can also disable or delete the partner's dedicated Foundry project to
reclaim resources. No cleanup is needed in your directory because no external
user records exist there.

---

## Authorization decision matrix

Pick the model(s) that match your trust posture:

| Model | Who decides who-can-call | Partner-side work after consent | Your-side work | When to use |
|---|---|---|---|---|
| **Tenant allowlist only** | You (by tenant) | None | Maintain `<issuers>` list | Coarsest. "Anyone in this tenant who consented can call." |
| **Tenant allowlist + oid allowlist** | You (by user) | Send you `oid`s out-of-band | Maintain `<issuers>` + Named Value/store of oids | Default recommendation when you don't trust partner admin to manage assignments. |
| **App roles on middle-tier** | Partner admin (assigns users to roles in their tenant; role flows in `roles` claim) | Assign users/groups to roles | Define roles in app manifest, enforce via `required-claims` | Only when you trust the partner admin to manage assignments correctly. |
| **Per-call entitlement service** | Your authz API/DB | None | Build & operate an authz service | Most flexible; needed when entitlements depend on more than identity (e.g., resource ownership). |

---

## Sanity test (per partner tenant)

A partner-tenant user with a successful end-to-end call should produce a USER
token (decode at https://jwt.ms) with:

- `aud` = `api://<your-middletier>` or the bare GUID
- `iss` = `https://login.microsoftonline.com/<partnerTenantId>/v2.0`
- `tid` = `<partnerTenantId>` (**not** your tenant)
- `oid` = an `oid` from the partner tenant
- `scp` includes `access_as_user`

And the APIM trace should show:

- `validate-jwt` passes (issuer matches the allowlist entry)
- `send-request` to AAD's `/oauth2/v2.0/token` returns 200
- The returned Graph/SharePoint token has `tid` = partner tenant
- Downstream call returns 200 with **partner-tenant** data

If `aud` looks right but `validate-jwt` fails with `IDX10205`
(*Issuer validation failed*), you forgot to add the partner tenant to
`<issuers>`. That's the most common onboarding miss.

---

## Common pitfalls (multi-tenant specific)

These are *in addition* to the single-tenant pitfalls in the Graph/SharePoint docs.

- **Forgetting the `<issuers>` allowlist.** Using `organizations/v2.0` for
  metadata while leaving `<issuers>` empty means *any* tenant that consents
  will pass `validate-jwt`. Always pair the templated metadata URL with an
  explicit allowlist.
- **App not actually multi-tenant.** If you forgot to flip the supported
  account types, partner-tenant users will get `AADSTS50020` or
  `AADSTS650057` at sign-in. Verify the manifest has
  `"signInAudience": "AzureADMultipleOrgs"`.
- **Trying to assign external users to your groups.** They don't exist in
  your tenant — there's nothing to assign. Use `tid`/`oid` claim checks or
  app roles instead.
- **Forgetting to add the partner to `PARTNER_FOUNDRY_MAP`.** `validate-jwt`
  passes (issuer is in the allowlist) but `/chat` returns 403 because the
  routing lookup fails. Onboarding step 3 covers this — verify the Named Value
  was updated and deployed.
- **APIM managed identity not granted Foundry RBAC.** `/chat` validates the
  user token, picks the right Foundry URL, then fails calling Foundry with
  401/403. Confirm APIM's MI has `Azure AI User` (or equivalent) on every
  per-partner Foundry project.
- **Assuming Graph data is yours.** OBO returns Graph data scoped to the
  USER's tenant — if a partner-tenant user calls `/me`, they get their *own*
  profile from *their* tenant, not anything from your directory. This is
  almost always desired but worth confirming with consumers.
- **Ignoring conditional access in the partner tenant.** Their CA policies
  apply to their users when those users acquire the USER token. If a partner
  tenant requires MFA / compliant device for your app, that's enforced at
  *their* sign-in — your APIM never sees it. Not your problem to fix, but a
  common source of "it doesn't work for some users" reports.
- **Assuming you can see who the user is by name.** You only get GUIDs
  (`tid`, `oid`) and whatever optional claims their tenant chose to release.
  Plan logging and support workflows around GUIDs, not names/emails.

---

## See also

- [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) — single-tenant
  Graph version (this doc inherits everything from it)
- [`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md) —
  single-tenant SharePoint version
- [`README.md`](./README.md) — auth flow background, sequence diagram,
  Microsoft Entra Agent ID references
- Microsoft Learn — [Tenancy in Microsoft Entra ID](https://learn.microsoft.com/entra/identity-platform/single-and-multi-tenant-apps)
- Microsoft Learn — [How to convert an app to be multi-tenant](https://learn.microsoft.com/entra/identity-platform/howto-convert-app-to-be-multi-tenant)
