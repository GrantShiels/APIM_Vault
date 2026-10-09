---
tags: [apim, iac, devo-28368, design, availability, customer-impact]
updated: 2026-08-17
status: sketch — not a full design
relates-to: D10
---

> [!warning] Audited 2026-10-07
> Only the Core-unavailable 503 is built (DEVO-28371). Treat the second response as an open idea (D10).

# APIM IaC — maintenance responses (design sketch)

> [!note] Deliberately high level
> A shape to react to, not a plan to build from. The detail comes when the ticket is picked up.
> The *policy* question — what customers should experience — is [[D10 - What customers see during a deploy window|D10]].
> This is the *mechanism*.

---

## The problem

Dayshape shows a "down for maintenance" page while it deploys. **The API gateway has no
equivalent.** A customer's integration hitting an endpoint mid-deploy gets whatever happens to
happen.

That matters more than it does for the website, because the callers here are **automated scripts,
not people**. A person retries. A script may fail the job, retry in a loop, or drop data silently.

## What Dayshape actually does today

*Verified 2026-08-17 in `pipelines/Templates/Tools/ActivateDeactivateMaintenanceMode.ps1`.*

It is a **port swap, not a shutdown**:

1. A second IIS site, `DownForMaintenance`, already exists on the VM, parked on port **4433**.
2. On activate, the two sites swap: maintenance takes **443**, the real site moves to 4433.
3. The real Dayshape site is then **stopped**.
4. The maintenance site's `web.config` is rewritten to serve `maintenance.html` as the default
   document, with `Cache-Control: no-store` — the fix for the incident where a customer believed an
   environment was down for a day after it had recovered.

The window is the **whole release** — activated near the start of `Dayshape.Deploy.Release.yml`,
deactivated at the end.

## ⚠️ The finding that shapes everything

Because the maintenance site answers on **the same address and port**, APIM's call to `/proxy/...`
does not fail. It reaches IIS, which has no such path, and returns a **404 with an HTML error page**.

> [!warning] A 404 is the worst possible signal here
> **404 means "this endpoint does not exist"** — a permanent client error. A well-written
> integration will *not* retry it. It may reasonably conclude we have withdrawn the endpoint.
>
> It also means **catching a connection failure would never fire**. From the gateway's point of
> view, Dayshape answered perfectly well.

So the current default behaviour is not merely untidy — it is actively misleading.

## The shape of the answer

Two distinct, deliberate responses, so that a support ticket saying *"I got X"* immediately tells us
where to look.

| Situation | Gateway up? | Response | How it is detected |
| --- | --- | --- | --- |
| **Dayshape is deploying** | ✅ | `503` · `DAYSHAPE_MAINTENANCE` | Backend signal — see below |
| **The gateway is deploying** | ✅ | `503` · `GATEWAY_MAINTENANCE` | Flag set by our own deploy |
| **The gateway is genuinely gone** | ❌ | *nothing — connection failure* | Not solvable from inside APIM |

Both dressed responses should carry a `Retry-After` header and a machine-readable code in the body,
so a client can act on it without parsing prose.

> [!important] Be honest about the third row
> If APIM itself is gone, **nothing is running to send a message**. Answering that would mean
> putting something in front of the gateway — a much larger piece of work, and almost certainly not
> this one. It should be *stated*, not quietly implied away.

### Detecting "Dayshape is deploying"

**Preferred — the maintenance site returns `503` for any path.** A small config change on the
maintenance site; 503 is the correct code for maintenance in any case, and today's 404 is arguably a
defect in its own right. The gateway then translates that into a proper API-shaped response.

*Not our code* — the maintenance site lives on the VM, so this needs whoever owns it.

**Fallback — the gateway notices the backend returned HTML instead of JSON.** No cross-team
dependency, but it is inference rather than a signal, and could misfire on a genuine 404.

### Detecting "the gateway is deploying"

A flag the deploy sets on the way in and clears on the way out, checked by the global policy.

> [!danger] The failure mode to design against
> **If a deploy fails halfway, the flag stays set** — and we advertise an outage that has already
> ended. That is the same shape as the cached-response incident, and worse than the problem it
> solves.
>
> Needs a guaranteed clear-down, plus an alert if the flag has been set longer than a deploy takes.

## Worth doing regardless

**Document the expected behaviour for customers.** One paragraph — *"you may briefly receive a
try-again response during maintenance; retry after a short delay"* — removes a whole class of
support ticket, whichever mechanism we choose.

## Open questions

- [ ] Do we approach the owner of the maintenance site for the 503 change, or ship the HTML-detection fallback first?
- [ ] Should the gateway flag be per-environment, or global?
- [ ] Does the release pipeline's maintenance window need to cover the APIM deploy too, or are they independent?
- [ ] What `Retry-After` value is honest? Depends on measuring a real window — blocked on [[D5 - Getting access to test on a real environment|D5]].

## Related

- [[D10 - What customers see during a deploy window]] — the policy decision this implements
- [[APIM IaC - ticket breakdown]] — **T10**
- [[APIM IaC - known traps and gotchas]] — G18 (the gateway keeps serving during a deploy)
