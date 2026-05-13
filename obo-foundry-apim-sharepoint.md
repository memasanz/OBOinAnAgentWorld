# OBO Flow: AI Foundry → APIM → SharePoint

This document describes how to set up an **On-Behalf-Of (OBO)** flow where an
AI Foundry agent calls an APIM-fronted SharePoint API on behalf of a signed-in
user. APIM is the middle tier that performs the OBO token exchange (Option A).

**Example endpoint:**
```
GET https://mmz-apim-std.azure-api.net/sharepoint/mngenvmcap272547.sharepoint.com:/sites/dataforfishing
```

---

## Why OBO?

The chain of identities looks like this:

```
User → AI Foundry Agent → APIM → SharePoint
```

SharePoint must enforce permissions based on **the user**, not the agent.
OBO lets a middle tier (APIM) exchange a token issued *for itself* into a new
token for a downstream API (SharePoint), preserving the user's identity (`sub`/`oid`).

---

## Token Flow Overview

```
1. User signs in → token with scope: api://<APIM_CLIENT_ID>/access_as_user
                   (audience = APIM's app registration)

2. AI Foundry agent calls APIM with that user token in
   Authorization: Bearer <token>

3. APIM validates the token (validate-jwt policy)

4. APIM performs OBO exchange against AAD:
      POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token
      grant_type            = urn:ietf:params:oauth:grant-type:jwt-bearer
      client_id             = <APIM_CLIENT_ID>
      client_secret         = <APIM_CLIENT_SECRET>
      assertion             = <user's token>
      scope                 = https://<tenant>.sharepoint.com/.default
      requested_token_use   = on_behalf_of

5. AAD returns a NEW token (audience = SharePoint, sub = the user)

6. APIM forwards request to SharePoint with the new token
```

---

## Step 1 — Create the APIM Middle-Tier App Registration

In **Entra ID → App registrations → New registration**:

- Name: `apim-obo-middletier`
- Single tenant
- No redirect URI needed (this app does not interactively sign users in itself)

After creation:

1. **Expose an API**
   - Application ID URI: `api://<APIM_CLIENT_ID>`
   - Add a scope: `access_as_user`
     - Admins and users can consent
     - Display name: "Access SharePoint on behalf of user"
2. **Certificates & secrets** → New client secret → save as `APIM_CLIENT_SECRET`
3. Note the **Application (client) ID** → `APIM_CLIENT_ID`
4. Note the **Tenant ID** → `TENANT_ID`
5. **API permissions** → Add a permission:
   - **SharePoint** → Delegated → `Sites.Read.All` (or `AllSites.Read`, etc.)
   - Click **Grant admin consent**
6. **Manifest** → set `"accessTokenAcceptedVersion": 2`

> **Recommended pattern:** use this single app registration as both the audience
> for the user's token *and* the identity that performs the OBO exchange. Fewer
> moving parts. If you split into a separate "client" app, you must add it to
> `knownClientApplications` in the APIM app's manifest.

---

## Step 2 — Configure AI Foundry to Acquire the User Token

In your AI Foundry agent's tool/action definition for this API:

- Auth type: **OAuth 2.0 (On-Behalf-Of / delegated user)**
- Authority: `https://login.microsoftonline.com/<TENANT_ID>`
- Scope: `api://<APIM_CLIENT_ID>/access_as_user`
- Client ID: `<APIM_CLIENT_ID>`

On first use, Foundry prompts the user to sign in & consent. The resulting
token is attached as `Authorization: Bearer …` when calling APIM.

---

## Step 3 — Create APIM Named Values

In **APIM → Named values**, create:

| Name | Value | Notes |
|---|---|---|
| `tenant-id` | your tenant GUID | |
| `apim-client-id` | from Step 1 | |
| `apim-client-secret` | from Step 1 | Mark as **secret**; ideally Key Vault-backed |
| `sharepoint-tenant` | e.g. `mngenvmcap272547` | The SharePoint tenant prefix |

---

## Step 4 — APIM Policy: Validate Inbound Token + Perform OBO

Apply this on the `sharepoint` API (or specific operation). It:

1. Validates the user's bearer token
2. Caches OBO tokens per-user
3. On cache miss, calls AAD to perform the OBO exchange
4. Replaces the `Authorization` header with the SharePoint token
5. Routes to the SharePoint backend

```xml
<policies>
    <inbound>
        <base />

        <!-- 1. Validate inbound user token -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" require-scheme="Bearer">
            <openid-config url="https://login.microsoftonline.com/{{tenant-id}}/v2.0/.well-known/openid-configuration" />
            <audiences>
                <audience>api://{{apim-client-id}}</audience>
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

        <!-- 3. Try cache -->
        <cache-lookup-value key="@("obo-sp-" + (string)context.Variables["userOid"])" variable-name="spToken" />

        <!-- 4. On miss, perform OBO exchange -->
        <choose>
            <when condition="@(!context.Variables.ContainsKey("spToken"))">
                <send-request mode="new" response-variable-name="oboResponse" timeout="20" ignore-error="false">
                    <set-url>https://login.microsoftonline.com/{{tenant-id}}/oauth2/v2.0/token</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/x-www-form-urlencoded</value>
                    </set-header>
                    <set-body>@{
                        return
                            "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" +
                            "&client_id={{apim-client-id}}" +
                            "&client_secret={{apim-client-secret}}" +
                            "&assertion=" + System.Net.WebUtility.UrlEncode((string)context.Variables["userToken"]) +
                            "&scope=" + System.Net.WebUtility.UrlEncode("https://{{sharepoint-tenant}}.sharepoint.com/.default") +
                            "&requested_token_use=on_behalf_of";
                    }</set-body>
                </send-request>

                <set-variable name="spToken" value="@{
                    var body = ((IResponse)context.Variables["oboResponse"]).Body.As<JObject>();
                    return (string)body["access_token"];
                }" />

                <cache-store-value
                    key="@("obo-sp-" + (string)context.Variables["userOid"])"
                    value="@((string)context.Variables["spToken"])"
                    duration="3000" />
                <!-- 50 minutes; under typical 60-min token TTL -->
            </when>
        </choose>

        <!-- 5. Replace Authorization with the SharePoint token -->
        <set-header name="Authorization" exists-action="override">
            <value>@("Bearer " + (string)context.Variables["spToken"])</value>
        </set-header>

        <!-- 6. Point to SharePoint backend -->
        <set-backend-service base-url="https://{{sharepoint-tenant}}.sharepoint.com/_api/v1.0" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

---

## Step 5 — Test End-to-End

1. **Get a user token manually** for development:
   - Use [MSAL.js test page](https://jwt.ms), or
   - `az account get-access-token --resource api://<APIM_CLIENT_ID>`
     (only works if Azure CLI's app is pre-authorized on your scope)
2. Decode the token at <https://jwt.ms>. Confirm:
   - `aud` = `api://<APIM_CLIENT_ID>`
   - `scp` contains `access_as_user`
   - `oid` is present
3. Call the APIM endpoint:
   ```http
   GET https://mmz-apim-std.azure-api.net/sharepoint/mngenvmcap272547.sharepoint.com:/sites/dataforfishing
   Authorization: Bearer <user-token>
   ```
4. In APIM, use **Test console → Enable tracing** to inspect the OBO request,
   the AAD response, and the final SharePoint call.

---

## Sanity Checklist

- [ ] APIM app has SharePoint delegated permission **with admin consent**
- [ ] User's token audience = APIM app's client ID / app URI
- [ ] User's token contains the required `scp` claim
- [ ] OBO scope uses **your tenant's** SharePoint URL with `.default`
- [ ] APIM client secret is stored in a **Key Vault-backed named value**
- [ ] OBO tokens are **cached per-user** (`oid`-keyed)
- [ ] `accessTokenAcceptedVersion = 2` in the APIM app manifest

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Wrong audience on incoming token | `validate-jwt` returns 401 | Token must target APIM app, not SharePoint directly |
| Missing `.default` in OBO scope | AAD returns `invalid_scope` | Use `https://<tenant>.sharepoint.com/.default` |
| No admin consent on SharePoint permission | AAD returns `invalid_grant` / `consent_required` | Grant admin consent in the APIM app |
| Wrong SharePoint resource URI | 401 from SharePoint | Must match tenant: `https://<tenant>.sharepoint.com` |
| No token caching | High latency, AAD throttling | Cache OBO tokens per `oid` |
| `accessTokenAcceptedVersion` = 1 | Validation fails on v2 endpoint | Set to `2` in app manifest |
