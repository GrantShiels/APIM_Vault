---
tags: [apim, iac, meeting-notes, devo-28368, devo-28370]
updated: 2026-08-07
date: 2026-08-07
attendees: [Grant Shiels, Richard Donkin, Marcus Thomson, Sam Clifford, Andy L]
related-ticket: DEVO-28368
status: held — notes written up, actions open
---

# Meeting Notes — Richard, Marcus, Sam C & Andy L, 2026-08-07

> Prep written 2026-08-06. Questions in priority order — Q1 is the one to make sure gets
> answered. Everything below the line is opportunistic if there's time.
>
> Context: this is the right room for the OpenAPI / version-alignment problem. Sam raised it
> originally, Marcus does the manual APIM setups today, Richard owns IaC review.

---

## Q1 — When does the orchestrator actually run? ⭐

**The question:** what triggers an orchestrator run, and how often does it happen for a given
customer?

Specifically:

- Does it run **every time a customer's environment is updated** (i.e. on every Core release
  to that customer)?
- Or only when **infrastructure** changes — new component, config change, new region?
- Is it ever run **manually / ad-hoc** by SRE?
- Can it run for a customer **without** a Core deploy happening at the same time — and vice
  versa, can Core be deployed **without** the orchestrator running?

### Why this matters (worth explaining in the call)

This directly determines how badly the OpenAPI version-alignment problem bites us.

| If the orchestrator… | Then… |
| --- | --- |
| runs on **every** customer update, alongside Core | Alignment is mostly self-correcting — the spec and the app move together. Version-pinning is a safety net. |
| runs **independently** of Core deploys | This is the real problem. APIM could be handed a spec for 26.5 while the customer is still on 26.4 — advertising endpoints that don't exist yet. Version-pinning becomes essential. |
| runs **rarely / manually** | APIM documentation goes stale between runs. The re-import problem (guide 7D) becomes a manual chore someone has to remember. |

Sam's original point, from #infra-platform on 20 July:

> *"IaC can be provisioned as part of a different lifecycle to software, so we need to make
> sure the spec that is used aligns with the core version that has been deployed."*

**I need the actual cadence to size this properly.** Right now I'm designing against an
assumption.

**Follow-up if the answer is "it varies":** is `CORE_SOFTWARE_VERSION` reliably set on every
orchestrator run, or only when a Core deploy is part of it? The LanguagePack hook depends on
that variable and fails hard without it — ours would too.

---

## Q2 — Can we add an OpenAPI publish step to the Core release pipeline?

**Aimed at:** Sam C primarily, Richard for the IaC half.

**What I'd propose.** One extra template call in
`pipelines/Templates/Core/Dayshape.Core.Release.yml`, immediately after the existing language
pack publish, uploading the OpenAPI spec to the artifact store under the version tag:

```
release/openapi/v26.5.0.0/openapi.v1.json
```

**Why it's smaller than it sounds** — three things already exist:

1. The release pipeline **already builds in Release configuration**, and the
   `GenerateOpenApi` MSBuild target runs `AfterTargets="Build"` in Release. **A fresh,
   correct spec is already being generated on the build agent** during every release build.
   Nothing new needs writing to produce it.
2. `Dayshape.LanguagePack.Publish.ps1` already does exactly this shape — takes a file from the
   checkout, uploads it to `dsartifactrepo` under `{versionTag}`, picking `release` vs
   `releasecandidate` from the branch. Ours would be shorter (no zipping needed).
3. `Products/Spoke/LanguagePack` already shows the consuming side — read
   `CORE_SOFTWARE_VERSION`, fetch the version-matched artifact, fall back to the repo file
   locally.

So this is **copy two existing patterns**, not invent anything.

**The bonus.** Generating on a build agent instead of a developer's laptop also kills the
timezone bug — the spec currently bakes in the timezone of whoever generated it, producing
~90 meaningless diff lines every time it's regenerated elsewhere. (Laszlo acknowledged this
on 17 July, fix promised in PR #6303 — **worth asking whether that ever landed**, it isn't on
`develop`.)

**What I need from the room:** is anyone opposed, and whose sign-off does a change to the Core
release pipeline need?

> [!note] Concrete breakdown if they ask "how much work?"
> Two new files (~40 line PowerShell script, ~18 line YAML wrapper) plus a 4-line call in
> `Dayshape.Core.Release.yml`. Under a day including a test run. Full detail in Part 7 of
> [[APIM IaC - OpenAPI source of truth (phase 7)]].
>
> **Also worth asking:** `Dayshape.LanguagePack.Publish.yml` is called from the HighVolume and
> FrontEnd build pipelines too — do those need the spec published, or is Core release the only
> one that matters?
>
> **And the dependency with a lead time:** the IaC needs *Storage Blob Data Reader* on
> `dsartifactrepo`. LanguagePack already needs this, so there's precedent — but it's an SRE
> grant and likely the long pole.

---

## Q3 — Marcus: the post-release OpenAPI update has never been done

Marcus said in #sre on 23 July:

> *"I don't know why that is there and we haven't used it in the past. We also haven't actually
> gone through a proper update to the OpenAPI doc after a new release either."*

**What I want to confirm:**

- Is that still true — has a post-release spec update been performed since?
- What's the *intended* process today, even if it's never been exercised?
- Guide section 7D warns that re-importing the spec **wipes the API policy**. Has anyone hit
  that in practice, or is it a theoretical warning?

**Why I'm asking:** if it's genuinely never been done, then guide 7D and section 12 are
untested procedure, and I should treat them as a design problem rather than a documented
process to replicate. Good news is it largely solves itself in Pulumi — the policy is declared
alongside the API, so re-running the deploy reapplies both.

---

## Q4 — Which file is the source of truth? (confirmation, not a debate)

Quick one to close out on the record.

The setup guide names **two different files** — `Infrastructure/openapi.v1.json` in the
variables table and section 12, but `Products/Spoke/APIM/openapi_docs/entities.json` in
Phase 7.

Everything points to `Infrastructure/openapi.v1.json`:

- **ADR-002 (Accepted)** says output is versioned at file level as `Infrastructure/openapi.v1.json`
- `nswag.v1.json` literally has `"output": "../../Infrastructure/openapi.v1.json"`
- `entities.json` is a hand-copy — generated by nothing, and ~5 weeks stale on `develop`
- `entities.json` is only referenced by the old `Products/Spoke/APIM/` component, which we've
  already dropped from our branch

**Ask:** anyone object to me treating `Infrastructure/openapi.v1.json` as the single source of
truth and raising the guide's Phase 7 wording as a correction?

---

## Q5 — Richard: the shared AppCfg change

During Phase 5 I added `delete_config_for_different_component()` to shared AppCfg code. It works, but it's a deviation in shared infrastructure that needs an Infra-Platform view beforethis goes anywhere near `develop`.

Also spotted a **latent bug** while in there — in app_cfg_interface._set_config_raw()`, the Key Vault branch sets `secret_id=config_value`, which is always `None` at that point. Not biting us, but worth someone owning.

---

## Q6 — Anything on `RequireApimOrigin`? (long shot)

`Dayshape.config` ships `RequireApimOrigin=true` with `ApimHmacSecret=REPLACE-WITH-HMAC-SECRET`, while the C# default is `false`. The filter fails closed, so on paper that config would 403 the whole Proxy API.

Was going to ask Anton, but if anyone in this room knows the history it saves a hop.

---

## Notes from the call

> Written up from the recording (`APIM Meeting overview.vtt`) plus notes taken live.
> Where the transcript was unclear I've flagged it rather than guessed.

> [!success] The three things that actually changed our plan
> 1. **The orchestrator runs on every customer deploy** — so the OpenAPI spec and the Core app
>    move together. Version alignment is real but self-correcting in the normal case.
> 2. **Sam endorsed the publish-artifact-then-consume design outright**, and framed it as an
>    enabler for blue/green deployments — a bigger prize than I'd pitched it as.
> 3. **The setup guide is AI-generated from the old IaC.** It is a starting point, not a
>    specification. This changes one of our standing working rules.

---

### Q1 — When does the orchestrator run? ✅ Answered

**Two triggers:**

| Trigger | What it is | When it's used |
| --- | --- | --- |
| `IaC Deploy` pipeline *(bracketed "versioning" — exact name to confirm)* | Manual run. You choose which components to deploy. | Sam's own sandbox testing; also the route used for production runs |
| Full software deployment | The same pipelines that deploy the Core app **also run the orchestrator** | Every customer version upgrade |

**How it decides what to deploy.** There's a **table in the Golden Source of Truth (GSOT)
database** listing which components are switched on for each customer environment. The
orchestrator reads that table and deploys everything enabled.

**It's a diff, not a rebuild.** If a component already exists it doesn't redeploy it — it checks
whether anything has changed and updates only if needed.

**Why the orchestrator is coupled to the Core deploy today:** the IaC updates image tags for web
apps so they match the version of the Core app being deployed. That's the existing precedent for
"infrastructure needs to know the Core version" — exactly the same shape as our OpenAPI problem.

> [!note] What this means for the OpenAPI design
> My assumption was right: customer takes a new version → deploy pipeline runs → orchestrator
> runs. So in the normal case the spec and the app move together, and `CORE_SOFTWARE_VERSION`
> will be set.
>
> The version-alignment work is still worth doing — it's what makes the *abnormal* cases safe,
> and it's what unblocks deploying APIM independently later. But it is **not** firefighting a
> live breakage.

**Timings discussed.** Basic v2 deploys in ~5 minutes; my full spike deploy from nothing is ~7
minutes. The older tier took ~an hour, which is where the guide's "4 hours" came from (really ~5
min of deploy plus an hour of manual configuration). Sam was satisfied that's fast enough.

---

### Q2 — Can we add an OpenAPI publish step? ✅ Endorsed

Sam: *"Yeah, yeah, absolutely."*

**He framed it as bigger than I did.** My pitch was "keep the spec aligned with Core". His view:

> *"Right now we deploy infrastructure alongside the core application because that's not where we
> want to be in a couple of years. In a perfect world we'd be able to do blue-green deployments
> and deploy some services in isolation. Publishing this artefact and then consuming it will
> unblock that sort of future vision."*

So the publish/consume split is a **step towards decoupling infrastructure deploys from Core
deploys**, not just a tidy-up. Worth leading with that framing when this goes to a PR.

**Copy the language pack approach.** Sam confirmed that's the model: *"Ideally you could copy
what I did for the language packs. I might have piggybacked off of that pipeline, I can't really
remember."* — he wrote it, and the piggyback is exactly what we found in the code.

> [!question] One nuance I need to resolve — decouple, or piggyback?
> Sam also said the OpenAPI generation *"shouldn't be built alongside the [core] project, it
> should be kind of its own CI pipeline so that we can build it and deploy it sort of separate."*
>
> That pulls slightly against "copy the language pack", which piggybacks on the Core release
> pipeline. My reading is that the separate CI pipeline is the **destination**, and piggybacking
> is the **acceptable first step** — same as language packs do today. But I should confirm rather
> than assume, because it changes where the new files live.

---

### Q3 — Post-release spec update, and migrating existing APIMs ✅ Answered

**Migration approach: destroy and rebuild.** Sam: *"That's what I would go with. Yeah, just
destroy it and rebuild it."*

Importing an existing resource into Pulumi state is possible but *"not a straight import — you
need to tweak it first so it aligns with the component."* Richard: *"pretty painful and manual.
If you can get it to be destroy and create, it's much easier."*

> [!warning] Marcus's operational constraint — this shapes the rollout
> **Azure holds onto an APIM name for a period after deletion.** You cannot delete and
> immediately redeploy under the same name.
>
> So the real-world sequence is: kill the old APIM → a few hours with no APIM → deploy the new
> one. Marcus suggested doing it in the afternoon and accepting the gap. With one customer on
> three environments, that's manageable.
>
> **Consequence:** we do *not* need to build a migration process. But we **do** need to test
> destroy-then-recreate before it happens at a customer, specifically for the name collision.

**Auto-destroy was considered and effectively rejected.** Richard's point: if Pulumi doesn't know
about a resource it will never touch it, so an "automatically clean up the old one" step would by
definition do nothing for a manually-created APIM. Deleting something Pulumi doesn't manage would
be a first for us.

**Lifecycle hooks exist if we ever need them.** The AI Agent component has *deployment hooks* —
before/after a deploy, and you can branch on whether it's a preview, an up, or a destroy. Sam has
used them to remove resources and then drop them from the state file. Precedent exists, but only
for Pulumi-managed resources. **Treat as a stretch goal — not worth time now.**

---

### Q4 — Which file is the source of truth? ✅ Confirmed

`Infrastructure/openapi.v1.json` — the one **not** in the APIM spoke folder. No objection to
treating the guide's Phase 7 wording as an error.

---

### Q5 — Shared AppCfg change ⏭️ Not covered

Ran out of time. Still needs an Infra-Platform view. Carry forward.

---

### Q6 — `RequireApimOrigin` ⏭️ Not covered

Ran out of time. Still one for Anton. Carry forward.

---

## Beyond the questions — things that came up unprompted

### 🔴 The setup guide is AI-generated

Sam, unprompted and with confidence:

> *"I am 99% sure that they built it from the IaC. So they built the IaC with AI, then they
> pointed AI at the IaC and said 'build me a doc', and then they handed it to us. The only human
> element is modifications me and Marcus have made to it. So you need to assume it's not
> something you should take verbatim. It's a starting point, but I wouldn't use that as the
> decision token for how you phase the work."*

**This is the single most useful thing I learned.** It explains every contradiction we've hit —
the two different spec paths, the phases that don't depend on each other, the steps Marcus says
have never been run.

**It also invalidates one of our standing working rules** (rule 5, "stay faithful to the setup
guide"). Updated in [[APIM IaC - working rules for implementation]].

---

### 🔴 APIM/Core version skew during a deploy

Sam raised whether we'd ever want two API specs live at once. After some back-and-forth the real
concern turned out to be narrower and more concrete:

> *"There's a race condition. If there's a breaking change to one of the endpoints, the API spec
> has to change at some point. So there will be a moment in time where APIM is newer or older
> than the Core app it's talking to. Any integrations interacting with APIM might fail during
> that window when otherwise they wouldn't."*

So it is **not** about supporting two versions permanently. It's about the deployment window.

**Ideas discussed, none decided:**

| Idea | Who | Notes |
| --- | --- | --- |
| Return a meaningful `503 Service Unavailable` from APIM during a deploy | Sam | So customer integrations know to back off and retry rather than failing oddly |
| An equivalent of **maintenance mode for the API** | Richard | Core already has maintenance mode + VM disable; APIM has no equivalent |
| Leverage **versioning headers**, stay backwards compatible, mark obsolete and redirect | Sam | What Reporting already does. Sam: *"Hard, I would say. Probably out of scope"* — needs changes to the Core APIs themselves |
| Decide the sequence — Core first then APIM, or the reverse | Richard | **Left unresolved.** Needs an answer before rollout |

Richard also mentioned a recent incident where a customer thought Stage was down for a day
because of **cache headers** still serving a maintenance response. Being fixed separately, but a
warning about how this class of problem presents.

> [!question] Open — who are our users?
> Sam pushed back on designing this before we know who consumes APIM. Two groups he named:
> a customer's **custom integration script**, and a **developer exploring the API**. How each
> reacts to an outage is different. He flagged that *"a whole piece of this work is how do we
> help customers build custom integrations"* — possibly out of our scope, but nobody owns it.

---

### 🟠 Auth — Sam wants the whole thing revisited

Sam: *"I would recommend revisiting your auth stuff as a whole and see if we get that locked
down a little bit."*

**He split the conversation into two halves**, which is a genuinely useful framing we didn't have:

1. **User authentication** — who uses APIM as an API service. Automated scripts (e.g. a customer
   integration seeding data), and developers exploring the API. Also covers machine-to-machine
   and APIM→Core. *Sam: this is the one to discuss first.*
2. **Management-plane access control** — who can get into APIM in Azure and change it.

**Secrets: the 1-year expiry is a live problem.** Nothing rotates them today. Two existing
options:

| Option | How it works | Sam's view |
| --- | --- | --- |
| Automated refresh task | Add the secret's ID to a storage account (blob store); a scheduled task refreshes it | Works, but only rotates — doesn't apply the new value anywhere |
| **AI Agent approach** | Vinny added this recently. Rotates the secret **and** applies it to the infrastructure that needs it | ⭐ *"That would get my vote"* |

Sam also suggested checking for a **secretless approach** — he had a quick look and couldn't find
one, but it's worth confirming properly.

**Secrets are visible in APIM.** Sam noticed the secret in APIM **isn't obfuscated** — you can
open it and read the value. We should look at what access developers have in APIM and whether we
need a read-only security group that can't see secret values.

**Still unresolved from our side:** Gabriela flagged that for production we had to add the
customer to **our own internal Entra tenant**, which we don't want to do. And there's something
about implicit authentication for the dev portal that nobody has documented.

**Plan:** write up the options with pros and cons, run past Sam / Richard / Marcus as engineers
and Gabriela as product. Sam: *"I'd be interested in seeing the Entra stuff. Might be worth
getting Infra-Platform written in as well — if Ivan is going to see it, he can coordinate."*

---

### 🟠 Testing and environments — the AI Agent lesson

This got more airtime than anything else, and it's a warning aimed directly at us.

> Sam: *"One of the problems we had with the AI Agent project was that the IaC was written but it
> was never really tested in anger. The developers were building the software locally, testing it
> locally, merging it, under the illusion that these things were working."*

**What testing exists today:**

| Layer | Status |
| --- | --- |
| Component unit tests (mocked Pulumi) | ✅ Exists — we already have 37 |
| Manual end-to-end at PR time | ✅ Exists |
| Automated tests against deployed infrastructure | ❌ Wanted, doesn't really exist |

Some incidental coverage comes from automated tests that happen to hit infrastructure when
deployed to internal environments (QAA4 was mentioned — *exact environment to confirm*).

**Get it across environments as early as possible.** Sandbox → internal → **customer stage**.
Problems that only appear once you leave sandbox:

- The target hub **doesn't have capacity**, and Azure rejects the deployment outright
- The shape of an internal or bespoke environment differs from sandbox
- **Sandbox is built with Pulumi; other internal environments are legacy and built with
  Terraform** — lots of small differences you can't anticipate

Stage is a slightly different shape again; production is close to stage. *"If it's working in
customer stage, it should be good."*

**Sam's suggestion — an internal stage environment.** He'd like demo environments moved into the
production group properly, and a separate internal stage created. **Dayshape internal is his
preferred candidate**: it's a real production environment (used for our own project management),
it's not critical, and *"we're the best people to experience that pain."* Richard asked whether we
could build an internal stage fully managed by us. Not a decision we own — one for SRE / Iain.

> [!question] Sam's direct question to us
> *"How do you get this across environments as quickly as possible, and what would an MVP look
> like? When would it be enough to actually have in a production site — even if that production
> site is Dayshape internal?"*
>
> This needs to be part of the feature delivery plan.

---

### 🟡 Who owns APIM after we build it?

Current pattern: **SRE**. If something's wrong with APIM as a cloud resource, SRE elevate access
in the management plane and fix it.

Sam: *"Big picture, should it be? Probably not, maybe not. But that's a bigger question."*
Richard: SRE are better placed for out-of-hours; business hours might be developers, out-of-hours
SRE — *"they should really be owning that."*

On production access: with PIM anyone could elevate and be audited, but production should stay
locked down. Customer developers would be expected to use **stage**, not production — Sam:
*"Why would they need production data? That'd be really scary."* Production access would likely
be limited to the automated integration itself.

---

### 🟡 How this work should be documented

Sam's steer, based on how AI Agent was done:

- AI Agent had **no feature delivery plan** — it was a tactical team taking existing IaC and
  making auth work. Documentation was **ADRs plus READMEs in the component**.
- There's a **`docs/` folder in the repo root** with ADRs for the AI Agent tactical work, plus
  others Marcus added recently. Worth reading as a model.
- **Architectural decisions → ADRs.** That's where "how are we doing auth, why are we moving from
  the old way to the new way" belongs.
- **Component documentation should grow with the component** and reflect what's actually there —
  a high-level README with diagrams. Sam has been using **Mermaid diagrams** *(transcribed as
  "angel diagrams" — assuming Mermaid, worth confirming)*. **That's where the diagrams Gabriela
  asked for should live.**
- A feature delivery plan is fine if Gabriela wants one — Sam thinks the last one was for
  Reporting Resilience.

---

### 🟢 Offer of help

Marcus: *"If you need more time with any of us, just reach out — you're not bothering us."*

Sam: his team is **shifting focus to DevEx over the coming months** — flag anything in our
process that's painful and they may be able to improve it.

---

## Actions

| # | Action | Owner | Status |
| --- | --- | --- | --- |
| 1 | Update the OpenAPI design doc with the answers from this call | Grant | ✅ Done |
| 2 | Amend working rule 5 — the guide is a starting point, not a specification | Grant | ✅ Done |
| 3 | Confirm whether the OpenAPI publish is a separate CI pipeline or piggybacks the Core release | Grant → Sam | 🔲 Open |
| 4 | Write the Entra/auth options doc — user auth **and** management-plane access, with pros/cons | Grant | 🔲 Open — blocks implementation |
| 5 | Get a dedicated meeting on the auth approach, incl. Gabriela and Infra-Platform (Ivan to coordinate) | Grant | 🔲 Open |
| 6 | Investigate the AI Agent secret-rotation component as the model for our secrets | Grant | 🔲 Open |
| 7 | Check whether a secretless approach to the APIM identity provider is possible | Grant | 🔲 Open |
| 8 | Look at dev access to APIM — secrets are readable; do we need a read-only group? | Grant → SRE | 🔲 Open |
| 9 | Decide the deploy sequence: Core first then APIM, or the reverse? | Grant → Richard/Sam | 🔲 Open |
| 10 | Define the environment rollout path and what an MVP in a production site looks like | Grant | 🔲 Open — for the delivery plan |
| 11 | Talk to SRE / Iain about an internal stage environment (Dayshape internal) | Grant | 🔲 Open |
| 12 | Test destroy-then-recreate of an APIM, including the name-hold delay | Grant | 🔲 Open — before any customer rollout |
| 13 | Read the AI Agent ADRs in `docs/` and model our documentation on them | Grant | 🔲 Open |
| 14 | Add a component README with Mermaid diagrams (covers Gabriela's ask) | Grant | 🔲 Open |
| 15 | Confirm Phase 2 default-config cleanup is unnecessary on Basic v2 | Grant | 🔲 Open |
| 16 | Collect DevEx pain points to feed back to Sam's team | Grant | 🔲 Open |

---

## Follow-up from Sam after the call

Direct message, 2026-08-07 11:07 — deliberately held back from the call.

> *"One thing you said that I didn't want to raise on the call: Auth component being updated to
> read a feature flag to conditionally add a redirect URI.*
>
> *We've done work on AiAgent to make the core app reg composable. You'll see in there we extend
> the core app reg for anything custom that's needed for AiAgent. Taking a similar approach here
> would be my preference. So, rather than having Auth component need to have an awareness of
> wider components, let the components decide if they need to extend any external components they
> relate to."*

Follow-up four minutes later:

> *"Taking a look at `ApplicationRedirectUris`, it might not be possible. Worth trying out anyway."*

**What this means.** Our Phase 4A approach has the **Auth component** carrying an `apim_enabled`
flag and generating APIM's redirect URIs. Sam's preference inverts that: **APIM should extend
Core's app registration itself**, so Auth doesn't need to know APIM exists.

**Status: parked.** Not acting on this yet — the auth approach needs a proper meeting first
(action 5). Recorded in [[APIM IaC - client secret decision]] so it isn't lost.

---

## Related

- [[APIM IaC - OpenAPI source of truth (phase 7)]] — the full write-up behind Q1–Q4, including the proposed plan (Part 7). **Updated with this call's answers.**
- [[APIM IaC - working rules for implementation]] — rule 5 amended after Sam's "the guide is AI-generated" point
- [[APIM IaC - client secret decision]] — where Sam's composable-app-registration follow-up is recorded
- [[APIM IaC - rollout and testing strategy]] — the environments / testing theme from this call
- [[APIM IaC - ticket breakdown]] — T7 unblocked by Q1's answer
- [[Meeting Notes - Sam Clifford on VP APIM IaC gaps]] — where the version-alignment point started
- [[APIM IaC - outstanding questions]]
