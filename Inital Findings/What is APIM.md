---
tags: [apim, concept, devo-28368]
updated: 2026-07-08
---

# What is APIM?

> Q1 of the Initial Investigation.

## Plain-language answer

**APIM = Azure API Management.** It's a Microsoft cloud product that acts as the **"front door"**
for an API. It sits **between our customers and the Dayshape backend**, so customers never talk
to our app directly — they go through APIM first.

Think of it like the **reception desk of a building**: it checks who you are, points you to the
right place, keeps a log of visitors, and stops any one person from overwhelming the building.

## What it does for us (the jobs it handles)

- **Security / access control** — checks the caller is allowed in (login tokens, optional
  request signing) before anything reaches our systems.
- **A "front door" gateway** — routes each request to the correct Dayshape endpoint.
- **Traffic limiting (rate limiting)** — caps how many calls a customer can make, so one heavy
  user can't slow things down for everyone.
- **Developer Portal** — a self-service website for customers: "Getting Started" docs,
  interactive API documentation, and a "try it" tester. Login is via Microsoft (Entra ID).
- **Logging & monitoring** — records activity into Azure App Insights so we can see usage and
  troubleshoot.

## How it fits Dayshape specifically

- One APIM instance is set up **per customer environment**, alongside their Dayshape app.
- It mostly **points at existing Dayshape endpoints** (core app + reporting) rather than needing
  brand-new services.
- The API definitions come from an **OpenAPI file** in our codebase that updates each release —
  so the docs/endpoints stay in step with the app.

## Why it matters

Without APIM, we can't safely expose our API to customers at scale. It's the layer that makes the
API **secure, documented, throttled, and observable** — the prerequisites for selling/offering it
to the **wider customer base in 26.5**.

## Related

- [[Why Ecosystem owns APIM]]
- [[What APIM means for Ecosystem]]
- [[APIM Findings Summary]]
