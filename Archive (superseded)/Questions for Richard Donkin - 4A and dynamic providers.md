---
tags: [apim, iac, open-questions, devo-28368, sre, richard-donkin]
created: 2026-08-04
status: living-doc
related-ticket: DEVO-28368
---

# Questions for Richard Donkin — 4A ownership + dynamic providers

> Companion to [[APIM IaC - outstanding questions]].
> Raised 2026-08-04 after we made the 4A ownership decision ourselves rather than waiting on SRE.
> Full context lives in the T3 section of [[APIM IaC - ticket breakdown]].

> [!important] Why we still need to talk to Richard
> We decided we do **not** need SRE to *make* the 4A decision for us — it is our code, our ticket,
> and the change is small and contained. That still leaves two things only Richard can give us:
>
> 1. **A viability check.** We are about to write to the **shared Core app registration**, which
>    governs live customer sign-in. If there is a reason Option 1 cannot work that we cannot see from
>    the code, we want to hear it *before* we build, not in review.
> 2. **Facts we cannot get from the repo** — what is actually on that app registration in live
>    environments, and whether SRE want identity edits gated.
>
> **Framing for the conversation:** we are not asking permission to proceed. We are saying "this is
> what we intend to do and why — tell us if it breaks something you can see and we cannot."

---

## Priority 1 — blocks starting the 4A build

### R1. Is Option 1 still viable now that Auth owns *both* lists?
We chose **Option 1** from your 2026-07-28 reply: shared Core registration, `Spoke/Auth` owning the
complete authoritative **SPA and Web** redirect-URI lists. APIM contributes nothing to the
registration and only consumes the identity.

Our reasoning: the setup guide's Phase 4 explicitly configures the *Core* app registration, so a
dedicated APIM registration would mean designing something new rather than automating the documented
manual process.

**Ask:** any reason that does not hold? Option 2 (dedicated registration) stays our fallback.

### R2. Are there redirect URIs on the Core app registration that Pulumi does not know about?
This is your "adopt-before-manage" warning from point 5. Because the list is written
**authoritatively** — whoever writes it replaces the whole thing — any URI present in Azure but
absent from our code gets **silently deleted** on the next deploy.

We are building against **Dreich**, which is non-production, so we do not need the prod GSOT export
for this ticket. But we do need to know:

**Ask:** in *any* environment, does the Core app registration carry redirect URIs beyond the single
customer Dayshape URL that `auth_component.py:126` declares? If so we need them enumerated before
this ships anywhere real (and formally before T12 rollout).

### R3. Do SRE want identity edits to stay a gated/manual step?
Your original note said the team owning the app registration may *choose* to keep identity edits
gated even if we make them automatable.

**Ask:** if we build this fully automated behind a default-off config flag, is that acceptable, or do
you want the redirect-URI write to remain a reviewed manual action?

---

## Priority 2 — shapes the work, not blocking today

### R4. The authentication spike — how should we actually run it?
Your point 4 is still the real prerequisite: prove the token the portal's Try-It flow obtains has
`aud`, `iss` and `scp` claims Core will accept, and that Core resolves the signed-in principal to a
Dayshape user. Nothing in the setup guide covers this and it has never been verified end-to-end.

**Ask:** what is the most practical way to test this? Can we exercise it against Dreich, or does it
need an environment with a fully configured Core API? Is there an existing token-inspection approach
the team uses?

**Note:** if a shared registration cannot mint an acceptable token, **Option 1 dies regardless of how
tidy it is** — so this arguably outranks everything above.

### R5. Are Pulumi Dynamic Providers the sanctioned way to fill provider gaps?
We need two things Azure's Pulumi provider has no resource for: deleting the demo API/product Azure
seeds into a new APIM, and switching off developer-portal self-registration.

Our first attempt used `local.Command` running a script that called the Azure REST API directly —
following `Modules/AGW/infra/app_gateway.py` and
`Spoke/Reporting/infra/reporting_component_networking.py`. Grant has ruled that out.

We then found `DynamicProviders/AGW/agw_writer.py` and `DynamicProviders/SpokeNumber/`, which **do**
write to Azure via the official SDKs but as proper Pulumi resources with real
`diff`/`create`/`read`/`delete`, state and preview.

**Ask:** is the dynamic-provider pattern the direction of travel, and are the `local.Command` cases
legacy? We would rather build the thing you want to see more of.

### R6. Adding `azure-mgmt-apimanagement` to `Products/requirements.txt`
Version 5.0.0 pulls cleanly from the feed and exposes exactly what we need (`ApiOperations`,
`ProductOperations`, `SignUpSettingsOperations`). Every other `azure-mgmt-*` package is already
pinned in that file.

**Ask:** any approval needed to add it, or is that routine?

---

## Priority 3 — decisions we have flagged but not resolved

### R7. APIM tier — BasicV2 or a classic tier?
Verified on two fresh services: **BasicV2 seeds nothing** — no Echo API, no Starter/Unlimited
products. The classic tiers (Developer/Basic/Standard/Premium) **do**.

This means the guide's Phase 2 "delete the default resources" step is a **no-op on v2** and has never
been observed deleting anything in our spike.

**Ask:** which tier are we standardising on? It changes whether Phase 2 needs to exist at all, and
has cost and feature implications we are not equipped to judge.

### R8. Region — we defaulted to West Europe
`ApiServiceCreationDisabledForSubscription` blocks BasicV2 in **UK South** on the Sandbox
subscription, so the spike defaults to `westeurope` (overridable via `apim_location`).

**Ask:** is West Europe acceptable long-term, or is UK South a data-residency requirement we need to
get unblocked?

### R9. Does disabling portal sign-up genuinely equal the guide's 4D step?
Guide 4D says: delete the "Username and password" row from the portal's Identities blade, to stop
customers registering with any email and password.

There is **no such delete in the ARM API** and no equivalent resource in the Pulumi SDK. The closest
lever is the portal's **"allow users to sign up"** setting, which we switch off. On a brand-new
service with sign-up off from day one, no local account can ever be created — so we believe this is
equivalent.

We will verify in the portal after a deploy. **Ask:** any reason that reasoning is wrong?

---

## Also worth raising

- **Legacy `Products/Spoke/APIM` tree** — you flagged twice that the empty legacy tree confuses both
  people and AI. Grant has already deleted it; folding into T1.
- You offered more detail on request — we should take you up on that for **R4** especially.

---

## Related

- [[APIM IaC - ticket breakdown]] — T3 holds the full 4A decision and change list
- [[APIM IaC - outstanding questions]] — the wider open-questions register (Q1–Q3)
- [[Meeting prep - SRE IaC and APIM call]]
- [Slack thread — Richard's 2026-07-28 reply](https://dayshape.slack.com/archives/C0BJAKHQCUQ/p1785252661079959?thread_ts=1785171923.315119&cid=C0BJAKHQCUQ)
