---
title: APIM handover — start here
updated: 2026-10-07
status: living-doc
tags: [apim, iac, handover, devo-28365]
---

# APIM IaC — handover, start here

> [!important] Source of truth
> **The Jira tickets in epic [DEVO-28365](https://airtsdev.atlassian.net/browse/DEVO-28365) and the
> `iac-main` branch** (`Products/Spoke/Apim`, `Products/DynamicProviders/Apim`). Where a note in this
> vault disagrees with either, the note is wrong. Written 7 Oct 2026; check Jira for anything newer.

## What this is, in one paragraph

APIM (Azure API Management) is the front door that lets customers call the Dayshape Entities API and
read the documentation on a developer portal. We are turning a working proof of concept into
IaC (Pulumi, `iac-main`) that can switch it on for any customer hands-off. Nobody on the team had used
APIM before, so these notes capture what was learned.

## Ticket map (status on 7 Oct 2026)

| Ticket | What it does, in plain English | Status |
| --- | --- | --- |
| 28368 | The spike: proved it can be done | Complete |
| 28369 | Production APIM component (the empty gateway) | Complete |
| 28370 | Picks up the API definition from the Core build | Complete |
| 28371 | Sets up the API, its routing and the "deploy in progress" answer | Complete |
| 28373 | Developer portal sign-in and API access token setup | Complete |
| 28372 | Signs each call so Core knows it came via APIM | First Review — Core side exists; **APIM-side signing policy not in the branch yet** |
| 31025 | Checks customer access tokens, uses the signed-in identity for rate limits | In Refinement |
| 31021 | Rate limits per operation, read from the API definition | In Refinement |
| 31392 | Publishes both Entities V1 **and V2** (and later other APIs) from Core's artifacts | In Refinement |
| 30004 | Automatically copies the customer documentation site onto each new APIM | Ready for Development |
| 31423 | Moves the master documentation site to a permanent, owned home (split out of 30004) | In Refinement |
| 30005 | Monitoring and alerting (should be one of the last things done) | Ready for Development |
| 30006 | Rollout, handover, turning it on per customer (final MVP ticket) | In Refinement |
| 30623, 30734, 28416 | After 30006 / separate: local dev gateway; API auth/SSO MVP; link in Core app | Not part of the MVP path |

**Suggested order:** 28371 → 28373 → 28372 + 31025 → 31021 → 31392 → 30004 → 31423 → 30005 → 30006.
Blocker links in Jira are the real rules: 28371→28373; 28373→28372, 30004, 31025; 28372→31025, 30006;
31025→31021; 31021→30006, 30005; 31392→30006, 30005; 30004→30006, 30005, 31423; 30005→30006.

### Old numbering used inside the older notes

The notes below say "Ticket 1–8". Mapping: 1=28369, 2=28370, 3=28371, 4=28372, 5=28373,
6=30004 (+31423), 7=30005, 8=30006. The component was called `ApimSpike`; on `iac-main` it is `Apim`
(`Products/Spoke/Apim/infra/apim_component.py`).

## Current facts that older notes get wrong

- **Tier:** code is still `BASIC_V2`, but the plan is **Standard v2**, because the Application Gateway
  (INFRA-777, Richard Donkin spiking) must sit in front for WAF policy. Older notes say "Basic v2 for
  every customer". No epic ticket records the change yet.
- **Region:** a config setting that defaults to the hub region. Not pinned to West Europe.
- **Sign-in secret:** created and replaced by IaC every 30 days (native `azuread.ApplicationPassword`
  + `time.Rotating`). Entra actually issues 2-year secrets. Not a dynamic provider, nothing manual.
- **OpenAPI source:** the resolver reads the Core build artifact
  `openapi/<CORE_SOFTWARE_VERSION>/openapi.v1.json` in the `dsartifactrepo` storage account.
  DEVO-31392 moves this to per-API folders (`entities/openapi.v1.json` + `v2`).
- **Deploy-window answer:** only the "Core unavailable" 503 exists. The second response in the design
  note is not built, although 28371's text says both are.
- **Portal address:** `https://<client-id>-<env-short>-spk-apim.developer.azure-api.net`.
- **Master docs site:** still `vprk2-sbx-spk-apim` in a personal sandbox (Slack mentions
  `vprk-`; the real one has the `2`). Moving it is 31423. Azure can't rename an APIM; the options are
  move (keeps the name) or build new and copy (new address).
- **Dev portal "Try it":** keeps the implicit OAuth flow temporarily — see
  [[Decision - retain implicit OAuth temporarily for Developer Portal Try it]]. Customer developer
  accounts: [[Decision - customer developer accounts in our Entra directory]]. 30734 owns the
  long-term answer (it says "no guest users by default").
- **Environments:** a new sandbox once failed because more than one Application Gateway existed in the
  hub (temporary gateways deleted by Paul Elliot). Expect more of this until INFRA-777 lands.

## Still-open decisions (not in any ticket yet)

D2 (master ownership and the deploy's permission, now mostly 31423), D7 (Microsoft does not support
moving portal content between tiers; accept the risk), D8 (data residency of the gateway region),
D10 (what customers see during a deploy), Q8 (email scope consent). See
[[_Decisions needing sign-off - start here]] and [[APIM IaC - outstanding questions]].

## Gaps: lessons in these notes that are NOT yet in a ticket

| Lesson | Where in the vault | Suggested home |
| --- | --- | --- |
| A component missing from the environment registry is **silently skipped and the pipeline goes green**; name is case-exact (`Apim`) | prerequisites §2, gotcha G31 | 30006 |
| Dev CLI only registers in the dev registry; naming one component drops its dependencies (include Auth) | G29, G32–G34 | 30006 runbook |
| Name stays reserved after delete (hours / 48 h / up to 7 days — sources disagree); purge is irreversible | G37 | 28369 note / 30006 runbook |
| Secrets in APIM are readable by APIM portal users | G17 | 28372 |
| APIM must deploy before Core gets its copy of the signing secret | G23 | 28372 |
| Policy XML traps | G3, G4, G20 | 28372, 31021, 31025 |
| Upper/Lower service connections cannot read the master site; role needs Owner/UAA to create | G25, G30, D2 | 31423 |
| Microsoft doesn't support content migration across tiers | G26, D7 | 31423, 30004 |
| Change fingerprint not reproducible across machines | G28 | 30004 |
| Deploy identity can read but not change variable groups | G42 | 30006 |
| 30-day rotation depends on deploys running; dormant envs drift | G38, D1 | 30006 runbook |
| Standard v2 + Application Gateway change and its cost | Slack, INFRA-777 | new ticket or 30006 |

## Where to find things

- Hands-on: [[APIM IaC - how to run a Dreich test deploy]], [[APIM IaC - deploying to a new environment (prerequisites)]]
- Traps register: [[APIM IaC - known traps and gotchas]]
- Rules: [[APIM IaC - working rules for implementation]]
- Design notes: `Implementation Ideas/` (each carries a banner saying what is stale)
- Superseded history: `Archive (superseded)/` (old ticket breakdown, 1 Sep audit, old decisions)
- Meeting notes and transcripts: untouched.
- Slack: #automate-apim-deployment

> [!warning] Note names still say "Ticket 1–8" and `ApimSpike`
> Use the mapping above. Each note has a banner at the top listing its specific stale parts.
