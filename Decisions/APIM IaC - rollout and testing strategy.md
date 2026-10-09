---
tags: [apim, iac, rollout, testing, environments, devo-28368]
updated: 2026-08-11
status: open — needs a plan, this is the raw material
related-ticket: DEVO-28368
---

# APIM IaC — rollout and testing strategy

> Written up from the 2026-08-07 call with Richard Donkin,
> Marcus Thomson, Sam Clifford and Andy L.
>
> **This is the biggest open theme in the whole spike.** It came up more than anything else on that
> call, and it doesn't have an answer yet. This document is the raw material for one, not the plan
> itself.

---

## The question Sam actually asked

He closed the call with this, and it's the thing to answer next:

> *"How do you get this across environments as quickly as possible, and what would an MVP look
> like?"*

Everything below is context for answering it.

---

## Part 1 — Why he asked: the AI Agent lesson

This is the part worth internalising. Sam described what went wrong with the AI Agent IaC:

> *"The IaC was written but never really tested in anger… developers building locally, testing
> locally, merging, under the illusion these things were working."*

**That is a precise description of our current setup.** Not a criticism of us — it's the default
shape of the work, and it's the shape that produced the problem before.

### What "under the illusion" means in practice

| What we have | What it proves |
| --- | --- |
| Component unit tests with **mocked Pulumi** | The code passes the right arguments. **Not** that Azure accepts them. |
| **Manual end-to-end checks at PR time** | It worked once, on one environment, when a human watched it. |
| A working **sandbox deploy** | It works on the sandbox. |

### What's missing

**Automated tests against actually-deployed infrastructure.** Nothing in our current setup would
catch a change that deploys fine on the sandbox and fails everywhere else.

There is some **incidental coverage** from internal environments — the IaC runs against them, so
breakage surfaces eventually. *(An environment name that sounded like **QAA4** was mentioned;
unconfirmed, worth checking.)* But incidental is not the same as deliberate.

---

## Part 2 — Why the sandbox proves less than it looks

Three problems that **only appear once you leave the sandbox**:

### 1. ⭐ The sandbox is Pulumi-built; other internal environments are legacy Terraform

This is the significant one. The environment we've done all our work on was itself built with
Pulumi. Most other internal environments were built with **Terraform**, which is being phased out.

So the sandbox isn't just *a* representative environment — it's arguably the **least**
representative one, because it's the only one shaped the way our tooling expects.

### 2. Capacity — the target hub may simply reject the deploy

Azure allocates capacity per region and subscription. A deploy that succeeds on the sandbox can
**fail outright** on another hub because there isn't room for it. We already hit a version of this:
Basic v2 APIM creation was blocked in UK South for our Sandbox subscription, which is why the spike
runs in West Europe *(see Q6)*.

This is not a code problem and no amount of local testing finds it.

### 3. Internal and bespoke environments have different shapes

Not every environment has the same components, the same naming, or the same networking. Code that
assumes the sandbox's shape will break on environments that differ — and we won't know which
assumptions we've baked in until we try.

---

## Part 3 — The proposed path

**Sandbox → internal → customer stage. As early as possible.**

The emphasis on *early* was Sam's. The value isn't in reaching customer stage eventually; it's in
reaching it before we've built ten phases on top of assumptions that don't hold there.

### The candidate first non-sandbox target: Dayshape internal

Sam's suggestion, and the reasoning is good:

- It's **real production**, so it's a genuine test rather than another sandbox
- It's **non-critical** — if it breaks, no customer notices
- *"We're the best people to experience that pain"* — if the deploy is rough, we're the ones who
  should find out

### Why stage matters specifically

**Stage is roughly equivalent to production.** It's the environment where the difference between
"works on sandbox" and "works for real" actually shows up.

It's also where customers' developers work. Richard was clear that production is locked down and
customer developers use stage: *"why would they need production data? That'd be really scary."* So
the developer portal audience is a **stage** audience.

---

## Part 4 — What this means for the work

### Reframing what "done" means for a phase

Right now a phase is done when it deploys to the sandbox and the tests pass. On the evidence above,
that's the definition that produced the AI Agent problem.

**A better bar:** a phase is done when it has deployed somewhere that isn't the sandbox.

That may be too strict to apply retroactively to Phases 1–6 — but it's worth deciding deliberately
rather than drifting.

### The MVP question

Sam asked what an MVP looks like. **Answered by the Product Owner (2026-08-11): there is no reduced
MVP — the full guide is in scope.** The complete feature is a hands-off orchestrator that can be run
whenever a customer opts into the new API endpoints. Customers will move across over the following
year rather than all at once.

What has changed since this was written: **all ten phases are now built**, so this is no longer
*"which do we build?"* but *"which do we ship, test and support?"* Some starting thoughts, **not
decisions**:

- The **documentation site carries most of the risk** — **D2** (which now absorbs the old D3 and D4)
  and **D7** exist because of it. If it isn't launch-critical, they leave the critical path at once.
  That single product answer changes the shape of the remaining work more than anything else.
  *(Updated 2026-08-13: this previously read "six of our ten outstanding decisions — D1, D2, D3, D4,
  D6, D7". D3/D4 were folded into D2 and D6 into D1; and **D1 was never about the documentation
  site** — it is the portal sign-in secret, and it is now resolved.)*
- The **guide being AI-generated (G16)** means its phase boundaries aren't authoritative. We can
  reorder or drop phases based on what an MVP genuinely needs, rather than following its structure.
- **Phase 2 is a confirmed no-op on Basic v2** — the tier doesn't create the default products the
  cleanup exists to remove, so the step logs "not found, skipping" and does nothing. It is harmless
  and can stay, but it should not be counted as MVP work. *(This previously read "unconfirmed" here
  while the working rules stated it as confirmed — resolved 2026-08-11 in favour of confirmed.)*

### The destroy-and-rebuild test is a rollout blocker

Migration is a **destroy and rebuild**, not an import *(Q1)*. But **Azure holds the APIM name for
a few hours after deletion** *(G15)*, and we have never tested reusing a name.

**That test has to happen on the sandbox before any environment migration is scheduled**, and it
takes most of a day because of the enforced gap.

---

## Part 5 — Ownership after handover

**SRE own it**, for now. Nobody proposed changing that for this work.

Sam on whether that's *right*: *"Big picture, should it be? Probably not… but that's a bigger
question."* Richard's practical argument for keeping it: **SRE covers out-of-hours**, and we don't.

**What it means for how we build:** built-for-SRE means preview-first, unambiguous failure messages,
and documentation that stands on its own. Not just working code. See Q10.

---

## Part 6 — Documentation, per Sam's steer

Relevant here because rollout is where documentation gets used in anger.

> AI Agent had **no feature delivery plan** — it has **ADRs plus component READMEs**, and that was
> enough.

**Where things go:**

| Kind of thing | Where |
| --- | --- |
| Architectural decisions | **ADRs** — the `docs/` folder at the repo root, alongside AI Agent's tactical ADRs and Marcus's recent ones |
| How a component works | **The component's README** — and it should **grow with the component**, not be written at the end |
| High-level overview | A README with **Mermaid diagrams** *(transcribed as "angel diagrams" — assuming Mermaid, unconfirmed)* |

⭐ **That's where Gabriela's requested diagrams belong** — the component README, not a separate
document.

---

## Part 7 — Open questions

> [!info] Updated 2026-08-11
> The blocker turned out to be narrower and more concrete than this section assumed. We **cannot
> deploy to any shared environment at all** — they live in a separate registry behind a credential
> store we're refused access to. So "is Dayshape internal the right first target?" is downstream of
> "can we deploy anywhere that isn't a personal sandbox?"
>
> That is now tracked for sign-off as **[[D5 - Getting access to test on a real environment|D5]]**,
> and there is a trap worth knowing about first — see **G29**.

**Now tracked as decisions:**

- [x] What does an **MVP** look like? → **answered: no reduced MVP, the full guide is in scope**
- [x] Can we deploy anywhere that isn't a sandbox? → **[[D5 - Getting access to test on a real environment|D5]]**
- [x] Is **Dayshape internal** the first non-sandbox target? → folded into **D5**
- [x] Do we change the **definition of done** to require a non-sandbox deploy? → asked in **D5**

**Still genuinely open, and ours to answer:**

- [ ] What **automated testing against deployed infrastructure** is realistic here? Who's done it
      before, and is there a pattern to copy?
- [ ] Which internal environments are **Terraform vs Pulumi**, and does the difference block us?
- [ ] Confirm the environment name — was it **QAA4**?
- [ ] Confirm **Mermaid** vs whatever "angel diagrams" was meant to be.

---

## Offers made on the call — worth using

- **Marcus:** *"if you need more time with any of us, just reach out."* Open offer, take it.
- **Sam:** his team is **moving to a DevEx focus** — he explicitly asked to be told about painful
  processes. The sandbox-vs-real-environment gap is exactly that.

---

## Related

- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — the source
- [[APIM IaC - outstanding questions]] — **Q11** (this theme), Q9 (how much of the guide), Q10 (ownership), Q12 (deploy sequencing)
- [[APIM IaC - known traps and gotchas]] — **G15** (name hold on delete), **G16** (guide is AI-generated), **G18** (version skew during deploys)
- [[APIM IaC - OpenAPI source of truth (phase 7)]] — §7.6 on the deployment window
- [[APIM IaC - ticket breakdown]] — where MVP scoping will land
- [[APIM IaC - working rules for implementation]] — rule 3 (write it like production code) is the same argument from a different angle
