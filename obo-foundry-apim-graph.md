# OBO Flow: AI Foundry → APIM → Microsoft Graph

This document describes how to set up an **On-Behalf-Of (OBO)** flow where an
AI Foundry agent calls a Microsoft Graph endpoint through APIM on behalf of a
signed-in user. APIM is the middle tier that performs the OBO token exchange
(Option A).

**Example endpoint (Graph equivalent of the SharePoint site lookup):**
```
GET https://mmz-apim-std.azure-api.net/graph/v1.0/sites/mngenvmcap272547.sharepoint.com:/sites/dataforfishing
```

---

## Why OBO?

```
User → AI Foundry Agent → APIM → Microsoft Graph
```

Graph must enforce permissions based on **the user**, not the agent. OBO lets
APIM exchange the user's token (audience = APIM) for a new token with
audience = Graph, while preserving the user's identity (`sub`/`oid`).

---

## Token Flow Overview

```
1. User signs in → token with scope: api://<APIM_OBO_MIDDLETIER_CLIENT_ID>/access_as_user
                   (audience = APIM's app registration)

2. AI Foundry agent calls APIM with that user token in
   Authorization: Bearer <token>

3. APIM validates the token (validate-jwt policy)

4. APIM performs OBO exchange against AAD:
      POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
      grant_type            = urn:ietf:params:oauth:grant-type:jwt-bearer
      client_id             = <APIM_OBO_MIDDLETIER_CLIENT_ID>
      client_secret         = <APIM_OBO_MIDDLETIER_CLIENT_SECRET>
      assertion             = <user's token>
      scope                 = https://graph.microsoft.com/.default
      requested_token_use   = on_behalf_of

5. AAD returns a NEW token (audience = Graph, sub = the user)

6. APIM forwards request to https://graph.microsoft.com with the new token
```

---

## Step 1 — Create (or Reuse) the APIM Middle-Tier App Registration

You can **reuse the same `apim-obo-middletier` app registration** used for the
SharePoint flow. Just add the Graph permissions to it.

In **Entra ID → App registrations → `apim-obo-middletier`**:

1. **API permissions** → Add a permission → **Microsoft Graph** → **Delegated permissions**.
   Pick the *minimum* permissions you actually need. Common choices:

   | Permission | Admin consent required? | When to use |
   |---|---|---|
   | `User.Read` | No | Sign in + read the signed-in user's own profile |
   | `Sites.Read.All` | **Yes** | Read items in all SharePoint sites the user can access |
   | `Files.Read.All` | **Yes** | Read all files the user can access (OneDrive + SP) |
   | `Mail.Read` | **Yes** | Read the user's mail |
   | `Calendars.Read` | **Yes** | Read the user's calendar |

   Click **Grant admin consent for `<tenant>`** at the top of the permissions list.

   ### About admin consent

   The Azure portal labels each permission with **Admin consent required: Yes/No**.

   - **No** (e.g., `User.Read`): each user *can* consent for themselves on first
     sign-in. They'll see a prompt like *"App needs permission to sign you in
     and read your profile."*
   - **Yes** (anything ending in `.All`, or anything that reads/writes data
     across users / the tenant): **only an admin can consent**. Users cannot
     self-consent.

   **Even when admin consent isn't required, you should still grant it for the AI Foundry scenario:**

   | Without admin consent | With admin consent |
   |---|---|
   | Each user sees a consent prompt on first use | Silent for all users |
   | AI Foundry / non-interactive flows may not surface the prompt cleanly | Just works |
   | OBO can fail with `consent_required` if the user hasn't consented yet | OBO works immediately |
   | Per-user consent — must be revoked per-user | Tenant-wide, centrally managed |

   Clicking **Grant admin consent for `<tenant>`** consents to *all* permissions
   on the app at once — both the optional (`User.Read`) and the required
   (`.All`) ones. One button covers everything.
2. Confirm the app still has:
   - **Expose an API** → scope `access_as_user` (audience for the user token)
   - A valid **client secret** → `APIM_OBO_MIDDLETIER_CLIENT_SECRET`
   - `"accessTokenAcceptedVersion": 2` in the manifest

> ⚠️ **`.default` returns all consented scopes.** The OBO token will carry
> every Graph delegated permission the user/admin has consented to for this
> app — not just one. That's expected behavior for v2 `.default` requests.

---

## Step 2 — AI Foundry Agent Config

No change from the SharePoint scenario. The user still signs in with scope:

```
api://<APIM_OBO_MIDDLETIER_CLIENT_ID>/access_as_user
```

The same user token works for both APIM APIs (SharePoint and Graph), because
the OBO exchange happens *server-side* with a different downstream `scope`
each time.

---

## Step 3 — APIM Named Values

Reuse the existing named values; add one for the Graph base URL if you like:

| Name | Value | Notes |
|---|---|---|
| `tenant-id` | your tenant GUID | Already exists |
| `apim-obo-middletier-client-id` | from app registration | Already exists |
| `apim-obo-middletier-client-secret` | from app registration | Key Vault-backed |
| `graph-base-url` | `https://graph.microsoft.com/v1.0` | New (optional) |

---

## Step 4 — APIM Policy: Validate Inbound Token + Perform OBO for Graph

Apply this on the `graph` API in APIM. The only differences vs. the SharePoint
policy are the **OBO scope** and the **backend base URL**.

```xml
<policies>
    <inbound>
        <base />

        <!-- 1. Validate inbound user token -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" require-scheme="Bearer">
            <openid-config url="https://login.microsoftonline.com/{{tenant-id}}/v2.0/.well-known/openid-configuration" />
            <audiences>
                <audience>api://{{apim-obo-middletier-client-id}}</audience>
            </audiences>
            <required-claims>
                <claim name="scp" match="any">
                    <value>access_as_user</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- 2. Extract user token + user object id (cache key) -->
        <set-variable name="userToken" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").Replace("Bearer ", ""))" />
        <set-variable name="userOid" value="@{
            var jwt = ((string)context.Variables["userToken"]).AsJwt();
            return jwt.Claims.GetValueOrDefault("oid", "");
        }" />

        <!-- 3. Try cache (note: Graph-specific cache key) -->
        <cache-lookup-value key="@("obo-graph-" + (string)context.Variables["userOid"])" variable-name="graphToken" />

        <!-- 4. On miss, perform OBO exchange for Graph -->
        <choose>
            <when condition="@(!context.Variables.ContainsKey("graphToken"))">
                <send-request mode="new" response-variable-name="oboResponse" timeout="20" ignore-error="false">
                    <set-url>https://login.microsoftonline.com/{{tenant-id}}/oauth2/v2.0/token</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/x-www-form-urlencoded</value>
                    </set-header>
                    <set-body>@{
                        return
                            "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" +
                            "&client_id={{apim-obo-middletier-client-id}}" +
                            "&client_secret={{apim-obo-middletier-client-secret}}" +
                            "&assertion=" + System.Net.WebUtility.UrlEncode((string)context.Variables["userToken"]) +
                            "&scope=" + System.Net.WebUtility.UrlEncode("https://graph.microsoft.com/.default") +
                            "&requested_token_use=on_behalf_of";
                    }</set-body>
                </send-request>

                <set-variable name="graphToken" value="@{
                    var body = ((IResponse)context.Variables["oboResponse"]).Body.As<JObject>();
                    return (string)body["access_token"];
                }" />

                <cache-store-value
                    key="@("obo-graph-" + (string)context.Variables["userOid"])"
                    value="@((string)context.Variables["graphToken"])"
                    duration="3000" />
                <!-- 50 minutes; under typical 60-min token TTL -->
            </when>
        </choose>

        <!-- 5. Replace Authorization with the Graph token -->
        <set-header name="Authorization" exists-action="override">
            <value>@("Bearer " + (string)context.Variables["graphToken"])</value>
        </set-header>

        <!-- 6. Point to Graph backend -->
        <set-backend-service base-url="{{graph-base-url}}" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />

        <!-- Optional: respect Graph throttling -->
        <choose>
            <when condition="@(context.Response.StatusCode == 429)">
                <set-header name="X-Graph-Throttled" exists-action="override">
                    <value>true</value>
                </set-header>
            </when>
        </choose>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

> 🔑 Use a **Graph-specific cache key** (`obo-graph-<oid>`) so it doesn't
> collide with the SharePoint OBO token cached under `obo-sp-<oid>`.

---

## Step 5 — Test End-to-End

1. Get a user token for `api://<APIM_OBO_MIDDLETIER_CLIENT_ID>/access_as_user`.
2. Decode at <https://jwt.ms> — confirm `aud`, `scp`, and `oid`.
3. Call the APIM endpoint:
   ```http
   GET https://mmz-apim-std.azure-api.net/graph/v1.0/me
   Authorization: Bearer <user-token>
   ```
4. APIM **Test console → Enable tracing** to inspect the OBO exchange and the
   downstream Graph call.

---

## Graph-Specific Considerations

1. **Granular scopes** — Pick the minimum delegated permissions. `.default`
   will return tokens carrying *all* admin-consented Graph permissions on the
   APIM app.
2. **Conditional Access (CA)** — Graph is commonly behind CA (MFA, compliant
   device). OBO can fail with `AADSTS50076` / `interaction_required` if the
   user's original sign-in didn't satisfy CA. The middle tier **cannot**
   satisfy MFA — the user must.
3. **Throttling** — Graph throttles aggressively. Caching OBO tokens (already
   done) and honoring `Retry-After` on 429 responses is important.
4. **Permission scope vs. resource access** — Having `Sites.Read.All` does not
   override item-level SharePoint permissions; OBO preserves user identity, so
   the user still needs access to the specific resource.
5. **Beta endpoint** — If you need `/beta`, change `graph-base-url` accordingly,
   but be aware Microsoft considers beta non-production.

---

## Sanity Checklist

- [ ] APIM app has the required Graph delegated permissions, **admin-consented**
- [ ] User's token audience = APIM app's client ID / app URI
- [ ] User's token contains the required `scp` claim
- [ ] OBO scope = `https://graph.microsoft.com/.default`
- [ ] APIM client secret is in a **Key Vault-backed named value**
- [ ] OBO tokens are cached per-user with a **Graph-specific key**
- [ ] `accessTokenAcceptedVersion = 2` in the APIM app manifest
- [ ] User's original sign-in satisfies any Conditional Access on Graph

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Wrong audience on incoming token | `validate-jwt` returns 401 | Token must target APIM app, not Graph directly |
| Missing/incorrect Graph scope | AAD `invalid_scope` | Use `https://graph.microsoft.com/.default` |
| No admin consent on Graph permissions | AAD `consent_required` | Grant admin consent in the APIM app |
| Cache key collision with SharePoint flow | Wrong token sent downstream | Use distinct keys (`obo-graph-<oid>`, `obo-sp-<oid>`) |
| CA blocking OBO | `AADSTS50076` / `interaction_required` | User must satisfy CA at original sign-in |
| Throttling | `429 Too Many Requests` from Graph | Cache OBO tokens; honor `Retry-After`; back off |
