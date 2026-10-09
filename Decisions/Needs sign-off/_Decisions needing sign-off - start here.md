---
tags: [apim, iac, devo-28368, needs-decision, index]
updated: 2026-08-13
status: living-doc
related-ticket: DEVO-28368
---

> [!warning] Status table audited 2026-10-07 — see [[_START HERE - APIM handover]]
> D1: resolved (built in DEVO-28373, Complete). D2: master move is now DEVO-31423; ownership and permission questions still open. D5: answered. D7, D8, D10: still open and not ticketed. D8's "West Europe" premise is outdated: the region is config, defaulting to the hub region.

# Decisions needing sign-off — start here

> [!abstract] What this folder is
> **Six decisions that cannot be made by the Ecosystem team.** Each needs a product, tech-lead,
> security or SRE call. Each is a short standalone document written for someone who hasn't followed
> the spike.
>
> They are not technical write-ups. They state what needs deciding, why it matters in business
> terms, the realistic options with honest trade-offs, and a suggested view — clearly marked as a
> suggestion. Every one ends with a blank **Decision** box.

**The headline:** all phases of work are built and the hard parts are proven against real Azure.
**What's left is overwhelmingly not code** — it's decisions and permission grants, and every one
needs somebody outside this team.

> [!info] Consolidated 2026-08-13 — ten documents became six
> Several decisions were tightly coupled: same subject, same people, same conversation. Splitting
> them was making them harder to act on, not easier. They are now single documents with **parts**.
>
> | Now | Was | Why merged |
> | --- | --- | --- |
> | **D1** — the sign-in secret | D1 + D6 | One secret. *How it's created* and *who replaces it* are the same conversation — and separating them was recreating the exact failure D6 warns about |
> | **D2** — the master documentation site | D2 + D3 + D4 | One instance. Who owns it, what permission the deploy gets, and where that grant lives are all the same SRE conversation |
> | ~~D9~~ — scope | *retired* | **Answered.** See below |
>
> **Old numbers are noted in each document**, so earlier notes and Slack threads still make sense.
> Numbers were deliberately *not* reshuffled — D5, D7, D8 and D10 mean what they always did.

---

## ✅ Already answered — scope *(was D9)*

**Question:** how much of the setup guide does every customer actually need?

**Answer from the Product Owner:** **all of it.** The guide was written as our step-by-step for
deploying APIM to customers, and the feature we deliver is a **hands-off orchestrator** that can be
run at any point when a customer decides to use the new API endpoints. We are not rolling APIM out
to everyone at once — customers will move across over the next year or so as they choose to.

> [!important] Why this answer matters to everything below
> There is no reduced MVP that removes the documentation site from scope. That means **D2 stays on
> the critical path**. Every decision in this folder is live.
>
> *(D1 was also on this list. It has since been resolved and built — it now needs a review rather
> than a decision.)*

---

## 📌 Take these in this order

### 1. Blocks everything

| | Decision | Sign-off |
| --- | --- | --- |
| **[[D5 - Getting access to test on a real environment\|D5]]** | Getting access to test on a real environment | SRE |

Nothing built has ever run outside one engineer's sandbox. This is the *"does it actually work?"*
question.

**Status:** in motion — a Dreich run is underway with Richard and Marcus.

### 2. Blocks launch

| | Decision | Covers | Sign-off |
| --- | --- | --- | --- |
| **[[D1 - The developer portal sign-in secret\|D1]]** | **Part 1** how the secret reaches production ✅ *resolved* · **Part 2** who renews it before it expires ✅ *resolved — it renews itself* | TL · SRE · Security |
| **[[D2 - The master documentation site\|D2]]** | **Part 1** who owns it ⬜ · **Part 2** what permission the deploy gets ⬜ · **Part 3** where that grant lives ⬜ | SRE · Security · PO · TL |
| **[[D8 - Is West Europe acceptable for customer API traffic\|D8]]** | Is West Europe acceptable for customer traffic? | PM · TL · data commitments |

**D8 is the odd one out** — it may need a compliance view rather than an engineering one.

### 3. Needed before launch, but not blocking build

| | Decision | Sign-off |
| --- | --- | --- |
| **[[D7 - Accepting an unsupported content migration\|D7]]** | Accepting an unsupported migration | PM · TL |
| **[[D10 - What customers see during a deploy window\|D10]]** | What customers see during a deploy | PM · TL · SRE |

These don't stop us building, and **cause incidents later if nobody answers them.**

---

## 👥 Grouped by who you need

| Who | Decisions |
| --- | --- |
| **Product Owner** | D2 *(Part 1)*, D7, D8, D10 |
| **Tech Lead** | D1, D2, D7, D8, D10 |
| **SRE** — Richard, Marcus | D1, D2, D5, D10 |
| **Security** | D1, D2 |
| **Data / compliance** | D8 |

> [!note] Two conversations would cover most of it
> **One with the PM** — D2 Part 1 *(ownership)*, then D7, D8, D10.
> **One with SRE** — D5 first, then D2 Parts 2 and 3.
>
> **D1 no longer needs a conversation to reach an answer** — both parts are resolved and built. It
> still wants a TL/SRE/Security read-through, because it puts a credential on Core's app registration,
> but it is a review rather than a decision.
>
> Both D1 and D2 are structured so you can read only the part relevant to you.

---

## 🔗 Two things that aren't decisions but belong in the same conversation

- **[[APIM IaC - authentication (phase 4)]]** — how customer sign-in actually works, what's proven and
  what isn't. Background reading for D1. Sam asked for auth to be discussed as a whole rather than
  piecemeal.
- **[[APIM IaC - known traps and gotchas]]** — the running register. Every decision here cites it.

---

## Status

| | Decision | Blocks | Decided? |
| --- | --- | --- | --- |
| **D1** Part 1 | How the sign-in secret reaches production | Ticket 5 | ✅ **2026-08-12**, revised **2026-08-13** — APIM creates it itself on Core's registration. Built and unit-tested; live proof outstanding |
| **D1** Part 2 | Who renews the secret before it expires | Ticket 5 | ✅ **2026-08-13** — nobody. It replaces itself every 30 days as part of the APIM deploy |
| **D2** Part 1 | Who owns the master documentation site | Ticket 6 | ⬜ |
| **D2** Part 2 | What permission the deployment gets | Ticket 6 | ⬜ |
| **D2** Part 3 | Where that grant lives | Ticket 6 | ⬜ |
| **D5** | Access to test on a real environment | **All** — nothing is proven without it | 🔄 in motion — Dreich run underway |
| **D7** | Accepting the unsupported migration | Ticket 6 | ⬜ |
| **D8** | West Europe for customer traffic | Ticket 1 | ⬜ |
| **D10** | Customer experience during a deploy | Ticket 3 | ⬜ |
| ~~D9~~ | ~~Scope~~ | — | ✅ **answered** — the full guide is in scope |

> [!info] Ticket numbers refer to the revised breakdown
> See [[APIM IaC - ticket breakdown]] *(revised 2026-08-18 — eight tickets plus one bug, replacing the
> earlier T1–T10 set)*. **D2 is the one to start early** — its code lands late, but the conversation
> needs SRE, Security, the PO and a TL, which has a longer lead time than the work does.
