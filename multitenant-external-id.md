# Multi-Tenant External Access — Option B: Entra External ID (CIAM)

This document describes the **Entra External ID** identity model for letting
external users call a Foundry-hosted agent that operates entirely on **your
tenant's** resources.

It is one of two valid options. See:
- **[Option A — Multi-tenant Entra app reg](./obo-foundry-apim-multitenant.md)** —
  partners use their own Entra tenant; one-time admin consent per partner
- **Option B — Entra External ID** *(this doc)* — partners (or their users)
  sign into your External ID tenant; no partner admin involvement

---

## When to choose Option B

Use Entra External ID when **any** of these are true:

- You have **too many partners** to chase admin consent in each Entra tenant.
- Some partners aren't on Entra at all (you want to accept Google sign-in,
  email OTP, or other social IdPs).
- You want **fully self-service onboarding** — a partner end user signs up,
  verifies their email, and is in.
- You don't want to depend on a partner-side admin doing anything, ever.

If none of those apply and you're dealing with a small number of B2B
partners that already have Entra, **Option A is simpler** — stick with it.

---

## What is Entra External ID in this picture

Entra External ID is a **separate Entra tenant** that you own and operate
specifically for external (customer/partner) identities. It is *not* your
workforce tenant. It has its own:

- Tenant ID and issuer URL (`https://<tenant>.ciamlogin.com/...`)
- App registrations (the multi-tenant `agent-host-webapp` from Option A is
  replaced by an External ID `agent-host-webapp` registration)
- User directory (external users live here, not in your workforce tenant)
- User-flow / custom-policy UX (sign-up, sign-in, password reset, MFA)
- Identity providers (federate to partners' Entra tenants, Google, email
  OTP, Facebook, etc.)

External users **never** appear in your workforce tenant. Your Tenant A
resources (Foundry, Cosmos, Search, APIM) still live in your workforce
tenant — only the *identity* layer moves to External ID.

---

## What changes vs. Option A

| Concern | Option A (multi-tenant) | Option B (External ID) |
|---|---|---|
| Identity tenant | Each partner's Entra tenant | **Your External ID tenant** (one for all external users) |
| `agent-host-webapp` registration | In Tenant A workforce | **In your External ID tenant** |
| Issuer in user tokens | `…/<partnerTenantId>/v2.0` | `…/<yourExternalIdTenantId>/v2.0` (always the same) |
| `tid` claim | Partner's tenant ID | Your External ID tenant ID (same for everyone) |
| "Which partner is this user?" | Inferred from `tid` | **Custom claim** — typically `extension_<appId>_partnerId`, an app role, or a group membership |
| APIM `<issuers>` allowlist | One entry per accepted partner tenant | **One entry** for your External ID issuer |
| Partner allowlist | The `<issuers>` list itself | A required-claim check on the custom partner ID claim |
| Onboarding per partner | One-time admin consent in Tenant B | **No partner-side action.** Your External ID admin (you) creates a "partner" record and assigns users to it. |
| Onboarding per user | Implicit on first sign-in | Sign-up flow (self-service) **or** invite by email |
| Federation to partner Entra | Inherent (it's their tenant) | **Optional** — set up External ID federation per partner if you want partner SSO |
| Works for partners not on Entra | ❌ | ✅ Google, email OTP, etc. |

Everything *downstream* of APIM — per-agent managed identity, per-partner
data slices, two-layer isolation, agent code patterns — is identical to
Option A. Only the identity model and the `validate-jwt` policy differ.

---

## Architecture (Option B)

```
┌────────────────────────────┐                ┌──────────────────────────────────────────────────┐
│  Your External ID tenant   │                │   Your workforce tenant (A)                      │
│  (separate Entra tenant)   │                │                                                  │
│                            │                │                                                  │
│  ┌──────────────────────┐  │                │   ┌────────────────────────────────┐             │
│  │ External user        │──┼─sign-in via────┼──►│ agent-host-webapp (External ID │             │
│  │ (Tenant B / Google /  │  │  user-flow     │   │  reg) — multi-tenant client    │             │
│  │  email OTP / …)       │  │                │   │  app pointing at External ID   │             │
│  └──────────────────────┘  │                │   └──────────────┬─────────────────┘             │
│                            │                │                  │ POST /chat                    │
│  ┌──────────────────────┐  │                │                  │ Authorization: Bearer USER tk │
│  │ External ID directory│  │                │                  │ (iss=External ID,             │
│  │ - federates to       │  │                │                  │  tid=External ID tenant id,   │
│  │   Tenant B Entra,    │  │                │                  │  partnerId=<custom claim>)    │
│  │   Google, email OTP  │  │                │                  ▼                               │
│  │ - app reg            │  │                │   ┌────────────────────────────────┐             │
│  │   agent-host-webapp  │  │                │   │ APIM /chat                     │             │
│  │ - users tagged with  │  │                │   │ - validate-jwt                 │             │
│  │   partnerId claim/   │  │                │   │   • iss = External ID          │             │
│  │   app role           │  │                │   │   • aud = agent-host-webapp    │             │
│  └──────────────────────┘  │                │   │ - require partnerId claim      │             │
│                            │                │   │ - allowlist partnerId values   │             │
└────────────────────────────┘                │   │ - lookup partnerId → Foundry   │             │
                                              │   │ - x-user-partnerId / x-user-oid│             │
                                              │   └──────────────┬─────────────────┘             │
                                              │                  │                               │
                                              │                  ▼                               │
                                              │   ┌────────────────────────────────┐             │
                                              │   │ Per-partner Foundry project    │             │
                                              │   │ (or single shared one — same   │             │
                                              │   │  trade-off as Option A)        │             │
                                              │   └──────────────┬─────────────────┘             │
                                              │                  ▼                               │
                                              │   ┌────────────────────────────────┐             │
                                              │   │ Per-partner data slice         │             │
                                              │   └────────────────────────────────┘             │
                                              └──────────────────────────────────────────────────┘
```

---

## APIM `validate-jwt` (Option B)

The key shift: **one issuer (your External ID tenant), with partner identity
asserted via a required custom claim**.

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized">
  <openid-config url="https://{{EXTERNAL_ID_TENANT}}.ciamlogin.com/{{EXTERNAL_ID_TENANT_ID}}/v2.0/.well-known/openid-configuration" />

  <audiences>
    <audience>api://{{AGENT_HOST_WEBAPP_CLIENT_ID}}</audience>
  </audiences>

  <issuers>
    <!-- ONE issuer — your External ID tenant.
         Partner segregation is by claim, not by issuer. -->
    <issuer>https://{{EXTERNAL_ID_TENANT_ID}}.ciamlogin.com/{{EXTERNAL_ID_TENANT_ID}}/v2.0</issuer>
  </issuers>

  <required-claims>
    <claim name="scp" match="any">
      <value>access_as_user</value>
    </claim>
    <!-- The custom partner ID claim must be present.
         Allowlist of accepted values is enforced here. -->
    <claim name="extension_{{AGENT_HOST_WEBAPP_CLIENT_ID_NO_DASHES}}_partnerId" match="any">
      <value>partner-b</value>
      <value>partner-c</value>
      <!-- add more as you onboard -->
    </claim>
  </required-claims>
</validate-jwt>

<set-variable name="callerPartner"
              value="@(context.Principal.Claims.GetValueOrDefault(
                  "extension_{{AGENT_HOST_WEBAPP_CLIENT_ID_NO_DASHES}}_partnerId",""))" />
<set-variable name="callerOid"
              value="@(context.Principal.Claims.GetValueOrDefault("oid",""))" />
```

The agent then uses `x-user-partnerId` (instead of `x-user-tid`) as the
partner identifier; everything downstream — `PARTNER_FOUNDRY_MAP` lookup,
data slice selection, query filtering — is the same as Option A.

---

## Per-partner-vs-shared Foundry project (orthogonal axis)

This decision is independent of the identity model:

- **Per-partner Foundry project + scoped MI** — same hard IAM wall as
  described in the [Option A doc](./obo-foundry-apim-multitenant.md#two-layer-isolation-model).
  Recommended for small-to-medium numbers of partners and for any partner
  whose contract requires IAM-enforced isolation.
- **One shared external Foundry project + shared MI with broad RBAC** —
  scales to many partners; cross-partner isolation falls back to the
  agent's `partnerId` filter on every query. Pair with per-partner data
  slices (Cosmos containers, Search indexes) so that a missing filter
  surfaces as "no results" rather than "wrong results."

> **Most common Option B shape:** External ID + one shared external Foundry
> project. Identity scales (no per-partner consent), and Foundry/data ops
> scale (no per-partner project provisioning). The flip side:
> cross-partner isolation is entirely application-layer.

---

## Onboarding a new partner (Option B)

| # | Owner | Action |
|---|---|---|
| 1 | You | In External ID, create the mechanism that tags users with `partnerId = partner-b` (custom extension attribute, security group, or app role). |
| 2 | You | Either (a) configure External ID federation to the partner's Entra tenant for partner SSO, or (b) enable email/OTP sign-up restricted to the partner's email domain. |
| 3 | You | Add `partner-b` to the `<required-claims>` allowlist in APIM `validate-jwt`. Deploy. |
| 4 | You | Provision the partner's data slice in Tenant A (Cosmos container, Search index, Storage container). |
| 5 | You | Provision a per-partner Foundry project (if using per-partner model) or grant the shared external Foundry MI RBAC on the new slice (if shared model). |
| 6 | You | Add `partner-b → <foundry url>` to `PARTNER_FOUNDRY_MAP` (per-partner model only). |
| 7 | Partner user | Sign up via the External ID user-flow URL, or accept an emailed invite. **No partner admin involvement at any step.** |
| 8 | Both | Smoke test. |

---

## Off-boarding (Option B)

Independent levers, any one of which cuts off the partner:

- **Remove the `partnerId` value from the `<required-claims>` allowlist** →
  APIM rejects all their tokens within seconds of policy deploy.
- **Remove the partner from `PARTNER_FOUNDRY_MAP`** (per-partner model) →
  APIM `/chat` returns 403 even if the token validates.
- **Remove the Foundry MI's role assignments on the partner's data slice** →
  agent code fails 403 immediately.
- **Disable / delete the partner-tagged users in External ID** → no new
  tokens issued.
- **Delete the partner's data slice** → physically removes their data.

---

## Running Option A and Option B side by side

You can mix and match. APIM accepts tokens from either issuer:

- Multi-tenant `<issuer>` entries (one per Tenant B partner) for Option A
  partners
- The single External ID `<issuer>` for Option B partners

Per-partner routing keys differ (`tid` for Option A, `partnerId` for
Option B). The simplest pattern: at the top of the policy, normalize into
a single internal "partner key" variable, then use that everywhere
downstream:

```xml
<set-variable name="callerPartner" value="@{
    var iss = (string)context.Principal.Claims.GetValueOrDefault("iss","");
    if (iss.Contains("ciamlogin.com")) {
        // Option B — use the partnerId custom claim
        return context.Principal.Claims.GetValueOrDefault(
            "extension_xxxxx_partnerId","");
    } else {
        // Option A — partner == tid
        return context.Principal.Claims.GetValueOrDefault("tid","");
    }
}" />
```

`PARTNER_FOUNDRY_MAP` and the data-slice resolver then key on
`callerPartner` regardless of source.

---

## Common pitfalls (Option B specific)

- **Forgetting to put the partner ID claim in the token.** External ID does
  not surface custom extension attributes by default — you must add them as
  optional claims on the app registration's token configuration. Symptom:
  `validate-jwt` fails with "missing required claim."
- **Using the workforce tenant ID in `validate-jwt`.** All Option B tokens
  are issued by your **External ID** tenant, not your workforce tenant.
  Putting the workforce tenant ID in `<issuers>` rejects every legitimate
  request.
- **Treating External ID as your workforce directory.** Don't put employees
  there. Keep workforce identity in the workforce tenant; External ID is
  for external users only.
- **No partner-domain restriction on sign-up.** If you enable open email
  sign-up without restricting domains, anyone on the internet with an email
  can become a "partner-b" user. Always tie the `partnerId` assignment to a
  validated email domain or an explicit invite flow.
- **Federating to a partner Entra tenant without confirming the user's
  identity in External ID.** Federation only proves "this person can sign
  into Tenant B." It does *not* automatically tag them with the right
  `partnerId`. Make `partnerId` assignment an explicit step, not a side
  effect of federation.
- **Skipping App Role / Group enforcement** when using app roles instead of
  custom extension attributes. The role must be both **assigned** to the
  user and **emitted** as a claim (set `Token configuration → Add groups
  claim` or the equivalent for roles).

---

## See also

- [`README.md`](./README.md)
- [`obo-foundry-apim-multitenant.md`](./obo-foundry-apim-multitenant.md) —
  Option A (multi-tenant Entra app reg)
- [`obo-foundry-apim-graph.md`](./obo-foundry-apim-graph.md) — single-tenant
  OBO to Microsoft Graph
- Microsoft Learn — [What is Entra External ID](https://learn.microsoft.com/entra/external-id/external-identities-overview)
- Microsoft Learn — [Customize tokens (claims) for External ID](https://learn.microsoft.com/entra/external-id/customers/how-to-customize-token)
- Microsoft Learn — [Federation in External ID](https://learn.microsoft.com/entra/external-id/customers/concept-authentication-methods-customers)
