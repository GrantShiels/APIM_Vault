---
tags: [apim, iac, open-questions, devo-28368]
updated: 2026-08-11
status: living-doc
related-ticket: DEVO-28368
---

> [!warning] Audited 2026-10-07 — partly stale, see [[_START HERE - APIM handover]]
> Q4 ("Basic v2 for every customer") is changing to Standard v2 (Application Gateway, INFRA-777). Q6 (West
> Europe) is now "hub region, configurable". Q5, Q7, Q9 are answered. Still open: Q3, Q8 (email scope
> consent → 30734), Q11 (testing against a real environment → 30006). Ticket numbers use the old scheme.

# APIM IaC — outstanding questions

> A **running register** of open questions from the spike.
> Every question follows the same shape: what it is in one line, **where it came from**, **how it
> relates to the setup guide**, and **what to ask**. Detail sits underneath — skip it in a call.
> Status: 🔴 open · 🟠 partial, needs confirming · ✅ answered (answer kept inline for the record).

---

## 📞 If you only have 20 minutes

> [!info] Updated 2026-08-07 after the Richard / Marcus / Sam C / Andy L call
> **Q1, Q2 and Q10 got answers** — see each entry. **Q5 is now parked**, not because it's less
> important but because Sam wants the whole auth picture discussed in a dedicated session.
>
> **The new top priority is not on this list yet: rollout and testing.** Sam's direct question was
> *"how do you get this across environments as quickly as possible, and what would an MVP look
> like?"* Captured in [[APIM IaC - rollout and testing strategy]].

The four worth spending the call on, in order:

| Priority | Question                                                    | Why now                                                                                                                       |
| -------- | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 1        | ~~**Q5 — who creates the client secret?**~~                 | ✅ **Answered 2026-08-13** — the deployment creates it itself and rotates it. See D1.                                          |
| 2        | **Q9 — how much of the 51-page guide do we actually need?** | We have built 6 of 10 phases. Sizing the remaining work depends entirely on this answer. **Sharpened by the guide being AI-generated — see G16.** |
| 3        | **Q3 — deploys outliving their own login**                  | Gates the pipeline path even if everything else is perfect. Basic v2 has softened it, not solved it.                          |
| 4        | ~~**Q1 — environments that already have an APIM**~~         | ✅ **Answered 2026-08-07** — destroy and rebuild, not import.                                                                  |

Quick ones to grab if there is time: **Q6** (which region).

---

## Register

| #   | Question                                                                                                | Status     | Best person to ask                    |
| --- | ------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------- |
| Q1  | An APIM **already exists** in the resource group — what happens when we run our IaC at it?              | ✅ answered | **Destroy and rebuild** (2026-08-07)  |
| Q2  | Can we update **just the API definition** without re-running everything?                                | ✅ answered | **Yes — it's a diff** (2026-08-07)    |
| Q3  | APIM can take longer to build than our **deploy pipeline stays logged in** for.                         | 🟠 softened | Sam Clifford · Infra-Platform        |
| Q4  | **Which APIM tier** do we deploy for real customers?                                                    | ✅ answered | **Basic v2**, spike and customers     |
| Q5  | **Who creates and stores the client secret** APIM needs to sign people in?                              | ✅ answered | **The deployment does** (2026-08-13)  |
| Q6  | APIM is being built in **West Europe** while the rest of the spoke is **UK South**. Is that acceptable? | 🔴 open    | SRE / Infra-Platform · Richard        |
| Q7  | Should APIM get **its own app registration**, or keep borrowing Core's?                                 | ✅ answered | **Borrows Core's — forced by G5**     |
| Q8  | The guide asks for an **email address** we do not currently grant permission for.                       | 🟠 open    | Richard · Auth owner                  |
| Q9  | **How much of the 10-phase guide** does every customer actually need?                                   | 🔴 open    | Richard (product/API owner)           |
| Q10 | **Who owns and runs this** once we hand it over?                                                        | 🟠 partial | **SRE for now** (2026-08-07)          |
| Q11 | **How do we get this across environments quickly, and what's the MVP?** | 🔴 open — **now the biggest** | Sam C · Andy L · Marcus |
| Q12 | **Deploy sequencing** — Core first or APIM first during an upgrade? | 🟠 half answered — **APIM first (G23)**; customer-experience half open | Richard · SRE |

---

## 🗳️ Where these questions now go for a decision

> [!info] Added 2026-08-11
> Ten of these questions can't be answered by the team — they need a product, tech-lead or SRE
> decision. Each now has its own short decision document in `Decisions\`, written to be read by
> someone who hasn't followed the spike.

> [!note] Consolidated 2026-08-13
> The original ten documents were merged down to six. Closely-coupled decisions that needed the same
> people in the same conversation are now single documents with parts. **D9 was answered and
> retired** — the full guide is in scope. Old numbers are shown below so earlier notes still make
> sense.

| Doc | Decision | Was | From |
| --- | --- | --- | --- |
| **D1 Part 1** | How the developer-portal client secret reaches production ✅ *resolved* | D1 | Q5, Q7, G7, G17 |
| **D1 Part 2** | Who renews the client secret before it expires ✅ *resolved — it renews itself* | D6 | Q10, G7 |
| **D2 Part 1** | Who owns `vprk2` as a production dependency ⬜ | D4 | Q10, G24 |
| **D2 Part 2** | Granting `vprk2` read access, and which role ⬜ | D2 + D3 | G24, G27 |
| **D2 Part 3** | Where that grant lives — code or by hand ⬜ | D3 | G27 |
| **D5** | Getting deploy access to shared environments | D5 | Q11 |
| **D7** | Accepting the unsupported portal migration | D7 | G26 |
| **D8** | Region — West Europe or UK South | D8 | **Q6** |
| **D10** | What a customer's integration sees during a deploy window | D10 | **Q12** (second half), G18 |
| ~~D9~~ | ~~MVP scope~~ → **answered: the full guide is in scope**, see Q9 | D9 | **Q9** |

---

## Q1 — An APIM already exists in the resource group

**Status:** ✅ **answered 2026-08-07 — destroy and rebuild.**
**Answered by:** Sam Clifford, Richard Donkin, Marcus Thomson.

> [!success] The answer
> **We destroy the existing APIM and rebuild it from code. We do not import it.**
>
> Sam, plainly: *"just destroy it and rebuild it."*
>
> Import is technically possible but *"not a straight import"*, and Richard described it as
> *"pretty painful and manual."* With **one customer across three environments**, a rebuild is
> cheaper than making import work.
>
> **So there is no migration process to build.** That's a chunk of anticipated work removed.

> [!danger] But there is one hard constraint — test it first
> **Azure holds the APIM name for a period after deletion** — Marcus's experience was a few hours.
> Since a rebuild reuses the same name, this will collide unless the gap is respected.
>
> Marcus's steer: **kill it, wait a few hours, then deploy — and do it in the afternoon** so the
> gap doesn't consume a working day.
>
> **We have never tested a destroy-and-recreate with name reuse.** This must be proven on the
> sandbox before it goes anywhere near the customer. Logged as **G15** in
> [[APIM IaC - known traps and gotchas]].

> [!note] Auto-destroy — considered, effectively rejected
> Having the IaC tear the old one down itself doesn't work: **Pulumi won't touch resources it
> doesn't know about**, and a hand-built APIM isn't in state, so the step would do nothing
> *(Richard)*.
>
> Pulumi **deployment hooks** do exist — before/after, able to branch on preview / up / destroy —
> and AI Agent uses them, so there's precedent. But they only act on Pulumi-managed resources.
> **Stretch goal; don't spend time on it.**

---

<details>
<summary>Original analysis, kept for the record</summary>

**Ask:** Infra-Platform (Nick / Paul / Marcus).

> **In one line:** if someone already built an APIM by hand, does our automation update it, refuse to
> run, or wipe it?

**Where it came from:** Grant, 2026-07-20, while working out what a second run of the deploy does.

**How it relates to the setup guide:** directly. The guide *is* a manual, click-through-the-portal
runbook — so **every environment that has ever followed it already has a hand-built APIM** that our
automation did not create and does not know about. Those are exactly the environments we would be
pointing this at.

**Why it matters:** the worst case is our automation tearing down a live customer's front door
because it did not recognise it.

### What we think happens

Everything hinges on **Pulumi state** — the automation's own record of what it believes it built.

| Situation | What happens |
| --- | --- |
| APIM exists **and** our automation built it | ✅ Safe. It compares what is there against what is wanted and updates only the difference. This is the normal, repeatable case. |
| APIM exists but our automation **has never seen it** (the manual-guide case) | ⚠️ It tries to create one that is already there → Azure rejects it. Needs a decision — see below. |
| No APIM yet | ✅ Normal build, no issue. |

For the middle case there are two ways out:

- **Adopt it** (`pulumi import`) — tell the automation "this existing thing is yours now", then manage
  it from then on. **Non-disruptive**, and the likely recommendation.
- **Delete and rebuild** — clean, but tears down a working service. Only sane for empty/dev
  environments.

> ⚠️ **What forces the decision:** an APIM's name is **globally unique across all of Azure** — it owns
> a public web address. So you cannot quietly stand a second one up next to the first. You must
> either adopt it or replace it. "Run both" is not on the table.

**Outcome:** the second option won. The globally-unique-name point turned out to be the *reason*
for G15, rather than an argument for import.

</details>

### Related
[[APIM IaC - ticket breakdown]] T7 / T11 · [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] · [[APIM IaC - known traps and gotchas]] G15

---

## Q2 — Updating just the API definition

**Status:** ✅ **answered 2026-08-07 — Option 1 confirmed. It's a diff, not a rebuild.**
**Answered by:** Sam Clifford.

> [!success] The answer
> **Option 1 was right.** The orchestrator does an incremental comparison — existing components are
> checked for changes, not torn down and redeployed. A new spec updates the API and leaves the
> service, policies and sign-in config alone.
>
> **When it runs:** two triggers.
> 1. An **`IaC Deploy` pipeline** — manual, lets you pick which components to run. Used for both
>    sandbox testing and real production runs. *(Exact name unconfirmed.)*
> 2. The **full software deployment pipeline** — the pipelines that deploy Core to a customer also
>    run the orchestrator.
>
> Which components run comes from **a table in the GSOT database**, per customer environment.
>
> **On the third bullet below — "should the Core release pipeline trigger this automatically?" —
> the answer is yes, and it already effectively does.** Sam endorsed publishing the spec as a
> versioned artifact so the IaC can fetch the one matching the deployed Core version. There's
> already precedent: the IaC updates **container image tags** to match the Core version.

Full design: [[APIM IaC - OpenAPI source of truth (phase 7)]].

> [!warning] What this doesn't solve
> There's still a **window during a deploy where APIM and Core are on different versions**, and a
> customer's automated integration can fail in it. Unsolved — see **G18** and §7.6 of the OpenAPI
> doc. New question **Q12** covers the sequencing half of it.

---

<details>
<summary>Original analysis, kept for the record</summary>

**Ask:** Infra-Platform · Sam Clifford.

> **In one line:** the API definition changes every Core release — we do not want that to mean
> re-running the whole APIM deployment.

**Where it came from:** Grant, 2026-07-20 · independently raised by **Sam Clifford** the same day as
his **number one build priority** from the Voodoo Park code.

**How it relates to the setup guide:** the guide's **Phase 7** imports the Entities API by hand from
an OpenAPI file. That file is **regenerated every Core release**, so it is the one part of APIM that
changes constantly — everything else in the guide is set-and-forget.

**Why it matters:** Sam's angle is sharper than "make it fast". The **automation and the software
release on different schedules**, so the risk is deploying an API definition that does not match the
Core version actually running. That is a correctness problem, not a convenience one.

> [!important] This is now a requirement, not a question
> Supporting spec-only updates is an **acceptance criterion on T7**. The open part is only *which*
> option we take.

### The options

| Option | What it is | Verdict |
| --- | --- | --- |
| **1. Just re-run it** | The automation only changes what actually differs, so a new spec updates the API and nothing else. Version becomes a config setting. | ✅ **Confirmed 2026-08-07** — this is how it works |
| **2. Target one resource** | Ask the automation to update only the API and skip everything else. | Available via the `IaC Deploy` pipeline's component selection, if ever needed |
| **3. Split the API out** | Make the fast-changing API its own deployable thing, separate from the slow-changing platform. | ⭐ **Closer to Sam's long-term view than expected** — he wants infrastructure decoupled from Core deploys eventually. Not now, but not over-engineering either |
| **4. Update Azure directly, outside the automation** | A script that pushes the spec straight to Azure. | ⚠️ **Avoid** — the automation would not know, and the next run could silently revert it |

> **Worth knowing:** "re-running everything" is not as bad as it sounds. The automation is
> *incremental* — it compares wanted-vs-actual and touches only the difference.

</details>

### Related
[[APIM IaC - ticket breakdown]] T7 · [[APIM IaC - OpenAPI source of truth (phase 7)]] · [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]]

---

## Q3 — Deploys outliving their own login

**Status:** 🔴 open — real, known blocker with a candidate fix.
**Ask:** Sam Clifford · Infra-Platform.

> **In one line:** building an APIM can take about an hour; our deploy pipeline's login expires after
> exactly an hour and cannot renew itself — so a deploy can fail purely because it took too long.

**Where it came from:** **Sam Clifford**, 2026-07-20 — his **top practical concern** with the existing
Voodoo Park code.

**How it relates to the setup guide:** it does not, and that is the point. The guide assumes a
**human clicking in the portal**, where nothing expires mid-way. This problem only exists *because*
we are automating it. It is a genuinely new risk that automation introduces.

**Why it matters:** it can stop the very **first** deploy of a new customer's APIM from ever
finishing. Nothing else in the component matters if this is not solved.

### Current thinking
- **Sam's candidate fix:** move the pipeline off the current short-lived credential type onto one that
  does not hit the one-hour ceiling. He was explicit that other options are worth exploring.
- **Basic v2 has helped** — see Q4. The v2 tiers build in **minutes rather than about an hour**, which
  shrinks the window a lot. It reduces the risk; it does not remove it.
- Other angles: make the build **asynchronous** (start it, let the pipeline finish, reconcile later);
  or find a **longer-lived credential** for slow steps.

### Ask on the call
- [ ] Is the credential swap still the agreed direction, and what does it cost us in security terms?
- [ ] Does the orchestrator **already** have a pattern for long-running steps that works around this?
- [ ] Given Basic v2 builds in minutes — **is this still a blocker, or now just a risk to watch?**

> **Not blocking the spike.** We deploy from a dev machine, not the pipeline, so this does not bite
> yet. It must be solved before SRE can run it for real. Worth its own ticket.

### Related
[[Meeting Notes - Sam Clifford on VP APIM IaC gaps]] · [[Team skills vs APIM delivery - risk assessment]]

---

## Q4 — Which APIM tier? ✅ Basic v2

**Status:** ✅ answered — **Basic v2**, for the spike and every customer deployment.

> **In one line:** Basic v2 is the cheapest tier that can actually do what the guide asks for.

**Where it came from:** the Phase 4 deploy on 2026-08-05, which failed with *"Method not allowed in
BasicV2 pricing tier"*.

**How it relates to the setup guide:** the guide **never states a tier**. It was written against a
classic tier, so some of its steps quietly do not work on the tier we are using — which is exactly
how we found this.

### Why Basic v2 holds up
It is the cheapest tier supporting both things Phase 4 depends on: the **developer portal** and
**Microsoft Entra sign-in**. Classic Basic is cheaper but Microsoft marks Entra integration as
**unavailable** there — so the whole sign-in flow the guide is built around could not work at all.
**Basic v2 is the floor, not a preference.**

It also helps **Q3**: v2 builds in minutes rather than about an hour.

### What it commits us to
Consequences to design around, not reasons to reopen it:

- **Guide steps must be checked against v2, not trusted.** Already cost us one failure with six
  phases still unbuilt. See G11 in [[APIM IaC - known traps and gotchas]].
- **No private networking.** APIM reaches Core over the public internet. Fine while Core is publicly
  reachable; a blocker if that ever changes.
- **No fixed IP address.** Anything wanting to allow-list APIM by IP will not work.
- **No backup and restore.** Recovery *is* redeploying from automation — which is an argument for
  finishing this component properly, since it becomes the only way back.
- **No route between v2 and classic**, in either direction. Changing tier later means rebuilding every
  customer's instance from scratch.
- **Ceiling of 10 scale units**, no multi-region, no availability zones.

### Follow-ups (not blocking)
- [ ] Does anything in the estate expect APIM to have a **fixed outbound IP**?
- [ ] Is the **per-customer monthly cost at 100+ instances** understood and budgeted?

### Related
[[APIM IaC - known traps and gotchas]] G11 · set in `Spoke/ApimSpike/infra/apim_spike_component.py`

---


## Q5 — Who creates and stores the client secret? ✅ **answered**

**Status:** ✅ **answered 2026-08-13.** **The deployment creates it itself**, on Core's app
registration, and replaces it every 30 days. Nothing is stored anywhere and no human ever sees it.
Full write-up: **[[D1 - The developer portal sign-in secret]]**.

> [!success] Sam called this correctly on 2026-08-07
> His Slack follow-up said he'd prefer **APIM extend Core's app registration itself** (as AiAgent
> does), rather than the Auth component knowing APIM exists — and flagged Option 2, *"currently
> written off"*, as likely the right answer. **That is exactly what shipped.**
>
> His caveat — that `ApplicationRedirectUris` may not support it — was also right, and is why the
> redirect URIs deliberately **stayed** with Auth. That resource genuinely is authoritative.
> `ApplicationPassword` is not, so the secret could move while the URIs could not.
>
> He also asked whether a **secretless approach** exists. Checked: it does not. `IdentityProvider`
> and `AuthorizationServer` both take a plain string only — neither accepts a Key Vault reference,
> unlike APIM named values.

> [!success] The renewal sub-question is answered too
> *"Who renews it before it expires, and how would we know?"* — **nobody, and you don't need to.**
> Sam's preference was *"the AI Agent approach Vinny built recently, which rotates **and** applies the
> new value"*, and said *"that would get my vote."* The implemented approach does the same thing by a
> simpler route: a 30-day timer forces a fresh secret, and because APIM creates it in the same
> deployment that uses it, applying the new value is automatic rather than a separate step.
>
> **One residual risk, stated plainly:** renewal only happens when APIM is deployed. An environment
> left alone for over twelve months still drifts.

> [!danger] Still open — a related problem
> **Secrets stored in APIM are not obfuscated** — readable in the portal by anyone with access
> *(Sam, 2026-08-07)*. Logged as **G17**, and **not** solved by any of the above: the secret still
> ends up visible inside APIM regardless of who created it. A read-only developer group may be needed.

<details>
<summary>Why it was parked, and what changed — kept for the record</summary>

**Status was:** ⏸️ parked 2026-08-07 — pending a dedicated auth meeting.

> Sam's steer on 2026-08-07 was to *"revisit your auth stuff as a whole"* rather than settle the
> secret on its own. He also split auth into two problems we'd been treating as one:
>
> 1. **User auth** — how people and systems actually call the API. Four audiences: a customer's
>    automated integration script, a developer at a customer exploring by hand, machine-to-machine,
>    and APIM → Core. *Discuss this one first.*
> 2. **Management-plane access control** — who can administer APIM and see secrets.
>
> **Grant's decision: leave auth alone for now and get agreement in a proper session.**

</details>

---

<details>
<summary>Original write-up, kept for the record</summary>

**Ask:** Richard · Infra-Platform.

> **In one line:** APIM needs a password to prove who it is when signing users in. Right now a human
> creates that password by hand and pastes it into the deploy. Nothing automated can ship like that.

**Where it came from:** the Phase 4C build, 2026-08-04/05. It is the reason the deploy needed
`APIM_IDP_CLIENT_SECRET` set by hand before it would run.

**How it relates to the setup guide:** the guide's **Phase 4C** literally says to create a client
secret in one part of the Azure portal and paste it into another. **That instruction only makes sense
if a human is doing it** — automation has no portal to copy from and nowhere approved to keep the
value. This is the clearest example of a guide step that does not survive being automated.

**Why it matters:** the secret also **expires** (we set 12 months). Someone has to own renewing it for
every customer, or sign-in silently breaks a year after go-live.

### The three-way decision

> [!note] Now expanded into its own document
> Full write-up with evidence from the codebase, four options and honest pros/cons:
> **[[APIM IaC - client secret decision]]**. Summary below.

**The framing changed after the 2026-08-05 meeting.** The question is *not* "who types the secret in"
— Dayshape's standard pattern is that **automation generates the secret**, using
`azuread.ApplicationPassword`, and either stores it in Key Vault (`Modules/ServicePrincipal`) or feeds
it straight into the resource that needs it (`Spoke/AiAgent`). Both patterns already exist and work.

The real question is **who owns the credential**, because APIM does not own an app registration — it
borrows Core's, which Auth owns.

| Option | What it means | Verdict |
| --- | --- | --- |
| **1. Auth generates it, stores in Key Vault** | Auth owns Core's registration, so it mints the secret behind the existing `apim_enabled` flag; APIM reads it back | ✅ **Looks strongest** — single owner, reuses working machinery, not blocked on anything |
| **2. APIM generates it on Core's registration** | Simplest code, but two components writing credentials to one registration | ❌ Same problem that forced the redirect-URI rework |
| **3. APIM gets its own registration** | Cleanest ownership | 🔵 Possibly better long-term, but **blocked** behind Richard's token spike — see G9 |
| **4. A human mints it, stored centrally** | Status quo made official | ⚠️ A manual step per customer at 100+ customers, with silent expiry |

### Ask on the call
- [ ] **Which of the four** — and who does the work?
- [ ] Which **Key Vault**, and what naming convention?
- [ ] How is the **ordering dependency** handled (Auth must run before APIM)?
- [x] **Who renews it** before it expires, and how would we know it had? — *answered above*
- [ ] Does APIM's identity provider accept a **Key Vault / Named Value reference**, or only a literal?
- [ ] **New:** does the AiAgent composable app-registration pattern transfer to APIM?

</details>

### Related
[[APIM IaC - client secret decision]] — the full write-up, including Sam's counter-proposal ·
[[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] ·
[[APIM IaC - ticket breakdown]] T3 · [[APIM IaC - known traps and gotchas]] G7, G9, G17

---

## Q6 — West Europe or UK South?

**Status:** 🔴 open — needs a decision before production.
**Ask:** SRE / Infra-Platform · Richard.

> **In one line:** our APIM is currently being built in **West Europe** while everything else around
> it sits in **UK South**, because Azure would not let us create it in UK South.

**Where it came from:** the first successful deploy. Azure blocked new Basic v2 APIM creation in UK
South for our Sandbox subscription, so it was pointed at West Europe to get moving.

**How it relates to the setup guide:** the guide does not mention region at all — it assumes you pick
one in the portal and move on. It became a question only because we hit a hard Azure restriction.

**Why it matters:** two angles, and the second is the serious one.

- **Latency** — a small extra hop between APIM and Core. Probably not noticeable.
- **Data residency** — customer API traffic passing through a **different country**. That is a
  question for whoever owns customer commitments, not an engineering judgement call.

**Worth knowing:** Voodoo Park already run APIM this way, so there is precedent.

### Ask on the call
- [ ] Is West Europe acceptable for **production**, or only for the sandbox?
- [ ] Does **data residency** rule it out for any customer or region we sell into?
- [ ] Is the UK South restriction **permanent, or just our sandbox subscription?**

### Related
Tracked in session todos as `apim-region-signoff`.

---

## Q7 — Should APIM have its own app registration? ✅ **answered — no, and it can't**

**Status:** ✅ **answered 2026-08-13.** It borrows Core's, and that is **forced**, not a preference.

> **In one line:** APIM borrows Core's identity to sign people into its developer portal, because
> giving it its own would produce a sign-in token Dayshape refuses to accept.

**Where it came from:** Phase 4 build, 2026-08-04. We spent significant time exploring it before
parking it on Grant's call.

**The answer, and why it is not reopenable.** The guide has APIM **reuse Core's app registration**.
We assumed that was a shortcut; it is a constraint. The sign-in token Azure hands back is stamped with
the identity that asked for it, and Dayshape's core application only accepts tokens stamped with its
own. Confirmed from Core's source rather than guessed:

- `AuthenticationExtensions.cs` validates the audience against `{ ClientId, api://{ClientId} }` —
  **Core's own identity and nothing else**.
- The obvious workaround is closed too: `IdpConfigurationProvider` has exactly **two** hard-coded
  sign-in-provider slots, both already used by real customer providers. There is no third slot for
  APIM without changing Core.

Giving APIM its own identity is therefore a **Core code change**, not an infrastructure one. Tracked
as the G5 direction, outside this epic.

**The "two components writing to one app registration" worry is resolved separately.** It turned out
to apply only to **web addresses**, which really can have just one owner and stay with Auth. The
**sign-in secret** is a different kind of thing entirely and moved into APIM on 2026-08-13. See
**[[D1 - The developer portal sign-in secret]]**.

### Related
[[APIM IaC - known traps and gotchas]] G9 *(confirmed)*, G5 *(the root cause)* ·
[[D1 - The developer portal sign-in secret]]

---

## Q8 — The email address the guide asks for

**Status:** 🟠 open — has not bitten yet, but probably will.
**Ask:** Richard · whoever owns Auth.

> **In one line:** the guide asks Azure for the user's **email address** at sign-in, but our Auth
> component never granted permission to share it.

**Where it came from:** comparing our Auth component against the guide during Phase 4 — logged as G1.

**How it relates to the setup guide:** the guide's **Phase 4B** sets the requested information to
`openid profile email`. Our Auth component only grants `openid` and `profile`. The guide assumes
someone ticked the extra box in the portal; nobody did in ours.

**Why it matters:** the developer portal likely wants an email address to identify who has signed in.
If it does not get one, portal sign-in may fail or show users incorrectly — and it would surface as a
confusing login bug, not an obvious permissions error.

### Ask on the call
- [ ] Should Auth request the **email permission** as standard?
- [ ] Does anything else depend on Auth's current permission set that we would disturb?

### Related
[[APIM IaC - known traps and gotchas]] G1 · `Spoke/Auth/infra/auth_component.py`

---

## Q9 — How much of the guide does every customer need?

**Status:** 🔴 open — this one sizes the remaining work.
**Ask:** Richard (product / API owner).

> **In one line:** the guide is 51 pages and 10 phases. We have automated 4. Before we build the rest,
> we should know which parts every customer genuinely needs.

**Where it came from:** the build itself — having now worked through four phases, the shape of the
remaining six is clearer, and some look optional or customer-specific.

**How it relates to the setup guide:** entirely. The guide is written as **one continuous runbook**
with no indication of what is mandatory, what is a recommended default, and what is there for a
specific customer's situation.

**Why it matters:** it is the difference between a small remaining piece of work and a large one, and
it directly drives how the tickets get sized.

### Ask on the call
- [ ] Which phases are **mandatory for every customer** versus optional or situational?
- [ ] Are any steps in the guide there for **one customer's requirement** rather than as a standard?
- [ ] Is anything in it now **out of date** — written for an older Core or an older Azure behaviour?
- [ ] Should the guide be **retired** once this is automated, or kept as the manual fallback?

### Related
`files/guide.txt` (extracted guide) · [[APIM IaC - ticket breakdown]]

---

## Q10 — Who owns and runs this afterwards?

**Status:** 🟠 **partial — answered for now, but the wider question stands.**
**Answered by:** Sam Clifford, Richard Donkin, 2026-08-07.

> [!success] The answer, for now
> **SRE own it.** That's the current arrangement and nobody proposed changing it for this work.
>
> Sam, on whether that's *right*: *"Big picture, should it be? Probably not… but that's a bigger
> question."* Richard's practical case for keeping it there: **SRE is better placed for
> out-of-hours**, which is the part we can't cover.
>
> **So: build it for SRE to run.** Preview-first, clear failure messages, good documentation — the
> "why it matters" below holds and is now confirmed rather than assumed.

> [!note] A useful related answer on access
> **Production is locked down; customer developers use *stage*.** Richard was blunt about
> production access: *"why would they need production data? That'd be really scary."*
>
> That matters for how we think about the developer portal — the audience exploring the API by
> hand is on **stage**, not production.

---

<details>
<summary>Original write-up, kept for the record</summary>

**Ask:** Andy L (SRE lead) · Marcus.

> **In one line:** we are building this, but we are not a DevOps team — who runs it, and who fixes it
> at 2am?

**Where it came from:** running through it from the start of the spike; captured as a follow-up with
Andy L.

**How it relates to the setup guide:** today the guide is run by whoever has the portal access and the
patience. Automating it moves that job somewhere — we should be deliberate about where, rather than
finding out by accident.

**Why it matters:** it shapes what we build. "Built for SRE to run" means preview-first, clear
failure messages and good documentation — not just working code.

### Ask on the call
- [x] Who **runs** the deploy for a new customer — us, SRE, or a pipeline nobody watches? — *SRE*
- [x] Who is on the hook when it **fails in production**? — *SRE, on out-of-hours grounds*
- [ ] What does SRE need from us to accept the handover?
- [ ] Is there **pairing or review support** available while we build it?

</details>

### Related
[[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] · [[APIM IaC - rollout and testing strategy]]

---

## Q11 — How do we get this across environments quickly, and what's the MVP? 🔴 **new — now the biggest**

**Status:** 🔴 open — **this is the question Sam actually wants answered.**
**Ask:** Sam Clifford · Andy L · Marcus.

> **In one line:** the code working on the sandbox proves very little — what's the shortest path to
> it working on a real customer environment, and what's the smallest version worth shipping?

**Where it came from:** Sam, 2026-08-07, as his closing question. It was the strongest theme of the
whole call.

**Why it matters — the AI Agent lesson, in his words:**

> *"The IaC was written but never really tested in anger… developers building locally, testing
> locally, merging, under the illusion these things were working."*

That's the failure mode we're currently set up to repeat. Today we have component unit tests with
mocked Pulumi, plus manual end-to-end checks at PR time. What's missing is **automated tests against
deployed infrastructure**.

**What makes it hard — the problems that only appear off-sandbox:**

- The **target hub may lack capacity** and Azure simply rejects the deploy
- **Internal and bespoke environments have different shapes** to the sandbox
- ⭐ **The sandbox is Pulumi-built, while other internal environments are legacy Terraform** — so
  the sandbox is not representative of what we're deploying into

**The proposed direction:** sandbox → internal → customer stage, **as early as possible**. Sam wants
an **internal stage** environment in the path, with **Dayshape internal as the candidate** — it's
real production, non-critical, and *"we're the best people to experience that pain."*

Stage is roughly equivalent to production, so it's the meaningful test.

### Ask
- [ ] Is **Dayshape internal** the agreed first non-sandbox target?
- [ ] What does an **MVP** look like — which phases must be in it?
- [ ] What automated testing against deployed infrastructure is realistic, and who's done it before?
- [ ] Which internal environments are **Terraform vs Pulumi**, and does that block us?

### Related
[[APIM IaC - rollout and testing strategy]] — the full write-up

---

## Q12 — Deploy sequencing: Core first or APIM first? 🟠 **half answered**

**Status:** 🟠 **partial — the ordering is settled, the customer-experience half is not.**
**Ask:** Richard · SRE.

> [!success] The ordering half is answered — APIM deploys first
> Settled by building Phase 5, not by discussion. Core holds a **copy** of the HMAC signing secret,
> so APIM has to exist and hold the master copy before Core can be given it. See **G23**.
>
> This is now a property of the code, not an open choice.

> [!danger] The half that is still open is the one that affects customers
> *"What should a customer's integration script see while we're mid-deploy?"* remains undecided,
> and it is the more important half. APIM keeps serving throughout a Core deploy (**G18**), so a
> customer's automation can be talking to a half-upgraded system without knowing it.
>
> Tracked for sign-off as **D10**.

> **In one line:** during an upgrade there's a window where APIM and Core are on different versions
> — which one should move first, and what should callers see meanwhile?

**Where it came from:** Sam and Richard, 2026-08-07.

**Why it matters:** APIM's users include **customers' automated integration scripts**. A person
hitting a broken page retries; a script may fail the job, retry in a loop, or drop data. Core has a
maintenance mode and a VM disable — **APIM has no equivalent** and keeps serving throughout.

Sam's framing: *"You guys need to figure out who your users are. We don't want outages."*

Richard's version: *"How do we tell the customer's script that it needs to have an outage from now
to now while we update everything?"*

**Options floated, none decided:** a meaningful **503** from APIM during a deploy; a **maintenance
mode for the API**; **versioning headers** as Reporting does *(Sam: probably out of scope — needs
Core API changes)*.

> [!warning] Already real
> Richard mentioned a recent incident where **cache headers** left a customer believing Stage was
> down for a day after the outage had ended. Same class of problem.

### Ask
- [x] Core first or APIM first? — **APIM first**, settled by G23
- [ ] What should a customer integration see during the window? → **D10**
- [ ] Is a maintenance mode for the API worth building, or is a 503 enough? → **D10**

### Related
[[APIM IaC - known traps and gotchas]] G18 · [[APIM IaC - OpenAPI source of truth (phase 7)]] §7.6

---

## Related


- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — source of the 2026-08-07 answers
- [[APIM IaC - rollout and testing strategy]]
- [[APIM IaC - ticket breakdown]]
- [[APIM IaC - known traps and gotchas]]
