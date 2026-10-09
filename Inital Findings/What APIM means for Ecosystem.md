---
tags: [apim, ecosystem, impact, devo-28368]
updated: 2026-07-14
---

# What APIM means for Ecosystem

> Q3 of the Initial Investigation.

## Plain-language answer

We're taking on a capability that is **more Azure/DevOps-flavoured** than our usual codebase work.
There's an **existing IaC framework** APIM slots into (so we're not inventing the deployment
machinery), and an **older APIM component we can learn from** — but SRE have steered us to **build
our own APIM component from scratch** rather than finish the old one (see
[[Decision - do not build on existing APIM IaC]]). The job is to **build a repeatable APIM
deployment the framework way, close the manual gaps, and own it operationally**.

## What we're actually taking on

- **A new skills area for the team.** APIM lives in Azure (portal, Entra ID logins, Pulumi
  deployments). Neither I nor most of the team have deep APIM experience yet — hence this spike.
  Eve Raby (feature second) has some DevOps background, which helps.
- **A partly-manual process to improve.** Standing up one environment is **~0.5–3 days** today,
  following a 7-phase runbook (the [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)).
  Some of it is automated by the IaC; some steps are still hands-on.
- **Operational responsibility.** Once it's a wider product, we own day-2 concerns: monitoring,
  rollback, and knowing who fixes it when something breaks.

## What "done" looks like for 26.5

- Deploying APIM for a new customer is a **repeatable, low-touch** process (hours, not days).
- Config/docs come from a **single source of truth** (no manual drift between the app and APIM).
- **Monitoring/alerts** are standard, and there's a clear **SRE↔Ecosystem support model**.
- A recommendation on whether **full 26.5 launch is realistic** or whether we go **phased**.

## Near-term reality

- There's a **v2 APIM migration** in flight (with VDP/SRE) that we'd ideally be on **before** the
  wider launch — worth tracking as it affects build/deploy.
- The follow-on delivery is **already stubbed** as stories (DEVO-28369–28373) — our planning
  session turns these into a refined, estimated backlog.

## Why it matters

This is the difference between "we can do it once for one customer" and "we can offer it to the
**wider customer base** reliably". The spike's outcome shapes how much delivery work 26.5 needs.

## Related

- [[Why Ecosystem owns APIM]]
- [[APIM and SRE - understanding]]
- [[APIM Findings Summary]]
