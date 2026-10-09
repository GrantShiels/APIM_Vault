---
tags: [apim, meeting-prep, iac, devo-28368]
updated: 2026-07-27
related-ticket: DEVO-28368
---

# Meeting prep — SRE / IaC + APIM call

> Prep for the call with two SRE engineers on the APIM IaC spike.
> Companion docs: [[APIM IaC - outstanding questions]] · [[APIM IaC - ticket breakdown]] ·
> [[Team skills vs APIM delivery - risk assessment]].

## Who's in the room

- **IaC engineer** — best placed on the orchestrator / pipeline / Pulumi-state questions (Q1, Q2, SKU).
- **DevOps engineer** — has set up APIM **manually** following the
  [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide),
  and is familiar with IaC. Best placed on the tacit "what the guide doesn't tell you" knowledge.

## Goal of the call (say this up front)

> *"We're at spike stage on APIM IaC — not building for real yet. I want to (a) sanity-check our
> ticket breakdown against how you've actually done this, (b) show a small PoC to confirm we're on the
> right track, and (c) get your steer on a couple of unknowns we can't answer ourselves. We'll be
> handing the finished IaC to SRE to run, so I especially want your view on what 'good' looks like for
> you."*

Setting the handover expectation early usually gets people engaged — we're building **for them**.

## What to show

1. **The ticket breakdown first** — [[APIM IaC - ticket breakdown]] (T1–T12 mapped to guide
   phases). Higher-value artifact: ask them to poke holes in the **phasing and completeness**.
2. **The PoC branch** (`spike/DEVO-28368-apim-investigation`, code at
   `Products/Spoke/ApimSpike`). Phases 1–2 done, Phase 3 (service policy) in progress. Frame as
   *"proof the pattern works"*, not a finished design. Don't spend the call reading code line-by-line.

---

## Questions for the **DevOps / manual-guide** engineer

Their gold is the knowledge that isn't written in the guide.

### D1 — "Walk me through your manual runbook — which steps are the fiddliest or most error-prone?"
- **Where it came from:** the whole spike exists to automate their manual process.
- **Why it matters:** the fiddly manual steps are exactly the ones most worth automating and the most
  likely sources of bugs. Feeds [[APIM IaC - ticket breakdown]] prioritisation.

### D2 — "Are there steps you always do that *aren't* in the guide?"
- **Where it came from:** our phase map (T1–T12) is built **from the guide**, so anything not in the
  guide is invisible to us.
- **Why it matters:** catches gaps in our ticket breakdown before we commit to it.

### D3 — "Which steps do you think genuinely *can't* be automated?"
- **Where it came from:** our working assumption is that only the **Developer Portal content (T10)**
  and the portal-publish step are truly hard to automate (they live in the portal's own store, not as
  clean Azure resources).
- **Why it matters:** confirms/​corrects that assumption — drives [[APIM IaC - ticket breakdown]]
  T10 and the residual-manual-steps story ([DEVO-28371](https://airtsdev.atlassian.net/browse/DEVO-28371)).

### D4 — "What rate-limit values and CORS origins do you actually use — and what `counter-key` does the throttle policy use (per-IP? per-subscription?)"
- **Where it came from:** our Phase 3 spike currently uses **placeholder** values (`100 calls / 60s`)
  because the guide leaves `<rate-limit-number>`/`<rate-limit-seconds>` as fill-in-the-blank variables,
  and we couldn't confirm the `counter-key` from the guide. See [[APIM test setup - my data points]]
  (both values are still ⏳ TODO).
- **Why it matters:** fills the two concrete blanks in the PoC and confirms whether rate limits should
  be per-client config (our current thinking — ties to T2/T5).

### D5 — "For monitoring/alerts, what thresholds do you use?"
- **Where it came from:** the guide references an alert set but **doesn't specify thresholds**; the
  monitoring story ([DEVO-28372](https://airtsdev.atlassian.net/browse/DEVO-28372)) is a known blank.
- **Why it matters:** we can't build alert rules without knowing the thresholds they rely on.

### D6 — "When you've deployed APIM into an environment that *already had* one, what happened?"
- **Where it came from:** [[APIM IaC - outstanding questions]] **Q1**.
- **Why it matters:** real-world colour on the re-run/adoption problem (see Q1 below).

---

## Questions for the **IaC** engineer

Our genuine unknowns. See [[APIM IaC - outstanding questions]] for the full write-ups.

### Q1 — Re-running when an APIM already exists (adoption)
- **Ask:** *"Is `pulumi import` the agreed way to bring a hand-built APIM under IaC management, and
  does our orchestrator support it? What should the default re-run behaviour be — preview-first?"*
- **Where it came from:** existing customer environments will often **already have a manually-created
  APIM** (from the guide). An APIM service name is **globally unique**, so we can't run a second one
  alongside — we must **adopt (import)** or **replace** it.
- **Why it matters:** getting this wrong risks a Pulumi "already exists" failure or, worst case,
  tearing down a live customer's front door. Gates the [[APIM IaC - ticket breakdown]] T7/T11.
- **Status:** 🟠 partial — solid theory (`pulumi import`), needs confirming against our orchestrator.

### Q2 — Updating just the OpenAPI spec without a full re-deploy
- **Ask:** *"Does the orchestrator expose `pulumi up --target` (or equivalent) so we can update just
  the API/spec without re-running the whole component?"*
- **Where it came from:** the OpenAPI spec **regenerates every Core release**, so it's the   fast-changing part; everything else barely changes. Grant made this a firm requirement.
- **Why it matters:** we don't want a routine spec bump to mean a full APIM deploy. Pulumi's  incremental diff already gives us most of this for free; `--target` would make it faster/lower-noise.
  **Same idempotency mechanism as Q1** — one question part-answers both.
- **Status:** 🟠 partial — an **acceptance criterion on T7**; only the *how* is open. Sam Clifford
  flagged the deeper version-alignment angle (spec must match deployed Core version) as his #1 build
  priority — see [[Meeting Notes - Sam Clifford on VP APIM IaC gaps]].

### Q4 — Cross-team handshake (Entra app reg + Core config)
- **Ask:** *"Two steps touch shared objects — the Entra app registration (T3) and Core's deployment
  config for the HMAC handshake (T4). If we build these as IaC, does SRE's execution identity have
  permission to write them, or do those stay gated/manual on your side?"*
- **Where it came from:** [[APIM IaC - ticket breakdown]] T3/T4 — both reach outside our team.
- **Why it matters:** "no manual step at runtime" is only achievable if the running identity has the
  right access. They inherit this, so agree it early.

### Q5 — Handover model
- **Ask:** *"Once we hand this over, how do you want to consume it — runbooks, rollback, preview-first
  as the default safety behaviour? What does a clean handover look like from your side?"*
- **Where it came from:** the SRE-handover story
  ([DEVO-28373](https://airtsdev.atlassian.net/browse/DEVO-28373)) and our "built for handover" principle.
- **Why it matters:** defines the finish line for the whole piece of work.

---

## Cross-cutting things to validate
- **"If you were us, what would you get wrong first?"** — open prompt; usually surfaces the best traps.

## Before you hang up

- **Agree one concrete next step** (e.g. them reviewing the ticket plan async, or a focused Q1 follow-up).
- **Capture answers live** against Q1/Q2/Q4 so we can move them toward ✅ afterwards.

---

## Answers captured (fill in during the call)

| Ref | Answer / decision | Follow-up |
| --- | --- | --- |
| D1 |  |  |
| D2 |  |  |
| D3 |  |  |
| D4 (rate limits + counter-key) |  |  |
| D5 (alert thresholds) |  |  |
| Q1 (adoption / import) |  |  |
| Q2 (--target / spec update) |  |  |
| Q3 (Basic V2 SKU) |  |  |
| Q4 (Entra + Core access) |  |  |
| Q5 (handover) |  |  |

---

## Related

### Tickets
- [DEVO-28368](https://airtsdev.atlassian.net/browse/DEVO-28368) — APIM IaC investigation spike (this work).
- [DEVO-28369](https://airtsdev.atlassian.net/browse/DEVO-28369) — deployment orchestration (Azure DevOps pipeline).
- [DEVO-28370](https://airtsdev.atlassian.net/browse/DEVO-28370) — OpenAPI / config source of truth.
- [DEVO-28371](https://airtsdev.atlassian.net/browse/DEVO-28371) — automate residual manual steps.
- [DEVO-28372](https://airtsdev.atlassian.net/browse/DEVO-28372) — monitoring & alerts.
- [DEVO-28373](https://airtsdev.atlassian.net/browse/DEVO-28373) — SRE handover & runbooks.

### Docs
- [[APIM IaC - outstanding questions]] — Q1/Q2/Q3 full write-ups.
- [[APIM IaC - ticket breakdown]] — T1–T12 mapped to guide phases.
- [[Meeting Notes - Sam Clifford on VP APIM IaC gaps]] — origin of the provisioning-timeout point.
- [[Team skills vs APIM delivery - risk assessment]] — the five delivery stories in plain terms.
- [[APIM test setup - my data points]] — where the rate-limit / counter-key blanks live.
- [[Decision - do not build on existing APIM IaC]] — the build-from-scratch steer.
- [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide) — the manual runbook we're automating.
