---
tags: [apim, summary, findings, devo-28368]
updated: 2026-07-14
---

# APIM Findings Summary

> Q5 of the Initial Investigation — a tight recap of the above.

## In three sentences

We built a customer-facing API; **APIM is the secure, documented, throttled "front door"** in
front of it ([[What is APIM]]). Standing it up today is **mostly manual (~0.5–3 days per
environment)** and leans on SRE, which won't scale for a **26.5 wider launch**. This spike hands
ownership to **Ecosystem** and works out how to make deployment **repeatable** ([[Why Ecosystem owns APIM]], [[What APIM means for Ecosystem]]).

## Key facts

- **Prior IaC exists (reference only):** a Pulumi component (`Products/Spoke/APIM`) already covers
  most of the setup, but SRE have steered us to **build ours from scratch**, using the old one only
  as a reference — see [[Decision - do not build on existing APIM IaC]]. It's also **broken against
  the current libraries** (references a retired Azure API version), proven when we ran it.
- **Manual baseline:** 7-phase [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)
  (Azure service create alone is a 30–45 min wait).
- **Live so far:** only Baker Tilly (3 envs). A **v2 APIM migration** is in flight and wanted
  before wider launch.
- **People:** VDP built it; **SRE (Manmeet)** deploy/validate; Ecosystem (me + Eve Raby) now
  owning it ([[APIM and SRE - understanding]]).

## What the spike must produce (per the ticket)

- Findings written up + recommendation on the **target rollout model** and whether a full
  feature-delivery plan is needed.
- **Phase-by-phase** map of setup-guide steps: manual vs automatable, owner (Ecosystem/SRE/
  shared), and measured duration per environment.
- Rough **effort estimates** + refined delivery tickets.
- Clear call: **full 26.5 launch feasible, or phased?**

## Follow-on backlog (already stubbed)

- **DEVO-28369** — automate deployment orchestration across environments
- **DEVO-28370** — align OpenAPI/config source of truth and remove drift
- **DEVO-28371** — automate residual post-deploy manual steps
- **DEVO-28372** — standardise monitoring and production alert provisioning
- **DEVO-28373** — formalise SRE↔Ecosystem handover and operational runbooks

## My open questions to resolve next

- Exact **manual vs automated** split per phase (needs a hands-on/deploy walkthrough with SRE).
- **Real timing** per environment (hands-on time vs waiting/handover time).
- Whether **v2** should land before the wider launch.
- Feasibility of **full 26.5** vs a **phased** rollout.

## Related

- [[What is APIM]] · [[Why Ecosystem owns APIM]] · [[What APIM means for Ecosystem]] · [[APIM and SRE - understanding]]
