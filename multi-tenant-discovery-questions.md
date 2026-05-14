# Multi-Tenant Discovery — Questions to Pick the Right Flow

Use this checklist when scoping a new multi-tenant agent deployment. The
answers map onto one of the three patterns:

- **A.** [`obo-foundry-apim-multitenant.md`](./obo-foundry-apim-multitenant.md) —
  Multi-tenant Entra app reg (per-partner consent, no External ID)
- **B.** [`multitenant-external-id.md`](./multitenant-external-id.md) —
  Entra External ID (CIAM) front door, no OBO
- **H.** [`external-id-with-internal-obo.md`](./external-id-with-internal-obo.md) —
  Hybrid: External ID front door + OBO for internal users

If you're in a hurry, jump to [Triage in three questions](#triage-in-three-questions).
For a thorough discovery, work through the [full questionnaire](#full-questionnaire)
and use the [decision matrix](#decision-matrix) at the end.

---

## Triage in three questions

If you only have time for three:

1. **Will the agent ever need to act *as the user* against the user's own
   Microsoft 365 data (their mail, files, sites)?**
   - No → likely **A** or **B** (no OBO).
   - Yes, but only for *internal* employees → likely **H**.
   - Yes for *partner* users too → re-scope. OBO across partner tenants
     adds significant complexity not covered in these docs; talk to
     identity SMEs.

2. **How many partner organizations are you onboarding, and can you reach
   each partner's IT admin once for consent?**
   - A handful (≤ ~25) and yes → likely **A**.
   - Many (dozens to hundreds), or no reliable admin contact, or partners
     aren't on Entra at all → likely **B** (or **H**).

3. **Is there a single sign-in experience requirement for internal +
   external users (one URL, one branded sign-in page)?**
   - Yes → **B** or **H** (External ID is the single front door).
   - No, internal can use a separate corporate URL → **A** is fine on its
     own; internal users go through the same workforce-tenant path.

---

## Full questionnaire

### 1. Users and identity

1.1 **Who are the populations of users?**
   - [ ] Internal employees (your workforce tenant)
   - [ ] External partners with their own Entra tenants
   - [ ] External partners *without* Entra (small companies, individuals)
   - [ ] Consumers / general public (email signup, social IdPs)

1.2 **For external users, do they all have Entra, or is it mixed?**
   - All on Entra → **A** is viable.
   - Mixed / non-Entra → **B** or **H**.

1.3 **For external users, can you reliably reach a partner-side admin to
   grant one-time consent?**
   - Yes, every time → **A** is viable.
   - Sometimes / no → **B** or **H**.

1.4 **Do you want internal employees to sign in with their existing
   workforce credentials (SSO)?**
   - Yes → **A** (native), or **B/H** with workforce federated into
     External ID.
   - Don't care → any.

1.5 **Do partners (or their admins) have a hard "no shared accounts in
   another directory" rule?**
   - Yes → **A** is preferred (their users stay in their tenant; nothing
     materializes in your External ID).
   - No → **B** or **H** is fine.

1.6 **Is anyone OK with B2B guest invites?**
   - Yes → out of scope for these docs (this whole repo is "no B2B").
   - No (the standard answer here) → continue.

### 2. What the agent reads / writes

2.1 **Does the agent need to read the user's own M365 data (Graph,
   SharePoint, Outlook, OneDrive, Teams, etc.)?**
   - Never → **A** or **B**.
   - Internal employees only → **H**.
   - For external users too → **out of scope**; OBO across foreign tenants
     requires partner-tenant Graph permissions and per-partner Entra app
     consents that none of these docs cover.

2.2 **Are the backend resources the agent reads strictly inside your
   tenant (Cosmos, AI Search, Storage, internal APIs)?**
   - Yes → **A**, **B**, or **H** all work.
   - No, includes other Microsoft / third-party APIs as the user → may
     need OBO; **H** if internal-only, otherwise rescope.

2.3 **Does the agent perform write actions on behalf of the user (create
   files, send mail, post to Teams)?**
   - Yes → significantly raises the OBO authorization stakes; double
     check delegated permissions and admin consent.

### 3. Per-partner data and isolation

3.1 **Does each partner have / need their own data slice (separate
   Cosmos container, separate Search index, separate blob container)?**
   - Yes, contractually or operationally → enables the per-agent-MI
     hard-IAM-wall (covered in **A**, **B**, **H**).
   - No, single shared data store → you can still isolate at the row
     level, but cross-partner protection is application-layer only.

3.2 **Will any partner contractually require "our data is isolated by
   Azure RBAC, not application logic"?**
   - Yes → use the per-partner Foundry project + scoped MI pattern
     (covered in all three docs); don't go shared-Foundry for that
     partner.
   - No → either per-partner or shared Foundry is acceptable; pick based
     on ops cost.

3.3 **How many partners do you expect to onboard, total?**
   - 1–10 → per-partner Foundry project is cheap and clean.
   - 10–25 → still per-partner, but watch the ops overhead.
   - 25+ → strongly consider shared external Foundry project (the
     orthogonal axis covered in **B** and **H**).

3.4 **Will partners ever need custom system prompts, custom tools, or
   different model versions?**
   - Yes → per-partner Foundry project (their own customization surface).
   - No → shared external Foundry project is fine.

### 4. Operational and trust posture

4.1 **Does your security/compliance team accept "the policy filters
   correctly" as the cross-partner protection?**
   - No → per-partner MI with scoped RBAC is required (hard wall, IAM-
     enforced).
   - Yes → shared MI is acceptable.

4.2 **Are any partner workloads in regulated domains (HIPAA, FedRAMP,
   FINRA, GDPR with data-residency)?**
   - Yes → per-partner Foundry project + scoped MI; per-partner data
     slice; possibly per-region deployment.
   - No → less constraint on isolation model.

4.3 **Can you operate an additional Entra tenant (External ID)?**
   - Yes (it's a real product surface — sign-in UX, IdP federation,
     monitoring, lifecycle of users) → **B** or **H** are open.
   - No → **A** is your option.

4.4 **Who owns identity verification of external users — you or the
   partner?**
   - Partner (their IT vetted them) → **A** lets you inherit that trust.
   - You → **B** / **H** put the verification burden (email
     verification, domain restriction, invite gating) on you.

### 5. User experience and branding

5.1 **Do you need a branded, customizable sign-in page (your logo,
   custom flows, custom MFA prompts)?**
   - Yes → **B** or **H**.
   - No, partners can see "their tenant" sign-in is fine → **A**.

5.2 **Do you need self-service sign-up for external users (no admin in
   the loop)?**
   - Yes → **B** or **H**.
   - No, all onboarding is admin-mediated → **A** is fine.

5.3 **Will internal and external users hit the same URL / web app?**
   - Yes → **B** or **H** (single front door).
   - No, internal use a separate corporate URL → **A** works fine and
     keeps things simpler.

### 6. Conditional Access and MFA

6.1 **Whose Conditional Access policies should apply to partner users
   (MFA, device compliance, sign-in risk)?**
   - The partner's (let their workforce CA flow through) → **A** —
     partner CA applies natively because they sign in to their own
     tenant.
   - Yours (you set CA on your External ID directly) → **B** or **H**.

6.2 **Do you require enforced MFA for all external users?**
   - Yes → confirm the chosen path supports it. Both **A** (via partner
     CA, only if partner has it) and **B/H** (via External ID
     authentication policies you set) can do this. **A** depends on
     partner cooperation; **B/H** is in your hands.

### 7. Existing assets

7.1 **Do you already have an OBO middle tier (`apim-obo-middletier`)
   from the single-tenant docs?**
   - Yes → **H** reuses it directly. Lower implementation cost than
     starting from scratch.
   - No → **A** or **B**; add OBO later as **H** if requirements grow.

7.2 **Do you already operate an Entra External ID tenant for other
   products?**
   - Yes → **B** or **H** are nearly free to add this app to.
   - No → factor the External ID setup cost into your decision.

7.3 **Is there an existing Foundry deployment serving internal users
   today?**
   - Yes → upgrading it to multi-tenant is straightforward; check that
     its current data slice / MI assumptions match the chosen pattern
     before adding partner projects.
   - No → greenfield; pick whichever fits and build from scratch.

---

## Decision matrix

| If you answered… | …pick |
|---|---|
| No OBO + few Entra-only partners + admin reachable | **A** |
| No OBO + many partners OR non-Entra users OR self-service signup needed | **B** |
| OBO required for internal users + want a single front door | **H** |
| OBO required for partner users too (against partner M365) | Out of scope — escalate |
| Single front door required + partner-CA must transit | Hard to reconcile; usually **B/H** wins on UX, accept that you set CA |
| Existing single-tenant OBO setup, need to add external partners | **H** (reuses your middle tier) |
| Per-partner contractual IAM isolation + ≤25 partners | Any of A/B/H, with **per-partner Foundry project + scoped MI** |
| Per-partner contractual IAM isolation + 25+ partners | Same as above; per-partner is still required for those partners — push back on shared-Foundry for them specifically |
| No isolation contract + 25+ partners | **B** or **H** with **shared external Foundry project** |

---

## Red flags to escalate before picking

These don't fit any of the three patterns cleanly. Surface them early
rather than discovering them mid-build:

- **Agent must act as a partner user against partner-tenant M365.** This
  is OBO across foreign tenants; needs additional partner-side app
  registrations, per-partner Graph permissions, and partner admin
  consent on each one. None of the three docs cover this.
- **Partner must bring their own data plane (e.g., agent reads from a
  partner-hosted SQL DB).** Crosses an additional trust boundary; needs
  outbound networking, per-partner credentials/secrets, and a clear
  data-flow contract.
- **Partner must self-administer their Foundry project (write their own
  agents, deploy their own tools).** Conflicts with the "partner has no
  Foundry/Azure access" rule baked into all three docs. Either grant them
  delegated admin in your tenant (B2B-like, breaks the "no guests" rule)
  or stand them up with their own Foundry in their own subscription
  (different architecture).
- **No-internet / fully air-gapped partner.** External ID and the
  multi-tenant token issuance both depend on Entra reachability. Won't
  work without internet to login.microsoftonline.com / ciamlogin.com.
- **Sub-second latency requirement on the OBO path.** OBO adds an Entra
  round-trip on cache miss. Plan for that or rescope.

---

## See also

- [`README.md`](./README.md) — overview and decision quick-reference
- [`obo-foundry-apim-multitenant.md`](./obo-foundry-apim-multitenant.md) — Option A
- [`multitenant-external-id.md`](./multitenant-external-id.md) — Option B
- [`external-id-with-internal-obo.md`](./external-id-with-internal-obo.md) — Hybrid
