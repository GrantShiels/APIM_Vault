---
tags: [apim, iac, meeting-notes, devo-28368]
updated: 2026-08-18
date: 2026-08-18
attendees: [Grant Shiels, Sam Clifford, Marcus Thomson]
related-ticket: DEVO-28368
status: held — one answer changes the design, see A2
---

# Meeting Notes — Sam Clifford & Marcus Thomson, 2026-08-18

34 minutes. Called to sort out release-pipeline questions; ended up answering the thing that
had been blocking Dreich for two days.

> [!important] The headline
> **A2 below removes the need to write to an Azure DevOps variable group at all.** That single
> answer resolves both the ordering problem (G40) and the permissions wall (G42), and it is
> already the established pattern in three other components. Everything else here is secondary.

---

## A1 — Building the OpenAPI spec: approach confirmed

Grant's plan — generate the OpenAPI JSON during the release pipeline, publish it as an
artefact/blob, and have the orchestrator consume it — was put to both of them and **accepted
without objection**.

Sam: *"what you've described, building the open API spec, publishing it to a blob store, and
then consuming it in IaC… sounds good to me."*

The reasoning behind it, in Sam's words: the team has deliberately chosen to **bake as much as
possible into the orchestrator**. If something infrastructure-related is being deployed, it is
the infrastructure that manages it. Language packs were deliberately coupled this way so that
software deployment sits as close to infrastructure deployment as possible.

He was candid that this is *"a bit of an anti-pattern… it's really what CD pipelines are
intended for, but we aren't really using them for that purpose."* Worth knowing — the pattern
we are copying is a known compromise, not an ideal.

**Follow-on:** modelling this on the language pack flow is the right instinct and is what he
expected to hear.

---

## A2 — ⭐ The Core app can read App Configuration directly

**This is the important one.**

Grant explained the ordering problem: APIM generates the signing secret, Core needs it, so
Core has to deploy after APIM.

Sam's answer:

> *"You might be able to do that without being dependent on the core app. So the core app
> supports reading configurations from app config. So you could add a stateful app config as
> part of your APIM component, and then the core app would just read that as and when it needs
> it. In theory."*

Asked for an example to follow:

> *"Take a look for app config key set. It's like an aggregate resource I built for allowing
> you to add many app configs. [They] used it recently, in language packs or something."*

### Verified afterwards — it holds up

Checked against the codebase rather than taken on trust. See **G43** in the gotchas note for
the full evidence. Summary:

- The resource is `Modules/AppCfg/infra/app_cfg_key_value_set.py` — `AppCfgKeyValueSet`.
- **Three** components already use it to hand values to the Core app, none of which touch a
  variable group: LanguagePack, Solver and CognitiveSearch.
- Core genuinely does read App Configuration at runtime —
  `ConfigurationFactory.cs` calls `ConfigureAzureAppConfiguration("spk-core", …)`.
- It also resolves **Key Vault references**, so a secret never needs to sit in App Config as
  plain text.

**One correction to Sam's phrasing:** it is read at **application startup**, not continuously.
There is no refresh configured for ordinary settings, and the `IConfiguration` is registered as
a singleton specifically to keep settings stable for the lifetime of the app. That is still
enough to solve our problem, but "as and when it needs it" overstates it.

---

## A3 — Region availability is a known, unsolved platform gap

Grant raised that APIM Basic v2 is not available in every region, and asked where region
choice should live.

- Sam: the **hubs table in the golden source of truth** is where region information has
  started to be captured.
- Marcus: that would be the default; anything different would be stored in app config. He
  thinks this needs **a component that handles it properly**, and explicitly said *"this won't
  be on you, this will be on a platform"* — i.e. not Ecosystem's problem to solve.
- Marcus framed it as one instance of a wider category: *"is this available? No."* The same
  thing bites them with **virtual machines failing to deploy because there is no capacity**.

**Current process, per Sam, stated plainly:** you deploy using defaults, it fails with an
opaque error, you then add the config needed to make it work. He was not defensive about this
— *"we need to get away from SRE doing this"* — and offered that if it proves difficult, the
tooling burden is on the platform team, possibly by giving us validation capability.

Longer term he described wanting **config validation rules built into components**, with
resource availability as one of the checks. Hooks could do it in the short term, but he was
reluctant to add more there.

> [!note] Our position is fine
> Grant's approach — an explicit region config option, defaulting to the hub location — was not
> challenged. It is also close to what Marcus described as the intended end state.
> Relevant to [[D8 - Is West Europe acceptable for customer API traffic]].

---

## A4 — Retries are acceptable, but the flakiness itself is the real problem

Grant asked whether adding retries around the portal content script was acceptable practice.

**Yes, with a caveat.** Sam: avoid long sleeps, prefer **exponential backoff**, and accepted
the underlying reason without argument — *"Microsoft APIs are incredibly inconsistent. Their
resources are incredibly inconsistent, so we have to work around [it]."*

The caveat is sharper than the approval:

> *"Without understanding the full context, we do need to make that stable. It sounds like it's
> really flaky, and even with retry mechanisms you have, it could fail. So you will need to
> address that… Part of the problem I'm having today is flakiness [on the AI agent], and
> questions are being asked."*

**Read this as a warning, not a nicety.** Retries are a mitigation, not a fix, and flaky
deploys are actively causing the platform team grief elsewhere right now. He suggested pulling
the content into version control or building an artefact; Grant explained the reasons that has
been ruled out (documented in [[APIM IaC - portal content, why there is no IaC-native route]]).

**Action:** the current retry (3 attempts, 5/10/15s) is acceptable to ship, but exponential
backoff would match house preference, and the stability question deserves a proper look.

---

## A5 — There is no good story for "what does the customer see when it's down"

Grant asked how a customer hitting an endpoint should be handled when Core or APIM is
unavailable.

The honest answer is that **Dayshape does not handle this well today**:

- Core going down puts it into maintenance mode, but *"that doesn't mean the user will always
  see the maintenance mode screen"* — if the VM is shut down there is nothing to route to.
- Realistically the customer gets a **generic 503 from App Gateway, or a 404**. Sam: *"there's
  no nice handling of it."*
- Moving maintenance mode to a static web app has been discussed *"but it's never been a
  priority, so we've never looked at that."*

Two design steers worth keeping:

1. **Core should be the arbiter of whether something is down** — not APIM.
2. **Do not take APIM down as a whole because one backend is unavailable.** Thinking ahead,
   APIM could front several API services; one being down should not fail the rest. This
   matches what Gabriela had already asked about.

Sam also floated **blue/green deployment** for APIM to avoid a redeploy window at all, while
noting a full APIM redeploy is unlikely to be common.

Feeds [[APIM IaC - maintenance responses (design sketch)]] and
[[D10 - What customers see during a deploy window]].

---

## A6 — Release cadence, and why the secret renewal window is wrong

Grant asked how often the orchestrator runs. **Every release** — but the useful answer was the
one after it:

> *"Some customers take it every nine months, others will take it every eight weeks."*

**Nine months.** The current design models secret renewal on the AI agent's approach (renew
when under roughly four months remain, expiry a year or two out). On a nine-month-cadence
customer that is not safe.

Sam's steer on the principle:

> *"We need to make sure it's renewed regularly just in case customers take irregular releases…
> be quite aggressive with the renewal, but lax with the expiry."*

And on where it belongs: renewal *"would be sort of outwith the orchestrator"* — it is in there
now *"to make ends meet"*, not because that is the right home.

> [!warning] This needs revisiting
> A renewal process that only runs during a deploy cannot protect a customer who deploys twice
> a year. Either the expiry has to comfortably exceed the worst-case release gap, or renewal
> has to happen outside the orchestrator. Currently neither is true.

---

## A7 — SRE want to stop being the bottleneck, and want APIM moving fast

Unprompted, Sam laid out a change in how his team wants to work:

> *"We've become a bottleneck in development… We want to be facilitators rather than doing the
> actual work."*

His one ask for our plan:

> *"Think about how you can get APIM through various different stages up to a customer
> environment as soon as possible, and what the gates look like."*

Specifically: when can it be promoted to QA sites, which QA sites take it, when does it reach
Dayshape internal, how is it proven there, how does it reach a first customer environment, and
how do we get that customer taking updates regularly.

His reasoning is experience-based — *"from my experience on AI agent and all the other projects
I've worked on, the rollout always takes a backseat."*

**Action:** the rollout plan needs to be a first-class part of the delivery plan with explicit
gates, not an afterthought. Feeds [[APIM IaC - rollout and testing strategy]] and
[[D5 - Getting access to test on a real environment]].

---

## A8 — Review and assurance suggestions

For checking our own work:

- A **cloud architecture skill** in Copilot ("type cloud into Copilot") that applies the Azure
  **Well-Architected Framework** — five pillars including security. *(This is the
  `cloud-architecture-advisor` skill already in use in this repo.)*
- **Azure's own scanning** on the APIM resource, which flags issues by high/medium/low. Team
  convention: **action high, ignore medium and low.**
- SRE have someone who scans resources for standards adherence and actions the high items.

Also offered: Sam and Marcus are happy to review tickets asynchronously. Grant flagged he is
structuring the work into roughly six or seven pieces and will send them over.

---

## Actions

| # | Action | Owner | Notes |
| --- | --- | --- | --- |
| 1 | Replace the variable group write with an App Config entry | Grant | ⭐ Unblocks Dreich. See **G43** and ticket **T6** |
| 2 | Revisit secret renewal cadence against a 9-month release gap | Grant | A6 — current design is unsafe at that cadence |
| 3 | Add explicit rollout gates to the delivery plan | Grant | A7 — Sam's only real ask |
| 4 | Switch portal content retries to exponential backoff | Grant | A4 — and look properly at the flakiness |
| 5 | Send ticket breakdown to Sam and Marcus for review | Grant | Offered in the call |
| 6 | Run the WAF/cloud architecture review over the design | Grant | A8 |
| 7 | Region availability handling | *Platform* | A3 — Marcus explicitly said not ours |

---

## Related

- [[APIM IaC - known traps and gotchas]] — **G43** records the App Config verification
- [[APIM IaC - ticket breakdown]] — **T6** is the ticket A2 changes
- [[APIM IaC - HMAC signing secret decision (phase 5)]]
- [[APIM IaC - rollout and testing strategy]]
- [[D8 - Is West Europe acceptable for customer API traffic]]
- [[D10 - What customers see during a deploy window]]
- Transcript: `apim-release-pipeline-chat.txt` (session files)
