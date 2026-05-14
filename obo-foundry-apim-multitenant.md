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

- **Your tenant** owns: `apim-obo-middletier`, APIM, the downstream API integration
- **Each partner tenant** owns: their own Foundry project, their own
  `foundry-mcp-client` app registration, their own users
- **Tokens crossing the boundary:** only USER tokens (issued by the partner
  tenant) hitting your APIM

This changes the app-registration audience, the `validate-jwt` policy, and the
authorization model. Everything else (OBO mechanics, Graph/SP permissions,
caching) is identical to the single-tenant doc.

---

## Requirements

### Inherits from the single-tenant doc

All requirements from `obo-foundry-apim-graph.md` (or `-sharepoint.md`) still
apply. This document adds requirements **on top**.

### External-access requirements

| # | Requirement | Owner |
|---|---|---|
| 1 | `apim-obo-middletier` must be multi-tenant (`signInAudience = AzureADMultipleOrgs`) | You |
| 2 | Delegated Graph (or SharePoint) permissions on the middle-tier app must be admin-consented in your tenant | You |
| 3 | **One-time admin consent in each partner tenant** for `apim-obo-middletier` | Partner admin |
| 4 | APIM `validate-jwt` policy enforces an explicit issuer allowlist (one entry per accepted partner tenant) | You |
| 5 | Per-user authorization is enforced on your side using `tid` + `oid` claims (tenant allowlist and/or oid allowlist) | You |
| 6 | Each partner tenant runs its own Foundry project and its own `foundry-mcp-client` app registration | Partner |

> **Requirement 3 is non-negotiable in Entra.** There is no OAuth flow that
> issues a user token for an app in tenant B without an admin in tenant B
> consenting at least once. Microsoft Graph itself works the same way. After
> the one-time consent, the partner admin has no further required tasks.

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
┌─────────────────────────────┐         ┌─────────────────────────────┐
│   Partner tenant (B)        │         │   Your tenant (A)           │
│                             │         │                             │
│   ┌───────────────────┐     │         │   ┌─────────────────────┐   │
│   │ Foundry project   │     │         │   │ apim-obo-middletier │   │
│   │ (their Azure sub) │     │         │   │ (multi-tenant)      │   │
│   └────────┬──────────┘     │         │   └──────────┬──────────┘   │
│            │                │         │              │              │
│   ┌────────▼──────────┐     │         │   ┌──────────▼──────────┐   │
│   │ foundry-mcp-client│     │         │   │ APIM                │   │
│   │ (their app reg)   │     │  USER   │   │ (validate-jwt +     │   │
│   └────────┬──────────┘     │  token  │   │  OBO send-request)  │   │
│            │                │ ──────► │   └──────────┬──────────┘   │
│            │ user signs in  │         │              │              │
│            │ → user token   │         │              │ Graph token  │
│            │   for          │         │              │ (OBO result) │
│            │   middle-tier  │         │              ▼              │
│            ▼                │         │   Microsoft Graph /         │
│   Entra (tenant B issues    │         │   SharePoint REST           │
│   tokens; SP for            │         │                             │
│   middletier exists here    │         │                             │
│   after one-time consent)   │         │                             │
└─────────────────────────────┘         └─────────────────────────────┘
```

Key observations:

- The partner's user signs into the partner's Foundry → APIM Credential Manager
  brokers an auth-code flow against **the partner tenant's Entra**.
- The partner tenant issues a USER token with `aud = api://<your-middletier>`,
  `iss = https://login.microsoftonline.com/<partnerTenantId>/v2.0`,
  `tid = <partnerTenantId>`.
- That token is sent to **your** APIM.
- Your middle-tier app uses its own credential to perform the OBO exchange. AAD
  honors the request because the middle-tier SP exists in the partner tenant
  (consented in step 3 above).
- The resulting Graph token is for the **partner-tenant** user → Graph returns
  **partner-tenant** data. Your tenant's data is never exposed.

---

## App registration changes

Apply these changes to the existing `apim-obo-middletier` app reg from the
single-tenant doc.

### Make the middle-tier multi-tenant

In Entra → App registrations → `apim-obo-middletier` → **Authentication**:

- **Supported account types:** *Accounts in any organizational directory
  (Any Microsoft Entra ID tenant — Multitenant)*

This sets `signInAudience = AzureADMultipleOrgs` in the manifest.

> **Do not** include "personal Microsoft accounts" unless you have a specific
> consumer-account use case — adds attack surface for no benefit here.

### `foundry-mcp-client`

You do **not** publish a multi-tenant `foundry-mcp-client`. Each partner
tenant creates its own client app reg in their tenant, pointing at your
multi-tenant middle-tier as the API resource. This avoids sharing a client
secret across tenant boundaries.

(Alternative: a shared multi-tenant client is possible but requires shipping a
secret to each partner — generally not recommended.)

---

## APIM policy update

Replace the `validate-jwt` block from the single-tenant policy with an explicit
multi-tenant version. Every change is highlighted in comments.

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized">
  <!-- Templated tenant — accepts tokens from any consented tenant.
       Per-tenant allowlist is enforced by <issuers> below. -->
  <openid-config url="https://login.microsoftonline.com/organizations/v2.0/.well-known/openid-configuration" />

  <audiences>
    <!-- v1 audience format -->
    <audience>api://{{APIM_OBO_MIDDLETIER_CLIENT_ID}}</audience>
    <!-- v2 audience format (bare GUID) -->
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
```

### Optional: enforce a per-user `oid` allowlist

If you want fine-grained control over *which* users in an allowed tenant may
call the API — without trusting partner admins to manage app role assignments —
add a check after `validate-jwt`. Store the allowed `oid`s in a Named Value
(comma-separated) or, for more than a handful, in Cosmos / blob / Key Vault.

```xml
<set-variable name="callerOid"
              value="@(context.Principal.Claims.GetValueOrDefault("oid",""))" />
<set-variable name="callerTid"
              value="@(context.Principal.Claims.GetValueOrDefault("tid",""))" />

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

Highly recommended in the multi-tenant model — gives you a per-call audit
record without ever needing to know the user's name.

```xml
<log-to-eventhub logger-id="apim-audit">@{
    return new JObject(
      new JProperty("tid", context.Variables["callerTid"]),
      new JProperty("oid", context.Variables["callerOid"]),
      new JProperty("op", context.Operation.Name),
      new JProperty("ts", DateTime.UtcNow)
    ).ToString();
}</log-to-eventhub>
```

---

## Onboarding a new partner tenant

A repeatable runbook. Steps marked **You** are on the API publisher side;
**Partner** steps are done by an admin in the partner tenant.

| # | Owner | Action |
|---|---|---|
| 1 | You | Send the partner admin the consent URL: `https://login.microsoftonline.com/<partnerTenantId>/adminconsent?client_id=<middletier-app-id>` |
| 2 | Partner | Open the URL → review permissions → grant consent. SP for `apim-obo-middletier` materializes in their tenant. |
| 3 | You | Add `https://login.microsoftonline.com/<partnerTenantId>/v2.0` to the `<issuers>` allowlist in your APIM policy and deploy. |
| 4 | Partner | Create their own `foundry-mcp-client` app reg in their tenant. Add `access_as_user` permission on `apim-obo-middletier` (it will appear in the API picker after step 2). Admin-consent it. |
| 5 | Partner | Add APIM Credential Manager redirect URI: `https://global.consent.azure-apim.net/redirect/<your-apim-credmgr-guid>-<connection-name>`. (You give them the URL.) |
| 6 | Partner | Build the Foundry agent + MCP tool pointing at your APIM URL using their client ID + secret. |
| 7 | You | (If using oid allowlist) Receive list of `oid` GUIDs from partner; add to APIM Named Value. |
| 8 | Both | End-to-end smoke test (see *Sanity test* below). |

After onboarding, the partner admin has **no required ongoing actions**. User
add/remove inside their tenant flows through their own IdP normally; you only
update the oid allowlist if you're using one.

### Off-boarding a tenant — kill switch

To revoke an entire partner tenant immediately:

- Remove their `<issuer>` from the APIM policy → all their tokens stop
  validating within seconds of policy deploy.
- (Defense in depth) Ask their admin to remove the enterprise app for
  `apim-obo-middletier` from their tenant → no new tokens can be issued at all.

No cleanup is needed in your directory because no external user records exist
there.

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
- **Leaking a shared client secret.** If you go down the (not recommended)
  path of a single multi-tenant `foundry-mcp-client` shared across partners,
  rotation becomes a coordinated event across every partner tenant. Per-tenant
  client app regs avoid this entirely.
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
