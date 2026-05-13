# OBO Flows: AI Foundry → APIM → Downstream API

This repo documents **On-Behalf-Of (OBO)** auth patterns for letting an
AI Foundry agent call user-protected APIs through Azure API Management. APIM
is the middle tier that performs the OBO token exchange (Option A in our
design choices).

## Documents

| Scenario | File |
|---|---|
| OBO to **SharePoint REST** | [`obo-foundry-apim-sharepoint.md`](./obo-foundry-apim-sharepoint.md) |
| OBO to **Microsoft Graph** | [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) |

---

## The Common Pattern

Both scenarios use the **identical** OBO shape:

```
User → AI Foundry Agent → APIM (validate JWT + OBO exchange) → Downstream API
```

What's the same in both:

- **One APIM app registration** (`apim-obo-middletier`) acts as:
  - The audience for the user's token (`api://<APIM_CLIENT_ID>/access_as_user`)
  - The identity that performs the OBO exchange (using its client secret)
- **AI Foundry** acquires the user token using the APIM app's scope and
  attaches it as `Authorization: Bearer <user-token>` when calling APIM.
- **APIM policy** does:
  1. `validate-jwt` against the user token
  2. Cache lookup for an OBO token keyed on the user's `oid`
  3. On miss, `send-request` to AAD's `/oauth2/v2.0/token` with
     `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer` and
     `requested_token_use=on_behalf_of`
  4. Cache the result
  5. Replace the `Authorization` header with the new token
  6. `set-backend-service` to the downstream API

---

## What Changes Between Scenarios

| Concern | SharePoint REST | Microsoft Graph |
|---|---|---|
| **OBO `scope` parameter** | `https://<tenant>.sharepoint.com/.default` | `https://graph.microsoft.com/.default` |
| **Backend base URL** | `https://<tenant>.sharepoint.com/_api/v1.0` | `https://graph.microsoft.com/v1.0` |
| **Delegated permission on APIM app** | SharePoint → `Sites.Read.All` (or `AllSites.Read`) | Microsoft Graph → `Sites.Read.All`, `Files.Read.All`, `User.Read`, etc. |
| **Cache key prefix (recommended)** | `obo-sp-<oid>` | `obo-graph-<oid>` |
| **Endpoint shape** | SharePoint REST URI conventions | Graph path-based REST |
| **Throttling profile** | Standard SP REST throttling | Aggressive; honor `Retry-After` |
| **Conditional Access exposure** | Lower (SP REST often less restricted) | Higher (Graph commonly under MFA / compliant device CA) |
| **Permission granularity** | Fewer, broader scopes | Many fine-grained scopes |

---

## What Stays the Same

- App registration design (single middle-tier app, optional separate client
  app with `knownClientApplications`)
- `validate-jwt` audience = `api://<APIM_CLIENT_ID>`
- Required `scp` claim = `access_as_user`
- AAD token endpoint + OBO request shape
- Per-user caching strategy and TTL (~50 min, under the typical 60-min token
  lifetime)
- `accessTokenAcceptedVersion: 2` in the APIM app manifest
- Storing the APIM client secret in a **Key Vault-backed named value**
- AI Foundry agent configuration (the user token works for *both* APIs because
  the OBO exchange happens server-side with a different `scope` each time)

---

## Can One APIM App Serve Both?

Yes — and it's the recommended approach. On `apim-obo-middletier`:

- Add **SharePoint** delegated permission (e.g., `Sites.Read.All`)
- Add **Microsoft Graph** delegated permissions (e.g., `Sites.Read.All`,
  `Files.Read.All`, `User.Read`)
- **Grant admin consent** for both

Then create two APIM APIs (`/sharepoint` and `/graph`) with policies that
differ only in:

- The OBO `scope` parameter
- The cache key prefix
- The backend base URL

---

## Cross-Cutting Pitfalls

| Pitfall | Applies to | Fix |
|---|---|---|
| User token audience targets the downstream API directly | Both | Token must be issued for `api://<APIM_CLIENT_ID>` |
| OBO scope missing `.default` | Both | Use `<resource>/.default` for v2 endpoint |
| No admin consent on the downstream permission | Both | Grant admin consent on APIM app |
| Cache key collisions across downstream APIs | Both | Use distinct prefixes per downstream resource |
| `accessTokenAcceptedVersion = 1` | Both | Set to `2` in the app manifest |
| Conditional Access blocking OBO | Mostly Graph | User's original sign-in must satisfy CA |
| Ignoring downstream throttling | Mostly Graph | Honor `Retry-After`; back off |

---

## Decision Quick-Reference

- Need to read a SharePoint **site/list/library** with the SP REST surface?
  → SharePoint REST flow
- Need cross-Microsoft-365 data (Files, Mail, Calendar, Teams, Users)?
  → Graph flow
- Need both? → One APIM app, two APIM APIs, two policies (same shape)
