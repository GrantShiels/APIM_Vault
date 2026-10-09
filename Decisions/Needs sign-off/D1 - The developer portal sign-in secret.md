---
tags: [apim, iac, devo-28368, needs-decision, security, auth, operations]
decision-id: D1
absorbs: [D1, D6]
updated: 2026-08-13
status: resolved
sign-off: Tech Lead · SRE (Richard Donkin) · Security · Product Owner
blocks: nothing
---

> [!warning] Audited 2026-10-07
> Built in DEVO-28373. Any "12 months" wording below is wrong: Entra issues a 2-year secret and the IaC replaces it every 30 days (`azuread.ApplicationPassword` + `time.Rotating`). Rotation only happens when a deploy runs.

# D1 — The developer portal sign-in secret

> [!abstract] One secret, two questions — both now answered
> The customer documentation site needs a password-like value to sign users in. There are two
> separate questions about it, and they are almost always discussed together and then only the first has an answer.
>
> | | Question | Status |
> | --- | --- | --- |
> | **Part 1** | How does it **get created and reach a real deployment**? | ✅ **Resolved** — built and proven live 2026-08-12, then **revised 2026-08-13** (see below) |
> | **Part 2** | Who **replaces it before it expires**, and how do they know it's due? | ✅ **Resolved 2026-08-13** — nobody. It renews itself |
>
> *This page previously existed as two documents, D1 and D6. They were combined on 2026-08-13
> because they concern the same value and the same sign-off group, and separating them was
> recreating exactly the failure Part 2 warns about.*

> [!success] ✅ The revised design is proven end-to-end on a real environment — grant-sbx, 2026-08-13
> All four things that had to be true were checked against live Azure after the deploy, not assumed:
>
> | Check | Result |
> | --- | --- |
> | APIM creates the secret on Core's sign-in record | ✅ `apim-devportal-signin`, created 13/08/2026 12:50 |
> | Auth no longer creates one | ✅ Its component ran clean, with no secret of its own |
> | The old value is properly cleaned up | ✅ Gone from the sign-in record; the secure-store copy `apim-devportal-client-secret` deleted too |
> | Running again does **not** churn the secret | ✅ Second run left it untouched — same creation timestamp. The 30-day clock is holding, exactly as intended |
>
> That last one matters as much as the others: a secret that is replaced on *every* deployment is as
> broken as one that is never replaced, because each replacement leaves a window where the site is
> using a value that has just been withdrawn.
>
> The APIM gateway itself had not finished building when this was written, for reasons unrelated to
> this decision ([[APIM IaC - known traps and gotchas|G37]]).

> [!warning] ⭐ Part 1 was revised on 2026-08-13 — please read this if you saw the earlier version
> The original decision (Option 1: **Auth** creates the secret and hands it to APIM through the
> secure store) was built and proven working. It was then **replaced by Option 2** — APIM creates
> the secret itself — for two reasons:
>
> 1. **The stated reason for rejecting Option 2 was wrong.** It cited a warning in our own code
>    about two components fighting over the same record. That warning turned out to be about a
>    *different* kind of record, and does not apply here. Detail below.
> 2. **Option 1 had a real flaw that Option 2 removes.** Auth and APIM are not deployed together as
>    a rule. Every time Auth ran, it created a fresh secret — and portal sign-in stayed broken from
>    that moment until APIM happened to be deployed again.
>
> The 2026-08-12 evidence is kept below as a record of what was built. Nothing about it was wrong;
> it proved the mechanism worked. We simply found a better place for it to live.

---

## Why this matters at all

Dayshape is giving customers a documentation website where their developers log in and read how to use our API. To let someone log in, the website needs a shared password with Microsoft's sign-in service. That value is the "client secret".

Three properties of it drive everything below:

| Property | Plain English |
| --- | --- |
| **Automation has nothing to copy from** | The manual instructions say *"copy this value and paste it there"*. There is no clipboard in a pipeline. |
| **Azure shows the value once, then never again** | Not even to the person who created it. So "read it back later" isn't an option. |
| **It expires after 12 months** | And when it does, **every customer's portal sign-in silently stops working**, with no alert and no obvious cause. |

The first two are Part 1. The third is Part 2.

---

# Part 1 — How the secret reaches production ✅ **RESOLVED**

> [!success] Option 2 — API Management creates the value itself, on Core's sign-in record
> **Revised decision, 2026-08-13.** Supersedes Option 1, which was built and proven on 2026-08-12.
> No human step, no new permissions, no new infrastructure, and no handover between components.

## What we chose, in plain terms

The API Management deployment **creates the password-like value itself**, attaches it to the
sign-in record it already uses, and configures the documentation site with it — all in one run of
one component.

**Nobody types anything, and nothing is passed between two parts of the system.** The value is
created and used within a few seconds of each other, by the same piece of automation, so the two can
never disagree about what it is.

### Why not simply keep Option 1, which already worked?

Because "worked" was measured on a deployment where **Auth and APIM ran together**. In normal
operation they do not.

| | Option 1 (built 12 Aug) | Option 2 (revised 13 Aug) |
| --- | --- | --- |
| Who creates the value | Auth | API Management |
| How it reaches APIM | Filed in the customer's secure store, collected at deploy time | It never travels — it is created where it is used |
| What happens if **Auth** is deployed on its own | ⚠️ A fresh value is created. **Portal sign-in is broken until APIM is deployed again** | ✅ Nothing. Auth no longer touches it |
| What happens if **APIM** is deployed on its own | ❌ Fails with an error if Auth has never run | ✅ Works |
| Parts of the system that must agree | Two | One |

The failure in row three is the whole reason for the change. It is silent, it affects customers, and
it would be triggered by an ordinary Auth deployment that nobody would connect to the portal.

## What the alternatives were, and why not

|                                                                  | Why not                                                                                                                                                                                                    |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Option 1** — Auth creates it, files it in the vault, APIM collects it | **This was originally chosen and built.** It works, but only while Auth and APIM are deployed together. Deployed apart, an Auth run silently breaks portal sign-in until the next APIM run |
| **Option 3** — APIM gets its own sign-in registration            | Architecturally the tidiest, but our own application would reject the resulting sign-ins (**G5** — confirmed from Core's source, see below). It is also by far the most work and nobody on the team has the Entra expertise |
| **Option 4** — a person creates it once, kept centrally          | Re-creates by hand the manual burden this whole piece of work exists to remove, and puts one shared value across every customer                                                                            |

> [!danger] ⭐ The mistake that led to Option 2 being rejected first time — corrected 2026-08-13
> The original page rejected Option 2 on this basis:
>
> > *"Two parts of the system writing to the same sign-in record is a known way to have them
> > overwrite each other. There is already a comment in our own code warning about exactly this."*
>
> **The warning is real, but it is about something else.** It concerns the list of **web addresses**
> Microsoft is allowed to return a sign-in to. That list *is* declared as a complete set — so if two
> components each declared their own version, each would wipe the other's out. That is why the
> addresses have stayed with Auth and are not part of this change.
>
> A **password** is not declared that way. Each one is managed on its own and leaves the others
> alone. **Our own AI Agent already relies on this**, holding two separate passwords on a single
> record without any conflict — and it already writes to *this very sign-in record* for a different
> purpose. The thing we were worried about is something we already do, in production, today.

> [!success] ⭐ Option 3 is ruled out on code evidence, not just reasoning — 2026-08-13
> The original page blamed **G9** and said Microsoft may not permit it. **That was the wrong
> reason** — the real blocker is **G5**, and it is more decisive:
>
> - `AuthenticationExtensions.cs` sets `ValidateAudience = true` with
>   `ValidAudiences = { ClientId, api://{ClientId} }` — **Core accepts only its own client ID.** A sign-in issued for a separate APIM registration would be rejected by our own application.
> - The obvious workaround is also closed: `IdpConfigurationProvider` has exactly **two** hard-coded  identity-provider slots, and both are already used by real customer sign-in providers. There is no third slot to register APIM into without a Core code change.
>
> Same outcome, sounder footing. It closes the "should we revisit Option 3 later?" question, which
> was previously left open as a possible future tidy-up.

## What was actually built

### Revised build — 2026-08-13

Five changes, all of which make the system **smaller** than the Option 1 version:

1. **API Management creates the value** as part of its own deployment, and configures both sign-in
   settings with it in the same run.
2. **It renews itself every 30 days.** See Part 2 — this is what closes the expiry problem.
3. **Auth no longer creates or files it.** The step that wrote it into the customer's secure store
   is gone, and so is the step on the API Management side that read it back out.
4. **Auth no longer depends on Storage.** That dependency existed *only* because Auth was writing to
   the secure store. Removing it puts Auth back exactly where it was in the deployment order before
   this work started.
5. **Auth still declares the web addresses.** Unchanged, and behind the same "is APIM switched on?"
   check as before.

**Net effect: one component does one job, instead of two components doing half each.**

### Original build — 2026-08-12 *(superseded, kept as a record)*

1. **Auth creates the value and files it.** Inside the existing "is APIM switched on?" check, so
   customers without APIM are completely unaffected.
2. **APIM collects it** just before its deployment starts. If the value is not there, the deployment
   **stops immediately with a clear error naming Auth and the store** — rather than carrying on and
   half-working.
3. **The running order was made explicit.** APIM now formally declares that it needs Storage and
   Auth to have run first.
4. **A correction we found while building:** Auth now writes into Storage's secure store, so *Auth*
   needed to declare Storage too. Without it, a brand-new customer environment could have run Auth
   before the store existed. The knock-on effect across all 19 components was measured rather than
   assumed — only Auth and APIM moved, and both moved *later*, never earlier. Nothing else changed.

*Points 1, 2 and 4 have since been undone. Point 3 remains, but for a different reason — see below.*

## The proof

> [!note] What this proves, and what it does not
> This was a live run of the **Option 1** design. It is kept because it proves the parts that did
> not change: the value can be created by automation, no human ever sees it, and the documentation
> site signs people in with it. **It is not evidence for the revised design**, which needs its own
> Dreich run before this page can be called finished.

This was not reasoned about — it was run. A live deployment on 2026-08-12:

| What we wanted to prove | What happened |
| --- | --- |
| The running order is real, not luck | Components were deliberately listed in the **wrong** order; the deployment silently corrected them to **Storage → Auth → APIM** |
| Auth genuinely creates and files the value | Auth run: **4 things created**, 19 unchanged — including the new value, filed in the customer's store |
| APIM genuinely collects it | Log: `Read 'apim-devportal-client-secret' from vault 'grantsbxspkstrkv'` |
| APIM actually *uses* it | Both sign-in settings updated with the new value |
| The value is really in the store | Confirmed independently in Azure afterwards |
| Sign-in still works | Signed in to the developer portal successfully with the new value |

The hand-typed value was **removed from the machine entirely** before this run, so there was nothing
to fall back on. It worked without it.

## Two things worth knowing

> [!tip] No new permissions were needed — this was the big unknown, twice
> The original worry was that Auth would not be allowed to write into the secure store. It turned
> out the deployment already grants itself the necessary access on every run.
>
> The revised design raised the same question in a different place: **is API Management allowed to
> add a password to a sign-in record that Auth owns?** Checked on 2026-08-13, and the answer is yes,
> for two independent reasons. Every component in a deployment run signs in **once, as the same
> account** — there is no per-component identity to be short of a permission. And our **AI Agent
> already writes to this exact sign-in record** today, for an unrelated purpose. **Nothing had to be
> requested, and nothing had to be granted.**

> [!success] ⭐ APIM can now be deployed on its own again — this reverses an earlier warning
> The Option 1 design made "deploy APIM by itself" an invalid thing to do: if Auth had never run,
> the value was not in the store and the deployment stopped with an error.
>
> **That failure mode no longer exists.** APIM creates the value itself, so it has nothing to wait
> for.
>
> Auth is still listed as needing to run first, but the reason has changed from a **hard
> requirement** to **getting the order right**: Auth declares the web addresses Microsoft is allowed
> to return a sign-in to, and those are a **one-time** setup. They do not expire and they do not
> change, so Auth does not need redeploying alongside APIM ever again.

## The changeover — a one-time note

This only matters **once**, on the deployment that moves an existing environment from the old design
to this one. After that it never applies again.

> [!question] "Do we need to run APIM before Auth?"
> **No.** Asked and checked on 2026-08-13. The running order stays exactly as it is — **Auth, then
> APIM** — and nothing needs to be done by hand.
>
> *(An earlier draft of the plan said "APIM first, then Auth". That was wrong. It would have meant
> deliberately working against the declared order for no gain.)*

**What actually happens on that one deployment:**

1. **Auth runs first** and removes the old value it used to create, along with the copy in the
   customer's secure store.
2. **APIM runs second**, creates the replacement, and writes it into both sign-in settings.

Between those two steps, the sign-in settings still hold the old value — which has just been
deleted. So **signing in to the documentation site fails for the few minutes between the two**, in
the middle of a deployment window. Then it fixes itself.

> [!danger] Step 1 will fail the first time unless a flag is cleared first — see G36
> Found on the first real run, 2026-08-13. The old value **cannot be deleted**, because it was
> hung off the sign-in record and quietly inherited that record's "do not delete" safeguard. The
> deployment stops with `cannot be deleted` before it does anything.
>
> Clearing it is a **one-off manual step per environment**, run *before* the deployment. It is only
> needed where the old code actually ran — in practice **grant-sbx and possibly Dreich, nothing
> else**, because the flag that switched the old behaviour on is off by default everywhere and this
> has never been released.
>
> The full command and the alternative (a two-pass release that avoids touching deployment state by
> hand) are in [[APIM IaC - known traps and gotchas|G36]].

> [!warning] The thing to actually avoid
> Not the order — **deploying Auth without APIM on the changeover.** Do that and sign-in stays
> broken until APIM next runs, which could be weeks. That is the old flaw happening one last time on
> the way out.
>
> **Deploy the two together.** On Dreich that means `Auth,ApimSpike`, not `ApimSpike` alone — partly
> because naming one component does not pull its dependencies in
> ([[APIM IaC - known traps and gotchas|G34]]), and mainly because Dreich's sign-in record needs the
> rename that only Auth performs ([[APIM IaC - known traps and gotchas|G35]] — the real reason the
> earlier run failed).

**If a few minutes of downtime is ever unacceptable** (it is not on a sandbox, and probably not
during a customer's planned upgrade either), there is a zero-gap route: deploy **APIM on its own
first** so the new value exists and is in use, then deploy normally so Auth tidies up the old one.
This works *only because* APIM no longer needs anything from Auth at deploy time. It is an option,
not the recommendation.

## Answers this settled

- [x] **Which option** — 1, 2, 3, or 4? → **Option 1** chosen and built 2026-08-12; **revised to
      Option 2** on 2026-08-13 when the objection to it was found to be mis-attributed
- [x] **Is a single shared secret across all customers acceptable?** → **each customer gets their
      own**, created fresh in their own environment. This falls out of the design rather than being a
      separate choice. It matters for Part 2 — see below
- [x] **If Option 3, who confirms it's possible?** → not pursued; ruled out on code evidence
      (**G5**, confirmed 2026-08-13)
- [x] **Is API Management allowed to write to Core's sign-in record?** → yes; confirmed 2026-08-13
      from the deployment pipeline and from existing AI Agent behaviour

---

# ⚙️ The constraint that connects both halves

> [!important] APIM holds a **copy** of the secret, not a pointer to it
> This is the single most important technical fact on this page, and it is why Part 2 is harder than
> it looks. Found in the code 2026-08-12.

Phase 5's signing secret is stored in the vault and APIM is given a *pointer* to it
(`apim.NamedValue` with an **unversioned** Key Vault address). APIM never holds a copy, so replacing
that secret reaches APIM **with no redeployment at all**.

The two Phase 4 resources that need *this* secret — `apim.AuthorizationServer` and
`apim.IdentityProvider` — take a **plain value**. There is no Key Vault option on either.

| Consequence | |
| --- | --- |
| **APIM holds its own copy** | So the copy has to be kept correct |
| **Replacing the value needs an APIM deployment** | Unlike the signing secret |
| **"Store it in the vault" is not the whole answer** | The vault cannot be the source at all — see below |

**This is the fact that decided Part 1.** Because API Management holds a copy, whoever creates the
value and whoever writes the copy must run **at the same time**, or the two drift apart. The
revised design achieves that by making them the same component. Option 1 could not, which is why it
was replaced.

It is also the fact that shapes Part 2: a replacement value only takes effect when API Management is
deployed. That is fine — API Management is the thing being released — but it is why "renew during
deployment" is the right shape of answer and a background job is not.

> [!success] ⭐ The possibility that would have removed this constraint — checked, and closed
> **2026-08-13.** The earlier version of this page floated an idea: the Phase 5 signing secret sits
> in the vault and API Management is given a *pointer* to it, so replacing it needs no deployment at
> all. **Could the sign-in settings be pointed at the vault the same way?**
>
> **No.** Checked directly against the Azure library we use. The Phase 5 mechanism accepts a vault
> address as a first-class option. The two sign-in settings accept **a plain text value only** —
> there is no vault option on either. The difference is in Azure's API, not in how we are using it.
>
> This closes the question rather than leaving it open, and it is what makes automatic renewal
> (Part 2) necessary rather than optional.

---

# Part 2 — Who replaces it before it expires ✅ **RESOLVED**

> [!success] **M3 — automatic renewal.** Nobody replaces it. It replaces itself.
> **Decided and built 2026-08-13, alongside the Part 1 revision.**
>
> Every deployment of API Management checks how old the value is. If it is more than **30 days**
> old, a fresh one is created — and the new one is created **before** the old one is removed, so
> sign-in is never interrupted.
>
> **No alert to build. No calendar reminder. No team to hand it to.**

**Why this became the obvious answer rather than the ambitious one:** it was only affordable once
Part 1 moved the value into API Management. While Auth created it, "renew it automatically" meant
renewing it in a component that is deployed on a different schedule to the one that consumes it —
all the coordination problems, plus a new one. Once the same component does both, renewal is a few
lines using tooling **already in this repository and already used by our AI Agent**.

## What this replaces

The rest of this section is kept as a record of the problem and the options considered.

> [!danger] The failure this was heading for
> - Customers try to log into their API documentation and **cannot**
> - Nothing else breaks — **the API keeps working perfectly**, so no alert fires
> - There is no error explaining why, and no obvious connection to anything that changed
> - Because each customer has their own value, it happens **one customer at a time, in an
>   unpredictable order**
>
> The first we would know is a customer telling us.

**Nobody owned this.** It was not assigned to a person, a team, or a process — it was recorded as a
known trap (**G7**) and that was the full extent of the mitigation.

Most expiring things break loudly. This one would have broken **quietly and partially** — the API
itself is unaffected, so every monitor we have stays green.

## ⚠️ The one thing this does *not* solve

> [!warning] An environment that is never deployed still drifts
> Renewal happens **during a deployment**. An environment left alone for longer than the secret's
> lifetime has nothing running to renew anything, and its portal sign-in will eventually stop
> working.
>
> **This is a real residual risk and should not be read as "solved".** What has changed is the size
> of it:
>
> | | Before | After |
> | --- | --- | --- |
> | Breaks after | Guaranteed, on a date nobody knows | Only if an environment goes **two years without a single deployment** |
> | Applies to | Every environment | Only dormant ones |
> | Warning signs | None | An environment nobody has deployed for two years is visible in other ways |
>
> Customers receive **5–6 version deploys a year** — roughly one every two months — so every one of
> them renews the value and resets the clock to its full life. The remaining life should never drop
> anywhere near expiry in a live customer environment.
>
> The exposure is therefore **long-dormant internal or trial environments**, not customers.

> [!note] The secret actually lasts two years, not the twelve months we ask for — see G38
> The code asks for a one-year life (`end_date_relative="8760h"`). Entra **silently ignores that
> setting** and applies its own default of **two years**. Confirmed by reading the credentials back
> after the first sandbox run: created 13/08/2026, expires 13/08/2028.
>
> This is not a fault and needs no fix — renewal is driven by the 30-day clock, not the expiry date,
> and a longer life is the safer direction. It only means the dormant-environment window above is
> **two years rather than one**, so the residual risk is smaller than first written.
>
> The old Auth-created secret behaved identically, so this is long-standing behaviour rather than
> anything the D1 change introduced.

## Why 30 days, and not "renew every time"

Renewing on every deployment would be simpler to explain, but development environments are deployed
**several times a day**. Each renewal leaves the old value behind until it expires, so the sign-in
record would steadily fill with dead credentials.

30 days is chosen so that in a customer environment — deployed roughly every two months — **every
deployment renews**, while a busy development environment renews at most once a month.

## The options that were considered

*Kept as a record. M3 was chosen.*

### Monitoring

**M1 — an alert some weeks before expiry**
Something watches the expiry date and warns in advance.
**For:** turns a surprise into a scheduled task · we'd know before customers do
**Against:** needs building, and needs somewhere the alert actually gets seen
→ **Not needed.** This was only ever a way to make sure a human acted in time. There is no human step left to be late for.

**M2 — a calendar reminder**
**For:** free, today
**Against:** depends on a person still being here and still looking · poor fit now that each customer
renews on its own date
→ **Not needed**, for the same reason.

**M3 — automatic renewal** ✅ **CHOSEN**
The system replaces the value itself before it expires.
**For:** removes the problem entirely
**Against:** originally judged "most work by a distance" — that judgement was made before Part 1 was
revised, and it turned out to be the *least* work of the three.

### Ownership

**O1 — SRE own it** — they own APIM after handover and cover out-of-hours
**O2 — Ecosystem own it** — we built it, but we don't cover out-of-hours
**O3 — whoever owns Core's registration owns it** — logical while we borrow Core's registration, but
it hands someone a responsibility they may not know they have

→ **None required.** There is no recurring task to own. The only residual item is the dormant-
environment risk above, which is a general housekeeping concern rather than something specific to
this value.

---

## ⭐ The hinge question — asked, answered, and then reframed

*Updated 2026-08-13.*

The renew-during-deployment approach is close to free, but has one weakness: **if an environment is
not deployed for a long stretch, nothing renews.** So everything turned on how often a quiet
customer environment actually gets deployed.

**Grant's answer:** customers receive **5–6 version deploys a year.**

That comfortably clears the bar. The maths:

- The secret lasts **12 months**
- Every deploy more than 30 days since the last one mints a fresh value, **resetting the clock to a
  full 12 months**
- At 5–6 a year — roughly every two months — **every deploy rotates**, so the remaining life never
  drops below about ten months

> [!success] ⭐ The Part 1 revision turned a hard question into an easy one
> The earlier version of this page raised a serious complication:
>
> > *"The rotation lives in whichever component mints the secret — that is **Auth**, not Core. So the
> > real question is not 'is the Dayshape application deployed 5–6 times a year?' but **'is the Auth
> > component deployed 5–6 times a year?'** We cannot settle this from the code."*
>
> That was the right concern, and it was genuinely unanswerable — the release pipeline has
> per-component toggles, so a release does not necessarily run every infrastructure component.
>
> **Moving the value into API Management dissolves the question.** The renewal now lives in the
> component that *is* the thing being released. The question becomes **"is API Management deployed
> when we ship an API Management release?"** — which answers itself.
>
> This was not the reason for the Part 1 revision, but it is a substantial free benefit of it, and
> it removes the open item Richard flagged.

---

## Appendix — how the rest of Dayshape already handles expiring secrets

*Researched 2026-08-12, extended 2026-08-13. **These are findings, not a recommendation.***

Rather than design something new, we looked at how every other part of our system deals with the
same problem. There are **five different approaches already in use** — which is itself worth
knowing, because it means there is no single house style to follow. Richard Donkin confirmed this
independently: *"seems like there's no standard"*.

| Where | How long it lasts | What replaces it |
| --- | --- | --- |
| **Workday sync** | Set to the **year 2100** | Nothing — it effectively never expires |
| **Service Principal module** | Left at the supplier default (~2 years) | Nothing |
| **AI Agent** (two values) | ~1 year | See the update below — **one is now rotated** |
| **Deployment logins** ⭐ | **10 days** | A **scheduled job every Saturday** replaces them all, with alerting attached |
| **Website certificates** ⭐ | 90 days | Renewed automatically **during a deployment** if fewer than **60 days** remain |

### The three that are genuinely relevant

**1. The scheduled weekly job** — `Dayshape.Secrets.Renew.yml`

Already exists, already runs **every Saturday evening**, already has alerting wired up, and already
does precisely the job we need: it walks a list of values, replaces each one, files the new version,
and tidies up expired ones along the way.

Its limitation is **scope**, not capability. It currently only covers the **logins the deployment
pipeline itself uses**, held in one central store per environment — not the per-customer stores
where our value lives. The machinery is proven and running; it would need pointing at a different
set of values.

**2. The certificate approach** — `min_days_remaining: 60`

Uses no schedule at all. Every time a deployment runs it asks *"does this have fewer than 60 days
left?"* and renews it if so. Needs **no new scheduled job and no new alerting** — it rides along
with deployments that are happening anyway. The catch is the exact mirror: **if an environment is
not deployed for a long stretch, nothing renews.**

**3. ⭐ The AI Agent bot secret** — added 2026-08-13, flagged by Jamie (Platform TL)

**This is the closest precedent we have found**, and it post-dates the original research. A merged
change now rotates the AI agent's Teams bot secret automatically:

- Rotation happens **at most every four weeks**, driven by a calendar-derived window rather than by
  storing any state or calling Azure to ask when the secret expires
- The new secret is created **before** the old one is deleted, so there is no gap
- Crucially, it explicitly opts the secret **out of the protection its parent app registration
  carries** — without that, the secret inherits the protection and cannot be rotated at all

**Why it beats the certificate approach as a model for us:** it is the *same kind of thing* (an
application password on a protected app registration), not a TLS certificate, and it solves the
same protected-parent problem we would hit. Our secret is attached to a protected registration in
exactly the same way.

> [!success] And now it *has* been copied — 2026-08-13
> The caution that used to sit here said the AI Agent pattern **could not simply be copied**, because
> the AI Agent mints and consumes its secret inside one component whereas ours crossed a boundary:
> Auth minted it → the vault held it → APIM baked a literal copy into two settings. Rotation would
> delete the old value and leave APIM presenting a dead one until redeployed.
>
> **The Part 1 revision removed the boundary.** API Management now mints and consumes the secret
> itself, exactly like the AI Agent, so the precedent applies directly and that objection is gone.
>
> ⚠️ *Correction: the earlier version of this page cited that objection as **G23**. That was a
> mis-citation — G23 is about the Phase 5 **signing** secret and the copy **Core** holds of it, which
> is a different value and a different problem. **G23 is unaffected by this change and remains
> open.***
>
> One deliberate difference from the AI Agent: it derives its rotation window from the calendar. We
> use the provider's own built-in mechanism, which arrived in a version of the tooling newer than the
> AI Agent work and does the same job with less code.

> [!warning] AI Agent's *second* value still has the gap
> The rotation covers the bot secret. Whatever is decided here probably ought to be applied to the
> remaining one too — or at least raised with whoever owns it.

### The open questions this leaves

- ~~**Does a customer version deploy actually run the Auth component?**~~ ✅ **No longer relevant.**
  Renewal moved into API Management, so the question is now "does an API Management release deploy
  API Management?" — which answers itself. *This removes the item that needed SRE.*
- **Does anything today watch for secrets nearing expiry and warn us?** We found **no such
  monitoring anywhere**. Not needed for this value any more, but worth knowing as a general gap.
- ~~**Is frequent replacement acceptable for a customer-facing value?**~~ ✅ **Addressed by design.**
  Replacement is invisible to customers: the new value is created before the old one is removed, and
  nobody is ever holding a copy of it except API Management itself.
- **Should AI Agent's remaining value be fixed at the same time?** Still open, but now clearly a
  separate piece of work owned by another team rather than something this decision should widen to
  cover.

---

## What we need from you

> [!success] Both parts are settled — this page is now for information and sign-off, not decisions
> The list below is what *used* to be needed. It is kept so anyone returning to this page can see
> what changed.

- [x] ~~**Who owns renewal**~~ — nobody. It renews itself (**M3**)
- [x] ~~**How do they find out it's due**~~ — no notification needed; there is no task to be late for
- [x] ~~Where should such an alert go so it's actually seen?~~ — no alert being built
- [x] ~~Is a customer-visible sign-in outage acceptable if renewal is missed?~~ — automated before
      launch, so the question does not arise for live customers
- [x] ~~**Does a customer version upgrade actually run the Auth component, or can it be skipped?**~~
      — no longer relevant; renewal moved out of Auth
- [ ] **Should AI Agent's remaining value be fixed at the same time?** — the only item still open,
      and it belongs to another team

### What we would still like a view on

- [ ] **The dormant-environment risk.** An environment not deployed for 12 months still drifts. We
      believe this is acceptable — it affects internal and trial environments, not customers — but
      it is a conscious acceptance rather than something solved, and worth someone else agreeing to.
- [ ] **Confirmation that a live deployment behaves as designed.** The 2026-08-12 proof was of the
      superseded design. The revised design needs its own run before this is signed off.

> [!note] Background — 2026-08-12
> Reached out to **Richard Donkin** and **Marcus** for a view, particularly the deploy cadence
> question and whether the existing weekly renewal job could be re-pointed. Richard replied that
> there is no standard, leaned toward the certificate-style approach, and said SRE should own it.
> Jamie subsequently pointed us at the AI Agent rotation described in the appendix.
>
> **Richard's instinct turned out to be right** — the approach built here is the certificate-style
> one (renew during deployment) rather than a scheduled job. The one part of his view that no longer
> applies is SRE ownership, because there is no recurring task left to own.

## Sign-off

**Tech Lead · SRE (Richard Donkin) · Security · Product Owner**

The Product Owner matters for Part 2 because the impact is **customer-visible** and the recovery
involves someone noticing.

Sam Clifford asked for auth to be discussed as a whole rather than piecemeal, so this is best taken
alongside the authentication overview.

---

## Decision

**Part 1 — Resolved 2026-08-12, revised 2026-08-13.**

- *Original:* Option 1 — Auth creates the secret and files it in the spoke Key Vault; APIM collects
  it at deploy time. Built and proven in a live deployment.
- *Revised:* **Option 2** — API Management creates the secret itself, on Core's sign-in record. The
  reason Option 2 was originally rejected turned out to be mis-attributed, and Option 1 carried a
  real failure mode: an Auth deployment silently broke portal sign-in until APIM was deployed again.

**Part 2 — Resolved 2026-08-13. M3, automatic renewal.** The secret renews itself every 30 days
during an API Management deployment, replacing before deleting so sign-in is never interrupted. No
owner, no alert and no calendar entry is required. Residual risk accepted: an environment left
undeployed for 12 months still drifts.

**Still to do before sign-off:** a live Dreich deployment of the revised design. The 2026-08-12
proof was of the superseded design.

---

## Follow-up actions

- [x] ~~Add the "APIM cannot be deployed alone" caveat to the release documentation~~ — **no longer
      true.** APIM can be deployed alone again; the caveat should *not* be added
- [ ] Delete the hand-made `apim-auth` value from the Dreich sign-in record once this is deployed
      there — it is now redundant
- [ ] After the cut-over deploy, confirm the old `apim-devportal-client-secret` entry is gone from
      the customer's secure store, and that the credential Auth created has been removed
- [x] ~~Update **G9** in the traps register~~ — superseded: the honest blocker for Option 3 is **G5**,
      not G9. Reconcile the register accordingly
- [x] ~~Extend **G23** to cover the portal sign-in secret~~ — **withdrawn.** This was a mis-citation:
      G23 concerns the Phase 5 *signing* secret and Core's copy of it, a different value. G23 is
      unaffected and remains open
- [x] ~~Verify whether `apim.IdentityProvider` can take a named value reference~~ — checked
      2026-08-13: **it cannot.** Neither sign-in setting offers a vault option
- [x] ~~Close **G7** (12-month expiry)~~ — closed by automatic renewal
- [ ] Raise the AI Agent's remaining un-rotated secret with the team that owns it

---

## Related

- [[APIM IaC - client secret decision]] — the full analysis Part 1 summarises
- [[APIM IaC - authentication (phase 4)]] — how sign-in works end to end
- [[D2 - The master documentation site]] — the other half of the launch blockers
- [[APIM IaC - known traps and gotchas]] — **G7** (expiry), **G9** (own registration blocked),
  **G17** (secrets visible in the portal), **G23** (stale copy after rotation)


Loose Notes From chat with Jamie: 
- Look at AI agent multiple tenant application, will have to have 
	  Use bot app, AIA-bot
	  Anyone from Microsoft auth, but we will only accept tokens from verified issuers. So we'd only accept tokens from the specific customer and Dayshape. So it's locked down to just the specific customers. 
	  Don't jump straight into this as you need to manage with the customer, they need admin consent. 
	  We'd need an additional authentication that would only be used by the proxy endpoints, 
- Clients own their own app registrations and entras so we can't access them and they need to raise a ticket for them to do the work and will take a along time
- 
