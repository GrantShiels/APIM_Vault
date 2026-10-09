---
tags: [apim, meeting-notes, decision, devo-28368]
date: 2026-07-09
status: raw-to-tidy
---

# Meeting Notes — Iain Macnab (outcome)

> Outcome of the meeting with **Iain Macnab** (department lead).

## Key decision ✅

- **APIM ownership is confirmed: it sits with Ecosystem.**
- The goal: **replace the current long, manual setup process with an IaC process** that is
  **a lot simpler** and repeatable.
- This resolves the open "who owns it" question — focus now moves from *whether* to *how*.

## Three follow-up actions given to me

1. **Talk to Andy L** — lead of the **Infra/SRE** team. (Ownership boundaries, how APIM deploys
   fit their pipelines/IaC, what support/pairing they can offer.)
2. **Talk to Erdem** — he's doing **something similar** on his project (likely IaC/deployment
   patterns we can reuse or learn from).
3. **Voodoo Park docs** — Gabriela pointed me to the
   [VDP SharePoint collaboration site](https://dayshape.sharepoint.com/sites/Voodoo-Park-Collaboration).
   ⏳ *Still getting access.*

## New resource found

- Confluence **[Infrastructure as Code (IaC)](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4873388047/Infrastructure+as+Code+IaC)**
  space **+ all child pages** — the likely blueprint for the target IaC process.

## What this changes for the spike

- Ownership question → **closed** (see [[Why Ecosystem owns APIM]], [[Team skills vs APIM delivery - risk assessment]]).
- Next technical focus: understand the **target IaC model** (from the Confluence IaC space +
  Erdem + Andy L) and how the existing `Products/Spoke/APIM` Pulumi component maps onto it.

## Related

- [[Team skills vs APIM delivery - risk assessment]]
- [[APIM and SRE - understanding]]
- [[Meeting Notes - APIM walkthrough]]
