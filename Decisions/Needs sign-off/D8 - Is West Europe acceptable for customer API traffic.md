---
tags: [apim, iac, devo-28368, needs-decision, data-residency, compliance]
decision-id: D8
updated: 2026-08-11
status: needs-decision
sign-off: Product Owner · Tech Lead · whoever owns customer data commitments
blocks: production deployment
---

> [!warning] Audited 2026-10-07
> Premise outdated: the code uses config `apim-location`, defaulting to the hub region, not fixed West Europe. The data-residency question is still open.

# D8 — Is it acceptable for customer API traffic to route through West Europe?

> [!question] What we need decided
> Our API gateway is currently built in **West Europe**, while everything around it sits in **UK
> South**. This wasn't a choice — Azure refused to create it in UK South. **Is a different country
> acceptable for production, and does it conflict with anything we've promised customers?**

---

## Why this matters

When a customer's system calls the Dayshape API, the request now passes through the gateway before
reaching Dayshape. If the gateway is in West Europe, **that request travels through a data centre in
a different country** to the one holding the customer's data.

> [!warning] This is not an engineering judgement
> Whether customer API traffic may transit another country is a question about **contracts,
> compliance and customer commitments**. Engineering can describe the situation accurately; we can't
> decide whether it's acceptable.
>
> This should not be settled by default simply because the sandbox happens to work.

### How we ended up here

On the first deployment, **Azure refused to create the gateway in UK South** for our sandbox
subscription — a capacity restriction, not a configuration mistake. It was pointed at West Europe to
get moving, and it has stayed there since.

**We don't yet know whether that restriction is permanent or specific to the sandbox subscription.**
That is worth establishing before treating this as a real constraint.

### The two angles

| Angle | Assessment |
| --- | --- |
| **Speed** | A small extra hop between the gateway and Dayshape. Almost certainly unnoticeable. Not the issue. |
| **Data residency** | Customer API traffic transiting another country. **This is the real question.** |

**Worth knowing:** Voodoo Park already run their gateway this way, so there is precedent — though
precedent isn't the same as approval, and their customer commitments may differ from ours.

---

## The options

### Option A — confirm the restriction, then move to UK South if possible ⭐

Establish whether UK South is genuinely unavailable or just unavailable to our sandbox. If it's the
latter, use UK South for real environments and sidestep the question entirely.

**For:** may remove the problem altogether for near-zero effort · keeps everything in one country
**Against:** we don't know the answer yet · if the restriction is real, we're back to the others

### Option B — accept West Europe, with sign-off

Confirm with whoever owns customer commitments that this is acceptable, record it, move on.

**For:** no technical work · already proven to work
**Against:** needs someone to genuinely check contracts rather than assume · may be fine for some
customers and not others

### Option C — region becomes a per-customer setting

Deploy the gateway wherever each customer's environment sits.

**For:** solves it properly and permanently · handles customers with strict requirements
**Against:** more work · doesn't help where Azure won't allow the region · needs testing per region

---

## Our view

**Start with Option A — it's a question, not a project.**

Before designing around the constraint, we should find out whether it's real. If UK South is
available to production subscriptions, this decision disappears.

If it *is* a real restriction, our instinct is Option C is the honest long-term answer for a product
sold to enterprises with data-residency requirements — but that should be weighed against how many
customers genuinely have such a requirement, which is a question we can't answer.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

## What we need from you

- [ ] **Is West Europe acceptable for production**, or sandbox only?
- [ ] **Does data residency rule it out** for any customer or region we sell into?
- [ ] Can someone confirm whether the **UK South restriction is permanent** or sandbox-specific?
- [ ] If it varies by customer, does region need to become a per-customer setting?

## Sign-off

**Product Owner · Tech Lead · whoever owns customer data commitments**

If there is a compliance or legal function involved in data residency, this needs their view rather
than ours.

## Decision

*To be completed.*

---

## Related

- [[APIM IaC - outstanding questions]] — **Q6**, where this was first raised
- [[D5 - Getting access to test on a real environment]] — testing elsewhere would confirm the restriction
- [[APIM IaC - rollout and testing strategy]] — capacity differs by region and subscription
