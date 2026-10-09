---
tags: [apim, iac, framework, target-process, devo-28368]
updated: 2026-07-14
source: "Confluence: Infrastructure as Code (IaC) space + child pages"
---

# Target IaC process — Dayshape componentised framework

> What the "replace the manual process with IaC" actually means
> in practice, based on the internal [IaC Confluence space](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4873388047/Infrastructure+as+Code+IaC).

> [!note] Clarified 2026-07-14 — reuse the framework, rebuild the component
> The **framework, orchestrator and release pipeline are reused** (that's this note's point, and
> still true). But the **APIM component itself is built from scratch** — we do **not** finish the
> existing `Products/Spoke/APIM` code (see [[Decision - do not build on existing APIM IaC]]). Where
> older wording below says "finish/fit the APIM component", read it as **"build the APIM component
> fresh and fit it into the framework"**.

## The big realisation 🔑

We are **not building deployment automation from scratch.** Dayshape already has a mature internal
**"Componentised Framework"** for Infrastructure-as-Code, and APIM slots into it as a **Spoke
component**. The job is to **build our APIM component the framework way and fit it into the existing release pipeline** — not invent a new framework or pipeline. (The *component code* is a fresh build; the *framework around it* is reused.)

This **materially lowers the risk** from my earlier read (see update in
[[Team skills vs APIM delivery - risk assessment]]).

## How the framework works (plain terms)

- Components live under `Products/` → **Platform / Hub / Spoke**. APIM is a **Spoke** component.
- A **scaffolding script** (`new-spoke-component.sh`) generates the standard files for a new
  component — you don't hand-build the plumbing.
- Components are **Pulumi, written in Python**, with a set structure:
  - `infra/` — the Pulumi code (the cloud resources)
  - `tests/` — component unit tests
  - `hooks/` — lifecycle hooks run before/after Pulumi by the orchestrator
  - `ds_manifest.xml` — tells the orchestrator dependencies + deploy readiness
- An **IaC orchestrator** actually runs Pulumi for you; components are registered in
  `Products/Modules/Globals.py`.
- Dev is done in **VS Code + dev containers** (standardised setup).

## The release process (owned by Infra Platform)

`feature branch → PR → iac-main → RC tag → minor-release`

- The **Infra Platform team owns the release machinery** — `iac-main`, RC tagging, the
  "promotion PR" into `minor-release`, and the deploy pipelines.
- Tooling: `iac` CLI (creates version tags), **Build IAC Artifact** pipeline,
  **IAC Orchestrator Deploy** ADO pipeline, `create-promotion-pr`.
- **Versioning + rollback are built in** (versioned artifacts, tags).
- ⚠️ Branch from **`iac-main`**, not `minor-release`.

## What's still a real gap

- **Python** — the framework is Python; team writes C#/Vue. New language, learnable but real.
- **Pulumi / IaC concepts** — new paradigm for the team.
- **APIM-specific provisioning** — mapping the 7-phase manual guide into the component, incl. the
  fiddly bits (Entra config, portal publish) that may resist clean IaC → feeds **DEVO-28371**.
- **No APIM experience** in the team still stands — but Erdem + Andy L + VDP docs address this.

## How this maps to the epic tickets

- **DEVO-28369** (orchestration) → largely *use the existing framework + Infra Platform pipeline*,
  not build a new one.
- **DEVO-28370** (config source of truth) → fits the framework's config model.
- **DEVO-28371** (residual manual steps) → the genuinely APIM-specific automation work.
- **DEVO-28372** (monitoring/alerts) → provision within the component / with Infra Platform.
- **DEVO-28373** (SRE handover) → aligns to Infra Platform ownership of the release process.

## Related

- [[Meeting Notes - Iain Macnab outcome]]
- [[Team skills vs APIM delivery - risk assessment]]
- [[APIM Findings Summary]]
