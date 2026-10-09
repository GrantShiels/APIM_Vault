---
tags: [apim, iac, devo-28368, needs-decision, availability, customer-impact]
decision-id: D10
updated: 2026-08-11
status: needs-decision
sign-off: Product Owner · Tech Lead · SRE (Richard Donkin)
blocks: nothing today — decides what customers experience during every upgrade
---

> [!warning] Audited 2026-10-07
> Only the "Core unavailable" 503 is built (DEVO-28371). DEVO-28371's text claims both responses; it is wrong. Still open.

# D10 — What should a customer's integration see while we're mid-upgrade?

> [!question] What we need decided
> During a Dayshape upgrade there's a window where the API gateway and Dayshape itself are on
> different versions. **What should a customer's automated integration see during that window?**

---

## Why this matters

The customers of this API are not mainly people — they are **automated scripts** running on the
customer's side.

> [!important] People and scripts fail very differently
> A person who hits a broken page waits and tries again.
>
> A script that gets an unexpected response may **fail the job, retry in a loop, or drop data
> entirely** — and nobody notices until someone asks where the data went.

Dayshape has ways to handle this today: a maintenance mode and the ability to take instances out of
service. **The API gateway has neither.** It keeps serving requests throughout an upgrade, quite
happily, whatever state the system behind it is in.

So today the honest answer to *"what does a customer's script see during an upgrade?"* is:
**we don't know, and it's whatever happens to happen.**

### One half of this is already settled

The *ordering* question is answered: **the gateway deploys first.** Not by decision but by how it's
built — Dayshape holds a copy of a signing value the gateway creates, so the gateway must exist
first. That's now a property of the code.

**What's left is the half that affects customers**, and it's the more important one.

### This has already bitten us in a different form

Richard described a recent incident where cached responses left a customer believing an environment was down **for a day after the outage had actually ended.** Same category: the customer's view of our availability didn't match reality, and it cost more than the outage did.

As Sam put it: *"You guys need to figure out who your users are. We don't want outages."*

---

## The options

### Option A — the gateway returns a clear "temporarily unavailable" during a deploy ⭐

A standard, well-understood response that tells the caller to try again shortly.

**For:** the standard answer to this problem · well-behaved scripts handle it automatically · modest
work · honest **Against:** it is still an outage from the customer's perspective · badly-written scripts may treat it as a hard failure

### Option B — a proper maintenance mode for the API

Matching what Dayshape already has, with a considered response and, ideally, advance notice.

**For:** consistent with the rest of the product · best customer experience · could be announced ahead of time **Against:** the most work · needs design about who triggers it and when

### Option C — version negotiation, as Reporting does

Callers state which version they expect and the gateway routes accordingly.

**For:** removes the window entirely — no outage at all
**Against:** **Sam flagged this as probably out of scope** — it needs changes to the Dayshape API
itself, not just the gateway. Significantly the largest option.

### Option D — leave it as it is

Requests continue to be served throughout and whatever happens, happens.

**For:** no work
**Against:** unpredictable behaviour for automated callers, with no way to explain it afterwards ·
**this is what happens today by default**, so choosing it should be deliberate rather than passive

---

## Our view

**Option A for launch, with Option B as a follow-up.**

A clear "try again shortly" response is the standard, expected answer, and any competently written
integration will handle it. It converts "unpredictable" into "briefly and clearly unavailable",
which is a large improvement for modest effort.

Option B is better and worth doing, but it needs design input we don't have yet. Option C is the
ideal and, per Sam, probably not this piece of work.

> [!note] Whatever is chosen, one thing is worth doing anyway
> **Tell customers what to expect in the API documentation.** A documented "you may briefly receive
> a try-again response during maintenance; retry after a short delay" costs one paragraph and
> prevents a class of support ticket entirely.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

## What we need from you

- [ ] **Which option for 26.5?**
- [ ] How long is the window in practice? *(we can't measure this until **D5** is resolved)*
- [ ] Do we tell customers in advance, or only respond during the window?
- [ ] Should expected behaviour during maintenance be **documented for customers** regardless?

## Sign-off

**Product Owner · Tech Lead · SRE (Richard Donkin)**

The Product Owner matters because this defines what customers experience during routine upgrades —
a customer-experience commitment, not just a technical setting.

## Decision

*To be completed.*

---

## Related

- [[APIM IaC - outstanding questions]] — **Q12**, where this was raised and half answered
- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — the original discussion
- [[APIM IaC - known traps and gotchas]] — **G18** (the gateway keeps serving during a deploy), **G23** (why the gateway deploys first)
