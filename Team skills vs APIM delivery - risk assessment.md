---
tags: [apim, risk, delivery, ownership, ecosystem, devo-28368]
updated: 2026-08-19
status: living-doc
related-ticket: DEVO-28368
---

# Team skills vs APIM delivery — risk assessment

> [!abstract] What this is
> Ecosystem has been given a capability that sits **outside the team's established skill set**, and
> will own it long term. This sets out honestly what that means, what the risks are, and what would
> reduce them.
>
> **This is not an argument that the team can't do it.** The spike has demonstrated the opposite —
> six phases are built and the hard parts are proven against real Azure. It's an argument that the
> risks are **specific and named**, and most have practical mitigations that cost very little.

Written from evidence gathered during the spike, not from opinion. Everything here traces back to
something we hit, found, or were told.

---

## The situation, stated plainly

| | |
| --- | --- |
| **What we own** | API Management as a **product** — how it's built, configured, made repeatable, documented and evolved *(see [[Why Ecosystem owns APIM]])* |
| **What SRE owns** | The platform underneath — subscriptions, infrastructure, deeper operational guardrails |
| **Team's background** | Application development, inside the Dayshape codebase. Integrations are our home ground. |
| **Team's Azure/infrastructure background** | **Limited.** Nobody on the team had worked with API Management before this spike. |
| **Why us anyway** | The API is an Ecosystem product. Doing it any other way leaves SRE as a manual bottleneck on every customer deployment — which is precisely the problem being solved. |

**The reasoning for Ecosystem owning this is sound.** The risks below are about the *transition*,
not the decision.

---

## The risks

### R1 — Specialist knowledge we don't have, on the critical path 🔴

Two areas have surfaced where the right answer needs expertise nobody on the team has:

- **Microsoft Entra / identity.** The sign-in design is constrained by how identity tokens work
  (**G9**), and the current approach uses a **deprecated** method that will need replacing (**G5**).
  Deciding whether the better design is even possible needs someone who genuinely knows Entra.
- **Azure permissions and role design.** Getting this wrong fails in ways that are partial and
  late — a customer gets a documentation site with every page correct and no images (**G27**).

We can identify these questions accurately. **We cannot confidently answer them.**

> **Mitigation:** name a specific person outside the team for each area, rather than "ask SRE".
> Both are captured as decisions — **D1** and **D2** — so the ask is concrete rather than open-ended.

### R2 — We can't test anywhere real 🔴

Everything built has only ever run on **one engineer's personal sandbox**. Shared environments sit
behind access we're refused.

This is the risk Sam Clifford described from a previous project — *"the IaC was written but never
really tested in anger… under the illusion these things were working."* It's a fair description of
where we are, and **not through carelessness — we physically cannot test elsewhere.**

The sandbox is also arguably the **least** representative environment we own: it was built with
newer tooling than most real internal environments.

> **Mitigation:** **D5.** This is the single highest-value thing to unblock, and it's already in
> motion with Richard and Marcus.

### R3 — The documentation we were given isn't reliable 🟠

The 51-page setup guide is **AI-generated (G16)**, so its structure isn't authoritative and its
completeness can't be assumed.

The evidence is the trap register: **47 recorded traps**, the large majority of which the guide
doesn't mention. Several would have caused customer-visible failures.

> This cuts both ways. It's a real risk — but finding those 47 traps is exactly the value this spike
> produced, and a team with more Azure experience might have moved faster *past* some of them without
> writing them down.

> **Mitigation:** already working. Keep the trap register as a living document, and decide
> deliberately whether the guide is retired or kept as a fallback. *(Scope itself is settled — the
> full guide is in scope.)*

### R4 — Effort is higher than a comparable "finish the existing work" job 🟠

We assessed the existing partial APIM infrastructure code and concluded it shouldn't be built on
*(see [[Decision - do not build on existing APIM IaC]])*. That was the right call on the evidence,
but it does mean **starting from a blank page**, and the cost is higher than picking up someone
else's near-finished work would have been.

> **Mitigation:** none needed — this is a known, accepted, documented consequence of a deliberate
> decision. Recorded here so nobody is surprised by the effort later.

### R5 — Most of what's left needs someone outside the team 🔴

**Seven** outstanding decisions require product, tech-lead, security or SRE sign-off. **None** can be
made by Ecosystem.

> [!important] This is the risk most likely to be underestimated
> "Six phases built, the hard parts proven" sounds close to done. It isn't, and the remaining work
> is largely **not engineering** — it's decisions and permission grants, each depending on someone
> else's availability.
>
> Engineering capacity is not the constraint. **Other people's time is.**

> **Mitigation:** [[_Decisions needing sign-off - start here]] — each decision written as a
> standalone document for the person who needs to make it, ordered so the highest-leverage one comes
> first.

### R6 — Long-term ownership, including out-of-hours 🟠

Ecosystem doesn't currently carry out-of-hours responsibility. Several failure modes here surface
outside working hours or on long timers — the sign-in secret expiring after 12 months (**G7**) being
the clearest.

> **Mitigation:** settle ownership explicitly rather than by default — **D2 Part 1**. The split with
> SRE is already being formalised under **DEVO-28373**. *(D1 Part 2 was also listed here; it no
> longer needs an owner — the sign-in secret renews itself as of 2026-08-13.)*

### R7 — Concentrated in one person 🟠

The spike has largely been carried by one engineer. The accumulated knowledge — the traps, the
reasoning, the dead ends — sits mostly with them.

> **Mitigation:** this is the risk the documentation directly addresses, and it's in good shape. A
> second person walking through the trap register and the decision set would confirm whether it's
> genuinely transferable, and is cheap to arrange.

---

## Summary

| | Risk | Severity | Mitigated by |
| --- | --- | --- | --- |
| R1 | Specialist Entra / permissions knowledge missing | 🔴 | Named experts · D1 · D2 |
| R2 | Can't test on a real environment | 🔴 | **D5 — highest priority** |
| R3 | Source documentation unreliable | 🟠 | Trap register |
| R4 | Higher effort than a "finish it" job | 🟠 | Accepted, documented |
| R5 | Most remaining work needs others | 🔴 | The decision pack |
| R6 | Long-term and out-of-hours ownership | 🟠 | D4 · D6 · DEVO-28373 |
| R7 | Knowledge concentrated in one person | 🟠 | Documentation · a second reader |

---

## The plain version for a Product Owner

We've been given a piece of Azure infrastructure work that sits outside what this team normally
does, and we'll own it long term. That's a reasonable decision — it's our API, and the alternative
makes another team a bottleneck on every customer.

The spike went well. What was meant to be a two-week investigation produced working automation for
all ten stages of a manual process, and found **47 problems the original documentation didn't
mention** — several of which would have reached customers.

**The honest risks are three.** We can't test it anywhere except one engineer's sandbox, so we don't
truly know it works. There are two specialist areas where we can ask the right question but can't
answer it. And most of what's left isn't engineering at all — it's **seven decisions that need people
outside this team**, which makes other people's availability the constraint rather than ours.

None of these are reasons to stop. They're reasons the remaining timeline depends on things this
team doesn't control, and that's worth knowing before committing to a date.

---

## Related

- [[APIM IaC - what made this hard (retrospective)]] — the backward-looking companion: what actually made this hard
- [[Why Ecosystem owns APIM]] — why this sits with us
- [[_Decisions needing sign-off - start here]] — the seven still outstanding, and who each needs
- [[APIM IaC - known traps and gotchas]] — the 47 traps
- [[APIM IaC - rollout and testing strategy]] — R2 in full, including the previous project's lessons
- [[Decision - do not build on existing APIM IaC]] — the decision behind R4
- [[Existing APIM component - state assessment]] — the evidence for it
- [[APIM and SRE - understanding]] — the ownership split, formalised under DEVO-28373
