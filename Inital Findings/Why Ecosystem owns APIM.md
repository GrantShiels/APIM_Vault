---
tags: [apim, ownership, ecosystem, devo-28368]
updated: 2026-07-08
---

# Why Ecosystem owns APIM

> Q2 of the Initial Investigation.

## Plain-language answer

APIM is the delivery vehicle for our **customer-facing integrations API** — and that API is an
**Ecosystem product**. Ecosystem already owns how Dayshape connects to the outside world
(integrations like GSS/Workday). A public API is the same kind of work: an **outward-facing
capability that customers consume**. So it's a natural fit for us to own it.

## The fuller picture

- The API was built as a project **with Voodoo Park (VDP)** as delivery partner; **SRE** has been
  doing/validating the actual Azure deployments so far.
- That was fine for the **first customer (Baker Tilly)**, but it doesn't scale — it's slow and
  depends on SRE's time for every environment.
- For a **wider launch (26.5)** the capability needs a **long-term owning team** that lives in the
  product/codebase and can drive it forward — not a one-off project team. That's **Ecosystem**.

## What we own vs what SRE keeps

- **Ecosystem (us):** the API + APIM as a **product** — how it's built, configured, made
  repeatable, documented, and evolved.
- **SRE:** the **platform underneath** — Azure subscriptions, infrastructure, and deeper
  operational/security guardrails. They become a **support/partner**, not the delivery team.
- The exact split is being formalised — see [[APIM and SRE - understanding]] and **DEVO-28373**.

## Why it matters

Clear ownership stops APIM from being "everyone's and no-one's". It puts the capability with the
team best placed to make it **repeatable and customer-ready**, and frees SRE from being a manual
bottleneck on every deployment.

## Related

- [[What is APIM]]
- [[What APIM means for Ecosystem]]
- [[APIM and SRE - understanding]]
