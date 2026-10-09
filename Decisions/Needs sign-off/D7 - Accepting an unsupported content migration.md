---
tags: [apim, iac, devo-28368, needs-decision, risk, support]
decision-id: D7
updated: 2026-08-11
status: needs-decision
sign-off: Product Owner · Tech Lead
blocks: nothing — this is a conscious risk acceptance
---

# D7 — Accepting that Microsoft doesn't support how we copy the documentation

> [!question] What we need decided
> The way we copy the API documentation site onto each customer's environment is **not a supported operation** according to Microsoft's own documentation. It works, and we have proved it works. **Do we accept that risk, and who signs it off?**

---

## Why this matters

Copying the documentation site from our master onto a customer's uses a tool Microsoft publishes
for exactly this purpose. But their documentation — on a page updated **25 June 2026**, so this is
current, not a legacy caveat — says migration between the tier of API Management we use is **not
supported**.

That is precisely the copy we perform, on every customer deployment.

> [!info] To be clear about what this does and doesn't mean
> **It works.** We have run it against real Azure and verified the result: 70 content items copied,
> the customer's site correctly rebuilt, the customer's own sign-in settings preserved.
>
> The risk isn't that it's broken. The risk is that **if it ever stops working, we have no support
> route.** We couldn't raise a Microsoft ticket, because we'd be asking them to fix something they
> already say they don't support.

### What could actually go wrong

- A future Azure update changes the underlying behaviour and the copy silently produces a wrong result
- Microsoft retires the tool, or the version we depend on
- We hit a case the tool doesn't handle, and no one is obliged to help

**The likely warning is none.** This would surface as customer documentation sites looking wrong
after a deployment.

### There is a second, smaller version of the same problem

One piece of the documentation styling depends on a **machine-generated name** that Azure could
regenerate at any time. If it does, that styling breaks across every customer site simultaneously.
Same category of risk, smaller impact, and worth mentioning in the same conversation.

---

## Is there an alternative?

**We looked properly, and no.** This is written up in full separately, but in short:

We investigated whether the documentation could be held in our own source control and deployed like
normal code. **The words could be. The images could not** — Microsoft holds them in storage we don't
own and can only reach through a temporary one-hour link. Any home-grown version would reproduce
every page and lose every logo.

There is no maintained community alternative, and Microsoft's own guidance points at the same tool
we're using.

> **So this isn't a choice between a supported route and an unsupported one.** It's a choice between
> the unsupported route and not having customer-facing API documentation at all.

---

## The options

### Option A — accept the risk, document it, move on ⭐

**For:** honest about the actual situation · unblocks launch · the risk is low-likelihood
**Against:** we carry a dependency with no support route

### Option B — accept it, and reduce the impact

Same as A, plus practical mitigations:
- Keep a **copy of the master's content outside Azure**, so a bad copy can be restored
- Have the deployment **check the result** after copying and fail loudly if it looks wrong
- Pin the tool version *(already done)* so Microsoft can't change it under us without our say-so

**For:** the failure becomes recoverable rather than an incident · modest extra work
**Against:** more to build and maintain

### Option C — don't ship customer-facing documentation in 26.5

Deploy the API without the documentation site, and revisit later.

**For:** removes the risk entirely
**Against:** the documentation site is a large part of the customer value here · would need a
product conversation about what customers were promised

---

## Our view

**Option B.**

The risk is genuinely low — it works today, the tool is pinned, and this is Microsoft's own tooling
used in the manner they document elsewhere. But the *consequence* is customer-visible and the master
content has **no backup at all**, so a modest amount of safety-netting is well spent.

The single highest-value item is **a copy of the master content held outside Azure.** That protects
against this risk *and* the one in **D2 Part 1**, for the same effort.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

## What we need from you

- [ ] **Do we accept this risk for 26.5?**
- [ ] If yes, **Option A or B** — do we invest in the safety net?
- [ ] Who formally owns this accepted risk?
- [ ] Does it need recording anywhere beyond this document — a risk register, or the release notes?

## Sign-off

**Product Owner · Tech Lead**

This is a business risk acceptance, not a technical choice. The engineering position is that it
works and there is no alternative; whether that's acceptable is a product call.

## Decision

*To be completed.*

---

## Related

- [[APIM IaC - portal content, why there is no IaC-native route]] — the full investigation into alternatives
- [[D2 - The master documentation site]] — the same content, a different risk to it
- [[APIM IaC - known traps and gotchas]] — **G26** (this risk), **G24** (single point of failure)
