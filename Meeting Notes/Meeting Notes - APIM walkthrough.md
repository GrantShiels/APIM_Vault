---
tags: [apim, meeting-notes, raw, devo-28368]
date: 2026-07-09
status: raw-to-tidy
---

# Meeting Notes — APIM walkthrough (raw)

> **Raw dump from the setup-guide walkthrough — not yet tidied.**
> Author of the setup guide ran us through the process.

## Ownership doubt (flagged in the meeting)
- **Manmeet is also confused why this sits with us (Ecosystem).**
- Action: **going to ask who actually owns it.** → see [[Why Ecosystem owns APIM]].

## What APIM is / how it looks (from the walkthrough)
- The APIM has a **service that shows all of the API endpoints and how they function** — accessible via the **API Management** window (Azure).
- The APIM **sits between Dayshape and the other services**.
- The **Products** section in API Management **bundles all of the APIs together**.
- Under **APIs → All operations** you can see the **Policies** and the detail of each API.
  - Viewable in detail as a web page in the portal, **and editable via code**.

## Key takeaway on current state
- This is **all set up manually via Azure — it has NOT been set up via IaC.**
- It **should be a separate component and be deployed like one of our normal services in pipelines.**
- **The ask is basically to automate the entire setup guide.**

## Open thread
- Several calls booked today to clarify **whether this genuinely sits with Ecosystem or not.**

---
### To tidy later
- Reconcile "not set up via IaC" against the existing `Products/Spoke/APIM` Pulumi component
  (exists in repo but apparently not used for real deployments → confirm status/why).
- Fold ownership findings into [[Why Ecosystem owns APIM]].
