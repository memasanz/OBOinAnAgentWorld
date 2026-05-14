# Hybrid: Entra External ID Front Door + OBO for Internal Users

This document describes the **hybrid pattern** that combines:

- **External ID (CIAM)** as the single sign-in front door for *all* users —
  internal employees and external partners alike, and
- **OBO** through APIM **only** for internal users, so the **internal
  Foundry project's MCP tools** can act as the user against Microsoft Graph
  / SharePoint / etc. inside your workforce tenant.

External partners get the no-OBO, MI-only treatment from
[`multitenant-external-id.md`](./multitenant-external-id.md). Internal
employees additionally unlock the full OBO flow from
[`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) — invisibly,
through a second silent auth-code hop after sign-in.

---

## When to use this pattern

Pick this when **both** are true:

1. You want a **single front door** (and single web-app sign-in experience)
   for internal employees and external partners.
2. The **internal Foundry project** needs to act as the user against
   workforce M365 data (Graph, SharePoint, Outlook, Teams, …) via OBO,
   while external partner Foundry projects do **not** (they only touch
   Tenant A backend resources via per-agent MI).

If only #1 is true, use the plain
[`multitenant-external-id.md`](./multitenant-external-id.md) doc — the
External ID front door already supports internal users. If only #2 is
true, use [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md)
on its own.

---

## Two user populations, one door, two backend treatments

| | **Internal user** (workforce tenant A) | **External partner user** (Tenant B / Google / email OTP / …) |
|---|---|---|
| Signs into the web app via | External ID (federated to **workforce A**) | External ID (federated to Tenant B / Google / email OTP / …) |
| Token at the web app boundary | External-ID-issued (`iss = External ID`, `partnerId = internal`) | External-ID-issued (`iss = External ID`, `partnerId = partner-b`) |
| APIM `/chat` validates | Same External ID issuer | Same External ID issuer |
| Routed to which Foundry project | Internal project | Per-partner project (or shared external project) |
| Foundry project authenticates to **Tenant A backends** (Cosmos, Search, …) | Internal project's MI (RBAC scoped to **internal slice**) | Per-partner MI (RBAC scoped to **that partner's slice**) |
| Foundry project authenticates to **M365 (Graph / SharePoint)** as the user | **Yes — via OBO** through APIM `/graph` | ❌ never — partners have no M365 in your tenant to access |
| Extra app regs in workforce A used | `apim-obo-middletier`, `foundry-mcp-client` (existing OBO machinery) | None |

Notice the **External ID-issued token is not useful for OBO** against
workforce M365 — its `iss` is your External ID tenant, not your workforce
tenant. So when the internal Foundry project needs Graph, it triggers a
**separate, silent** auth-code flow against the workforce tenant via APIM
Credential Manager + `foundry-mcp-client` — exactly the existing OBO
pattern, just transparently chained behind the External ID sign-in.

---

## Architecture

```
┌────────────────────────────┐                ┌──────────────────────────────────────────────────┐
│  Your External ID tenant   │                │   Your workforce tenant (A)                      │
│  (separate Entra tenant)   │                │                                                  │
│                            │                │                                                  │
│  Federated IdPs:           │                │                                                  │
│  • Workforce A (employees) │                │  App regs:                                       │
│  • Tenant B Entra (B users)│                │  • agent-host-webapp (External ID reg)           │
│  • Google / email OTP / …  │                │  • apim-obo-middletier  ◄─── OBO middle tier     │
│                            │                │  • foundry-mcp-client   ◄─── credential mgr      │
│  ┌──────────────────────┐  │                │                                                  │
│  │ Internal user        │──┼─sign-in via────┼─► ┌────────────────────────────────┐             │
│  │ (workforce A)         │  │  External ID   │  │ Web app (agent host)           │             │
│  │  via federation       │  │  (silent SSO)  │  │ - holds External ID session    │             │
│  └──────────────────────┘  │                │  │ - calls /chat with EID token   │             │
│                            │                │  └──────────────┬─────────────────┘             │
│  ┌──────────────────────┐  │                │                 │                               │
│  │ Partner-B user       │──┼─sign-in via────┼─►               ▼                               │
│  │ (Tenant B / Google / │  │  External ID   │  ┌────────────────────────────────┐             │
│  │  email OTP / …)       │  │                │  │ APIM /chat                     │             │
│  └──────────────────────┘  │                │  │ - validate-jwt (iss=External ID│             │
│                            │                │  │   aud=agent-host-webapp,       │             │
└────────────────────────────┘                │  │   require partnerId)           │             │
                                              │  │ - lookup partnerId → Foundry   │             │
                                              │  └──────────────┬─────────────────┘             │
                                              │                 │                               │
                                              │      ┌──────────┼────────────────────┐          │
                                              │      ▼          ▼                    ▼          │
                                              │ ┌──────────┐ ┌──────────┐      ┌──────────┐    │
                                              │ │ Internal │ │ Partner-B│ ...  │ Partner-N│    │
                                              │ │ Foundry  │ │ Foundry  │      │ Foundry  │    │
                                              │ │ project  │ │ project  │      │ project  │    │
                                              │ └─┬──────┬─┘ └────┬─────┘      └────┬─────┘    │
                                              │   │MI    │OBO     │MI               │MI         │
                                              │   ▼      ▼        ▼                 ▼           │
                                              │ ┌─────┐  │   ┌──────────┐      ┌──────────┐    │
                                              │ │Inter│  │   │Partner-B │      │Partner-N │    │
                                              │ │nal  │  │   │data slice│ ...  │data slice│    │
                                              │ │data │  │   └──────────┘      └──────────┘    │
                                              │ │slice│  │                                      │
                                              │ └─────┘  ▼                                      │
                                              │       ┌────────────────────────────┐            │
                                              │       │ APIM /graph                │            │
                                              │       │ - validate-jwt             │            │
                                              │       │   (iss=workforce A,        │            │
                                              │       │    aud=apim-obo-middletier)│            │
                                              │       │ - OBO exchange → Graph tk  │            │
                                              │       │ - cache obo-graph-{oid}    │            │
                                              │       └──────────────┬─────────────┘            │
                                              │                      ▼                          │
                                              │              ┌──────────────┐                   │
                                              │              │ Microsoft    │                   │
                                              │              │ Graph        │                   │
                                              │              └──────────────┘                   │
                                              └──────────────────────────────────────────────────┘
```

Two APIM APIs:

- **`/chat`** — accepts External-ID-issued tokens, used by **all** users.
  Routes to the appropriate Foundry project by `partnerId`.
- **`/graph`** — accepts **workforce-issued** tokens with audience
  `apim-obo-middletier`, performs OBO. Only the **internal Foundry
  project** ever calls this; partner Foundry projects don't have access
  to it (and even if they did, their users' tokens have the wrong issuer
  and would fail `validate-jwt`).

---

## App registrations

Three app regs are involved. Two of them are exactly the OBO ones from
[`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md); the new one is
the External ID front-door app.

### `agent-host-webapp` — in your **External ID tenant**

The web-app sign-in audience for everyone (internal + external).

- Lives in your External ID tenant, not workforce.
- Multi-tenant client app (so users from any federated IdP can use it).
- Exposes `access_as_user` scope so it's the audience APIM `/chat` validates.
- Standard redirect URI for the web app's sign-in callback.
- Issues `partnerId` claim (extension attribute / app role / group).

### `apim-obo-middletier` — in your **workforce tenant A** (existing)

Unchanged from [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md).

- Holds Microsoft Graph delegated permissions (`User.Read`,
  `Files.Read.All`, etc.).
- APIM authenticates as this app to do the OBO `jwt-bearer` exchange.
- Audience for the workforce-issued user token used on `/graph`.

### `foundry-mcp-client` — in your **workforce tenant A** (existing)

Unchanged from [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md).

- The OAuth client APIM Credential Manager uses on Foundry's behalf.
- Mints **workforce-issued** user tokens with
  `aud = apim-obo-middletier`.
- Used **only** by the internal Foundry project. Partner Foundry projects
  do not have a Credential Manager connection to it.

---

## Sign-in flow (one front door)

```
1. Browser → web app
2. Web app → External ID /authorize (client_id=agent-host-webapp[EID])
3. User picks an IdP:
   ─ Internal: "Sign in with Workforce A"
                → External ID redirects to workforce /authorize
                → workforce signs the user in (MFA / CA applies HERE)
                → returns code to External ID
                → External ID issues its own token (iss=External ID,
                  partnerId=internal, oid=External-ID oid)
                → side effect: browser now has a workforce session cookie
   ─ External: "Sign in with Partner-B Entra / Google / email OTP"
                → External ID handles federation accordingly
                → External ID issues its token (partnerId=partner-b)
4. Browser ← External ID auth-code → web app
5. Web app exchanges code → External ID-issued tokens → session cookie
```

After step 5, every internal-user browser also holds a **workforce-tenant
session cookie** as a side effect of step 3's federation hop. This is the
key that makes the OBO chain silent later.

---

## OBO flow for internal users — the silent second hop

When the internal Foundry project's MCP tool needs Graph:

```
1. Foundry agent → invoke Graph MCP tool
2. MCP tool → APIM Credential Manager: "give me a USER token for this user"
   (connection = foundry-mcp-client)
3. Credential Manager → /authorize on workforce tenant
   client_id = foundry-mcp-client
   scope     = api://apim-obo-middletier/access_as_user offline_access
4. Browser hits workforce /authorize.
   Because the browser already has a workforce session (from sign-in step 3
   above), the auth-code flow is SILENT — no UI prompt.
5. Workforce returns code → Credential Manager exchanges for tokens
   → caches USER token + refresh token per user
6. MCP tool → APIM /graph with USER token (iss=workforce, aud=middletier)
7. APIM /graph: validate-jwt → OBO exchange → Graph token → call Graph
8. Result returned to agent → web app → user
```

Two important properties:

- **The user notices nothing.** Step 4 redraws no page; the iframe / popup
  flow completes via the workforce session cookie.
- **The OBO machinery is identical** to the single-tenant OBO docs. We did
  not invent a new flow — we just stacked it after External ID sign-in.

If a partner user (`partnerId = partner-b`) somehow tried to invoke a
Graph MCP tool, the flow would fail at step 4 — they have no workforce
session, no consent, and no `foundry-mcp-client` user record. Don't expose
the Graph MCP tool in partner Foundry projects.

---

## APIM policies

### `/chat` — same as the External ID doc

External-ID-issued tokens; partner allowlist via `partnerId` claim.
See [`multitenant-external-id.md` → APIM `validate-jwt`](./multitenant-external-id.md#apim-validate-jwt-option-b)
for the full policy. The `<required-claims>` allowlist must include
`internal`:

```xml
<claim name="extension_{{AGENT_HOST_WEBAPP_CLIENT_ID_NO_DASHES}}_partnerId" match="any">
  <value>internal</value>
  <value>partner-b</value>
  <value>partner-c</value>
</claim>
```

### `/graph` — same as the OBO doc, scoped to workforce

Workforce-issued tokens; OBO exchange against Microsoft Graph. See
[`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) for the full
policy. The key constraint:

```xml
<validate-jwt …>
  <openid-config url="https://login.microsoftonline.com/{{WORKFORCE_TENANT_ID}}/v2.0/.well-known/openid-configuration" />
  <audiences>
    <audience>api://{{APIM_OBO_MIDDLETIER_CLIENT_ID}}</audience>
    <audience>{{APIM_OBO_MIDDLETIER_CLIENT_ID}}</audience>
  </audiences>
  <issuers>
    <!-- ONLY workforce tokens are accepted on /graph.
         External-ID-issued tokens cannot pass this gate. -->
    <issuer>https://login.microsoftonline.com/{{WORKFORCE_TENANT_ID}}/v2.0</issuer>
  </issuers>
  <required-claims>
    <claim name="scp" match="any"><value>access_as_user</value></claim>
  </required-claims>
</validate-jwt>
```

This dual-policy setup is what enforces the rule: **only internal users
can hit `/graph`**. The wrong-issuer check is automatic and unforgeable.

---

## Foundry project configuration

### Internal project

- System-assigned MI with RBAC on the **internal data slice** (Cosmos,
  Search, Storage as needed).
- MCP tools have **two flavors**:
  - **Tenant-A backend tools** — call Cosmos / Search / etc. directly
    using `DefaultAzureCredential` (the project's MI).
  - **Graph / SharePoint tools** — call APIM `/graph` (or `/sharepoint`).
    Use APIM's **Credential Manager** connection to `foundry-mcp-client`
    as the OAuth provider, exactly like the single-tenant OBO docs.
- Reads `x-user-partnerId = internal` and `x-user-oid` from APIM headers
  for per-user filtering on Tenant-A data.

### Partner projects (one per partner, or one shared external)

- System-assigned MI with RBAC on **that partner's slice only**.
- MCP tools call backends directly using the project's MI.
- **Do not** wire up any tool that calls APIM `/graph`. Partner users have
  no workforce session, no foundry-mcp-client consent, and no business
  with workforce M365.
- Reads `x-user-partnerId` and `x-user-oid` from APIM headers for tagging
  / filtering.

---

## Onboarding

### Internal — initial setup (one time)

1. In External ID, add **workforce tenant A** as a federated IdP.
2. In External ID, ensure internal users get tagged `partnerId = internal`
   (extension attribute, app role, or group). Auto-assign on first
   federated sign-in is the simplest pattern.
3. Add `internal` to APIM `/chat` `<required-claims>` allowlist.
4. Add `<your-tenant-id> → <internal-foundry-url>` to
   `PARTNER_FOUNDRY_MAP`.
5. Provision the internal Foundry project + internal data slice + RBAC
   for the project's MI on the slice.
6. **Stand up `apim-obo-middletier` and `foundry-mcp-client` in workforce
   tenant A** following the OBO doc. Wire up APIM `/graph`. Grant admin
   consent on Graph permissions.
7. In the internal Foundry project, add an APIM Credential Manager
   connection to `foundry-mcp-client` and bind the Graph MCP tools to it.

### Onboarding a new partner

Same as [`multitenant-external-id.md` → Onboarding](./multitenant-external-id.md#onboarding-a-new-partner-option-b).
**Do not** grant the partner project any access to `foundry-mcp-client`,
to APIM `/graph`, or to the workforce-tenant OBO machinery.

---

## Common pitfalls (hybrid-specific)

- **Trying to use the External-ID token for OBO.** It can't work — its
  `iss` is your External ID tenant, and AAD's OBO endpoint requires the
  user token's issuer to match the tenant where the OBO is being
  performed. The silent second hop via `foundry-mcp-client` is the only
  supported path.
- **Forgetting the workforce session-cookie dependency.** The "silent"
  second hop relies on the browser having a workforce session from the
  External ID federation step. If the user landed via a non-federated IdP
  (Google / email OTP), they have no workforce session → the second hop
  is no longer silent. This is fine for partners (they shouldn't reach
  `/graph` at all) but means **internal users must sign in via the
  workforce IdP**, not via Google or another personal IdP, for OBO to be
  invisible.
- **Granting partner Foundry projects access to `foundry-mcp-client`.**
  Don't. Even if the OBO would fail at the issuer check, the principle of
  least privilege says the partner project shouldn't have a credential
  connection it should never use.
- **Letting an internal Foundry tool route partner users to `/graph`.**
  The agent code should be defensive: when `x-user-partnerId != internal`,
  Graph tools should refuse to invoke at all. Don't rely solely on the
  APIM `<issuers>` check — make it explicit in code too (defense in
  depth).
- **Using the External ID `oid` for OBO cache keying.** APIM `/graph`
  caches OBO tokens keyed on the **workforce `oid`** (the one in the
  user token from `foundry-mcp-client`), not the External-ID `oid`. The
  two are different identifiers for the same human; mixing them up
  silently breaks per-user caching and potentially leaks tokens.
- **Skipping admin consent on `apim-obo-middletier`'s Graph
  permissions.** Same as the single-tenant OBO doc. Your workforce admin
  must grant consent once for the delegated permissions.

---

## See also

- [`README.md`](./README.md) — auth families overview
- [`multitenant-external-id.md`](./multitenant-external-id.md) — External ID
  pattern without OBO (external partners only, or simple internal use)
- [`obo-foundry-apim-multitenant.md`](./obo-foundry-apim-multitenant.md) —
  Option A: multi-tenant Entra app reg (no External ID)
- [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) — single-tenant
  OBO to Microsoft Graph (the building block reused here for `/graph`)
- [`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md) —
  same pattern as Graph but for SharePoint REST
