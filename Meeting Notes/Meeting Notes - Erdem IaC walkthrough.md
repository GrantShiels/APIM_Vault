---
tags: [apim, meeting-notes, iac, erdem, testing, devo-28368]
date: 2026-07-10
status: raw-to-tidy
---

# Meeting Notes — Erdem (IaC walkthrough)

> Notes from the call with **Erdem** about his IaC project.
> Context: [[Target IaC process - componentised framework]].

## Headline

- **Erdem is also early-doors** and doesn't know a huge amount about this yet either — he's been
  **working through the IaC Confluence docs** and **talking to Nick about Logic Apps.**
- Reassuring for us: even someone a step ahead is still ramping — this is **new ground across the
  department**, not just our team.

## What he showed / explained

- He **gave us a tour of how to use the IaC setup.**
- **Spokes are separate parts (components) that get deployed** — they behave **like microservices**
  (each component is independently deployable). APIM is one such Spoke component.
- **`ds-platform-sqlserver`** (Azure DB) **stores the details of all environments and what they
  need** — a central registry of env config the IaC uses.
  - Azure link: `ds-platform-db / ds-platform-sqlserver` (subscription `a6128f37-4d81-4832-961b-0f7078d1228b`).
  - ⏳ TODO: confirm exactly what APIM needs from this / how it reads it.

## Open question raised: how do we TEST this? ⚠️

- To validate our component we need to **run the orchestrator to stand up our own resource group in
  a sandbox-like environment.**
- This is a **key unknown** — needs answering before implementation. Take to **Andy L / Infra
  Platform**: how does an Ecosystem dev safely run the orchestrator against a sandbox to test the
  APIM component end-to-end? (Access, which sandbox, teardown, cost.)
- Note: framework also has **component unit tests + smoke tests** (see
  [[APIM component - orchestrator and config integration]]) — but Erdem's point is about **real
  end-to-end deploy testing**, which unit tests don't cover.

## Doc he shared — Container Apps Gotchas

[Building Container Apps (Gotchas / Questions)](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/5463801868/Building+Container+Apps+Gotchas+Questions)
— covers many issues he hit. **Mostly Container-App-specific (Email/Workday), so not all applies
to APIM**, but these parts transfer:

- **App Config vs environment variables** — rule of thumb: ~99% of config goes in **App Config**;
  env vars only for structural things known at deploy time. *(Directly relevant to fixing APIM's
  hardcoded config — DEVO-28370.)*
- **How to add a new AppCfg key** — via `component_configs.csv` (per-client) or in IaC code
  (`AppCfgKeyValueSet`) for values derived from Pulumi resources. *(This is exactly the mechanism
  APIM's CSV should use properly.)*
- **Build & push to ACR without a pipeline** (az acr login + podman) + restart to pick up new
  image. *(Useful for local testing loops.)*
- **Debugging a failed component / reading logs** (Serilog→blob, App Insights health workbook, raw
  container logs while "Activating"). *(General IaC debugging technique.)*
- ⚠️ Much of the doc (Container App Environment, Functions runtime vars, health probes, storage
  account as Functions "brain") is **container-app-specific and likely NOT applicable to APIM** —
  don't over-index on it.

## Actions / follow-ups

- [ ] Andy L: **how do we run the orchestrator to test in a sandbox** + what access we need.
- [ ] Confirm what APIM reads from **`ds-platform-sqlserver`** / the env registry.
- [ ] Ask Nick about Logic Apps only if it turns out relevant to APIM (probably not — flag).

## Related

- [[Target IaC process - componentised framework]]
- [[APIM component - orchestrator and config integration]]
- [[Meeting prep - Andy L and Erdem]]
