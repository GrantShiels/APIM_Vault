---
tags: [apim, iac, devo-28368, needs-decision, access, testing, rollout]
decision-id: D5
updated: 2026-08-11
status: needs-decision
sign-off: SRE (Richard Donkin · Marcus Thomson)
blocks: all testing outside personal sandboxes
---

# D5 — Getting access to test this on a real environment

> [!question] What we need decided
> Everything built so far has only ever run on **one engineer's personal sandbox**. We cannot
> currently deploy it anywhere else, because the list of real environments sits behind access we
> don't have. **Can we get that access, or should someone else run the test for us?**

---

## ✅ Answered — 2026-08-11, by Richard Donkin

> [!success] The route exists, and it doesn't need the access I assumed
> Richard's guidance: **deploy via the Azure DevOps pipeline, not the dev container.**
>
> - Each environment has its own deploy pipeline — Dreich is `Dayshape.Deploy.Internal-Dev-Dreich.yml`
>   (ADO definition **739**), which runs the orchestrator code
> - It takes an **`iacBranchOverride`** parameter, so a spike branch can be deployed without merging
> - For a single combined core-software + IaC deploy you'd need IaC version tagging — more involved,
>   and not needed for testing
>
> **The registry access I framed this decision around isn't required.** The pipeline runs as its own
> service connection, which already has it. Option C below — *"a pipeline that deploys to one internal
> environment on demand"*, which I listed as the most setup effort — **already exists for every
> environment.**

> [!warning] A working-rules miss worth recording
> Rule 1 is *reuse what already exists*. I proposed building something that was already there, because
> I reasoned from the blocker I'd hit (the dev container) rather than asking how deployments are
> actually done. **The dev container is an engineer's local tool; it was never the deployment route.**

**What remains of this decision:** nothing blocking. The residual questions — whether the definition
of done should require a non-sandbox deploy, and what automated testing against deployed
infrastructure looks like — are worth answering but no longer gate anything.

**Status: effectively resolved.** Retained for the reasoning and for the finding below, which came
directly out of acting on it.

---

## 🔴 What acting on this immediately uncovered

Verifying the Dreich route surfaced a production gap that the sandbox testing would never have shown.
Full detail in **[[D2 - The master documentation site|D2]]**; in short:

| Service connection | Access to the content master |
| --- | --- |
| `…Orchestrator.Internal` *(Dreich)* | ✅ — but **only by inheritance**, not by design |
| `…Orchestrator.Sandbox` | ✅ — same |
| `…Orchestrator.Lower` | ❌ **none** |
| `…Orchestrator.Upper` 🔴 **production** | ❌ **none** |

**The Dreich test will pass. Production would fail.** That is precisely the false-confidence pattern
this decision was written to avoid — and it was found by pursuing it.

---

## Original framing *(retained — the reasoning still holds)*



## Why this matters

This is the "does it actually work?" question, and right now we can't answer it.

> [!warning] What our testing currently proves — and doesn't
> | What we have | What it actually proves |
> | --- | --- |
> | Unit tests | The code passes the right instructions to Azure. **Not** that Azure accepts them. |
> | A working sandbox deployment | It works on **one** sandbox. |
> | Manual end-to-end checks | It worked once, when a person watched it. |
>
> **Nothing we have would catch something that works on a sandbox and fails everywhere else.**

Sam Clifford described exactly this pattern causing problems on a previous project:

> *"The IaC was written but never really tested in anger… developers building locally, testing
> locally, merging, under the illusion these things were working."*

That is a fair description of where we are. Not through carelessness — we physically cannot test
anywhere else.

### Why the sandbox proves less than it looks

- **The sandbox was built with the newer tooling**; most real internal environments were built with
  older tooling being phased out. So it's arguably the **least** representative environment we
  could have picked.
- **Capacity differs by region and subscription.** A deployment that succeeds on the sandbox can be
  refused outright elsewhere. We have already hit a version of this *(see **D8**)*.
- **The API endpoints we've built point at a web address that does not exist.** So while the
  plumbing deploys, **no request has ever reached a real Dayshape system.**

---

## What we found

There are **two separate lists** of environments. Deployments read from one or the other.

| List | Contains | Our access |
| --- | --- | --- |
| Development list | 73 entries — **all personal sandboxes** | ✅ Works |
| Real-environment list | Every shared and customer environment | ❌ **Refused** |

The second sits behind a credential store we're not permitted to read. Dayshape's own internal test
environment — a live, healthy, non-customer-facing environment, ideal for this — is in that second
list.

> [!danger] There is a trap here, and it is easy to walk into
> The development deployment command, when given an environment it doesn't recognise, **offers to
> add it**:
>
> `First time seeing client_id 'drch'. Add to GSOT? (y/n)`
>
> Answering **yes** silently creates a **wrong** record — pointing at the wrong subscription and a
> web address that doesn't exist. It would appear to succeed and produce something broken in the
> wrong place.
>
> **The safe answer is always no.** Recorded so nobody else finds this the hard way.

---

## The options

### Option A — grant us read access to the real-environment list

**For:** we can test properly and iterate quickly · no one else's time consumed
**Against:** it's a production-adjacent credential store, so it needs proper justification · may be
more access than is appropriate for this purpose

### Option B — someone with access runs the deployment while we watch ⭐

A short pairing session with SRE.

**For:** no permanent access change · knowledge transfer both ways · someone experienced is present
the first time this touches a real environment
**Against:** needs their time · slow to iterate if we need several attempts

### Option C — a pipeline that deploys to one internal environment on demand

**For:** repeatable · no interactive access needed · closest to how it'll really run
**Against:** most setup effort · still needs the environment registered by someone with access

### Option D — accept sandbox-only testing for now

**For:** no dependency on anyone
**Against:** **this is the option that produced the problem Sam described.** We would be shipping
something whose only evidence is that it worked on the least representative environment we own.

---

## Our view

**Option B to start, then Option C.**

A pairing session gets us a real answer quickly and puts an experienced pair of eyes on the first
real deployment. If that goes well, a pipeline makes it repeatable.

Option A may not be necessary at all if B and C work — which is worth saying, because asking for
production-adjacent access should have a real justification behind it and there may not be one.

**Option D should be explicitly rejected rather than allowed to happen by default.** Right now it is
what happens if nobody decides anything.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

## What we need from you

- [ ] **Which option?**
- [ ] If B — who, and roughly when?
- [ ] Is Dayshape's internal test environment the right first target, or is there a better one?
- [ ] Should "deployed somewhere that isn't a sandbox" become part of our definition of done?

## Sign-off

**SRE — Richard Donkin, Marcus Thomson**

*Grant has already opened this conversation with both.*

## Decision

*To be completed.*

---

## Related

- [[APIM IaC - rollout and testing strategy]] — the fuller picture, including the previous project's lessons
- [[APIM IaC - outstanding questions]] — **Q11**
- [[APIM IaC - known traps and gotchas]] — **G15** (name held after deletion, blocks rebuild testing)
