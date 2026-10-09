---
tags: [apim, iac, decision, sre, existing-component, devo-28368]
date: 2026-07-13
source: "Slack thread with SRE team (Ivo)"
thread: "https://dayshape.slack.com/archives/C04PX2AL8SD/p1783680097364809"
---

# Decision — do NOT build on the existing APIM IaC

> **Key steer from SRE (Ivo).** Overrides the earlier
> "complete-and-wire-it" framing.

## The decision ✅

- **Ivo (SRE) has said we should NOT use the existing APIM IaC code** (`Products/Spoke/APIM`).
- We may **look at it for inspiration / reference**, but we should **build the component from the
  ground up** rather than trying to complete or work off it.
- Source: [Slack thread with the SRE team](https://dayshape.slack.com/archives/C04PX2AL8SD/p1783680097364809).
  *(Full thread not re-read here — Slack tooling unavailable this session; capture from Grant.)*

## Why this matters — it changes our earlier conclusion

Earlier notes ([[Existing APIM component - state assessment]],
APIM component - orchestrator and config integration) concluded **"complete and wire it, not
rebuild it"** — because the existing component already automated most of the manual guide.

**That framing is now superseded.** SRE's steer is a **greenfield rebuild** (informed by, but not
based on, the old code).

> [!note] Both things can be true
> The old component is still **valuable as a reference** — it shows *which* APIM resources are
> needed and how they map to the manual guide. We just don't inherit its code/config as a starting
> point. The mapping work we did on it is **not wasted** — it becomes the spec/checklist for the
> rebuild.

## Likely reasons for the steer *(to confirm with Ivo/SRE — not stated here)*

Consistent with what we already found in Why the APIM IaC was never finished:
- It was a **VDP proof-of-concept**, never productionised or owned by SRE.
- Hardcoded pentest config, placeholder secrets, no tests, bare manifest, entities/resources drift.
- Framework has **moved on** since it was written (Jan 2026) — building fresh on the current
  framework may be cleaner than retrofitting an old POC.
- SRE will **own/support** the deployment path, so they want it built their way from the start.

**TODO:** confirm Ivo's actual stated reasons (re-read thread when Slack access is back).

## Impact on the spike / epic

- **Effort goes UP vs the "complete it" scenario** — this is a build-from-scratch, not a finish job.
  Revisit [[Team skills vs APIM delivery - risk assessment]] with this in mind.
- **The reference value remains**: our [[Existing APIM component - state assessment]] resource
  inventory = the feature checklist for the new build.
- **DEVO-28369/28370/28371/28372** are all now "build new," not "adapt existing."
- Reinforces the need for **SRE pairing** — if SRE want it built their way, we need their patterns
  and standards up front.

## Actions

- [ ] Confirm Ivo's reasons + whether there's an SRE-preferred **reference component** to model on.
- [ ] Re-baseline effort/risk once the rebuild scope is clear.
- [ ] Keep the old component's resource list as the **spec**, not the codebase.

## Related

- [[Existing APIM component - state assessment]]
- Why the APIM IaC was never finished
- [[Team skills vs APIM delivery - risk assessment]]
