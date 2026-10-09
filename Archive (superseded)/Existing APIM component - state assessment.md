---
tags: [apim, iac, existing-component, state-assessment, devo-28368]
updated: 2026-07-14
source: "Products/Spoke/APIM (repo)"
---

# Existing APIM component — state assessment

> Ground-truth review of the code already in
> `Products/Spoke/APIM`. Context: [[Target IaC process - componentised framework]].

> [!warning] Superseded direction (2026-07-13) — see [[Decision - do not build on existing APIM IaC]]
> SRE (Ivo) has since said we should **NOT build on this component** — use it for **inspiration
> only** and **build from the ground up**. This note's "complete it, don't rebuild it" conclusion
> below is therefore **superseded**. The value of this note is now as a **reference / feature
> checklist** (which APIM resources are needed and how they map to the manual guide), **not** as a
> starting codebase.

## Headline

The existing component is a **strong, fairly complete first attempt** — **not** a rough skeleton.
It already automates **most of the 7-phase manual guide** in Pulumi/Python. This **reconciles the
contradiction** from the walkthrough ("not set up via IaC"): the IaC **exists and is substantial**,
but appears **never fully productionised or adopted into the release pipeline**, so real
deployments still happen manually via the portal.

## What it already does (maps to the manual guide) ✅

From `infra/apim_component.py` — the component provisions:

- **APIM service** (SKU, capacity, location, tags, client-cert handling) → *Guide Phase 1*
- **Service-level rate-limiting policy** (global) → *Phase 3*
- **Entra ID Identity Provider + Authorization Server** (dev-portal login + Try-It) → *Phase 4*
- **HMAC signing secret** as a Named Value → *Phase 5*
- **APIs from OpenAPI + default Product linking** → *Phases 6 & 7*
- **App Insights logger + per-API diagnostics** → *(logging part of monitoring)*
- **Self-hosted gateways** (Developer SKU only) → *local/dev use*
- **Cleanup of Azure's default Echo API / Starter+Unlimited products** → *Phase 2*
- **Developer portal publish** (only if `auto-publish-portal` = true)
- Exports `apim-name` and `resource-group`

That's genuinely most of the manual runbook already codified.

## Gaps / why it's still a "first attempt" ⚠️

1. **Config is not productionised** (`config/component_configs.csv`):
   - Hardcoded **pentest** values — `resources-baseurl = https://pentest25.st.dayshapetest.app/...`
   - Hardcoded instrumentation key and IDP client-id
   - **`INSERT-SECRET-HERE`** placeholder for the IDP client secret; **hmac-secret blank**
   - → Secrets must come from **Key Vault**, and per-customer values parameterised, before this is
     safe/repeatable. Feeds **DEVO-28370**.
2. **Only one API wired up** — CSV configures **v1 `resources`** only; reporting APIs (v1/v2) the
   README describes are **not** in the config.
3. **No monitoring alerts** — component creates logging (App Insights) but **no alert rules /
   action groups**. Alerting is a real gap → **DEVO-28372**.
4. **No tests** — the framework expects a `tests/` dir with unit tests; there are **none**. Feeds
   the framework's testing-strategy requirement.
5. **Manifest is barebones** — `ds_manifest.xml` is `0.0.1` with **no dependencies** declared;
   needs wiring for orchestrator/release-pipeline deployment → **DEVO-28369**.
6. **Residual manual steps remain** — portal publish defaults off; Entra redirect URIs; end-to-end
   verification → **DEVO-28371**.
7. **Redundant manual guide** — `manualsetup.md` in the component mirrors the Confluence guide
   (source-of-truth drift to resolve → **DEVO-28370**).

## Why this matters (as a reference checklist)

> Per [[Decision - do not build on existing APIM IaC]], we are **building from scratch**, not
> completing this code. Read the points below as a **spec/checklist for the new build** — what it
> must cover — not as "remaining work on this component".

- This inventory shows the **hardest technical part is well understood**: the full set of APIM
  resources the manual guide needs is already mapped in Pulumi terms here. That knowledge feeds the
  new build even though the code itself is not reused.
- The new build must still cover: productionise config + secrets (Key Vault), wire into the
  orchestrator/release pipeline, add reporting APIs, add alerts, add tests, and automate the last
  manual steps.
- It remains IaC/DevOps/Python work (real upskilling) — and, as a from-scratch build, the effort is
  **higher** than a "finish it" job would have been. See [[Team skills vs APIM delivery - risk assessment]].
- **Answered:** the earlier "complete it vs rebuild it?" question is now decided — **rebuild** (SRE
  steer). This note's lasting value is the resource inventory as the spec.

## How it maps to the epic tickets

| Ticket | Existing component status |
|--------|---------------------------|
| DEVO-28369 orchestration | Component exists; **not wired to orchestrator/pipeline** (manifest bare) |
| DEVO-28370 config source of truth | **Biggest gap** — hardcoded pentest/placeholder config + secrets |
| DEVO-28371 residual manual steps | Portal publish + Entra + verification still partly manual |
| DEVO-28372 monitoring/alerts | Logging present; **alerts entirely missing** |
| DEVO-28373 SRE handover | Aligns to Infra Platform pipeline ownership |

## Related

- [[Target IaC process - componentised framework]]
- [[Meeting prep - Andy L and Erdem]]
- [[Team skills vs APIM delivery - risk assessment]]
