---
tags: [apim, sre, handover, devo-28368]
updated: 2026-07-08
---

# APIM and SRE — understanding

> Q4 of the Initial Investigation.
> **Purpose:** build my own understanding of the SRE relationship so I can write a proper
> handover message / arrange meetings **later**. This is *not* the message itself.

## Where things stand today

- **SRE currently do (or validate) the actual APIM deployments.** Historically it's been a
  manual, hands-on Azure activity done with/by SRE for each environment.
- **Voodoo Park (VDP)** built the API and the APIM infrastructure-as-code; SRE has been the team
  putting it into environments and checking it works.
- Only **Baker Tilly** is live so far (3 environments). Deployments have been described as
  **~1 day per environment** in practice, with re-creates sometimes needed.

## Key people (my contacts)

- **Manmeet Kaur Bharaj** — main SRE point of contact for APIM/deployments.
- **#sre** Slack channel — general SRE questions.
- **infra-platform / Paul Elliot** — helped with related environment/reporting issues before.
- **Hugh** — mentioned around SRE deployment testing for the v2 migration.
- (Ecosystem side: **me** as acting TL, **Eve Raby** as feature second.)

## What the handover needs to cover (my checklist for later)

- **Ownership split** — which APIM setup-guide phases become **Ecosystem**, which stay **SRE**,
  which are **shared** (build vs deploy vs operate vs incident response).
- **Access & permissions** — what Azure/Entra access Ecosystem needs to deploy and operate APIM,
  and what stays gated behind SRE.
- **Day-2 operations** — monitoring, rollback, and break-glass: who does what when it breaks.
- **Current pain points** — which manual steps eat the most time / are riskiest (feeds the
  automation planning).
- **The v2 migration** — status and whether we should be on v2 before wider launch.

## How this maps to the backlog

This whole area is already captured as **DEVO-28373 — "formalise SRE↔Ecosystem handover and
operational runbooks"** (ownership matrix, runbooks, sign-off, break-glass/rollback owners).
JPD-988 explicitly calls out SRE liaison, so this isn't optional.

## My next action (later, not now)

- Arrange a **handover / kickoff conversation with Manmeet (SRE)** to walk the setup guide and
  agree the ownership split — then write it up in DEVO-28373.

## Related

- [[Why Ecosystem owns APIM]]
- [[What APIM means for Ecosystem]]
- [[APIM Findings Summary]]
