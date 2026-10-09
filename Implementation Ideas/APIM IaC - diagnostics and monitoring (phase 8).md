---
tags:
  - APIM
  - IaC
  - decision
  - as-built
  - phase8
related: DEVO-28368
status: superseded by the code — built and merged (commit 2a78113)
date: 2026-08-07
updated: 2026-08-07
---

> [!warning] Audited 2026-10-07
> Tier references to Basic v2 are changing to Standard v2 (INFRA-777). Monitoring is DEVO-30005, which should be one of the last tickets done.

# APIM IaC — diagnostics and monitoring (Phase 8)

> [!warning] Superseded by the code
> Phase 8 is **built and merged** (commit `2a78113`). This document previously said "availability result pending"; that result was never recorded here, so treat the deployed alerting as the source of truth.
>
> **Where this document and the code disagree, the code is right.** Kept for the reasoning behind
> the decisions, not as a description of how things behave today.

> [!abstract] The whole thing in four sentences
> Once APIM is serving traffic, we need to be able to answer *"is it up, and what happened to that
> customer's request?"* Phase 8 wires four things that together answer both questions: a **link** to
> Application Insights, a **recipe** for what gets recorded about Entities API traffic, a **stream**
> of raw gateway logs into Log Analytics, and an **outside-in ping** that tells us the gateway is
> alive. All four are built in IaC and deployed cleanly — the manual guide's version of this is
> roughly forty copy-pastes per API, per customer.

**Status: built, deployed, and verified against the guide.** One thing still to confirm by eye — see
[Still to confirm](#still-to-confirm).

---

## 1. What Phase 8 actually does, in plain terms

Four separate Azure resources, doing two different jobs.

| # | What it is | The question it answers |
| --- | --- | --- |
| 1 | **Logger** | *Where do APIM's traces go?* Connects this APIM service to the hub's Application Insights. |
| 2 | **API diagnostic** | *What do we record about a request?* Which headers, how much detail, how often. |
| 3 | **Diagnostic setting** | *What happened across the whole gateway?* Streams raw logs + metrics to Log Analytics. |
| 4 | **Availability web test** | *Is it even up?* Pings the gateway every 5 minutes from four locations outside Azure. |

The first two are about **debugging a specific request**. The last two are about **knowing the
service is healthy**. Both matter, and they use different Azure plumbing, which is why there are four
resources and not one.

---

## 2. Decision — we moved the diagnostic setting into Phase 8

The guide files the Log Analytics diagnostic setting under a later section called *"Setup alerts in
production"*, not under Phase 8.

**We moved it.** It is logging plumbing, not alerting. It's filed with the alerts only because the
alert rules query the data it produces — but building it here means the alerting work later has
nothing half-finished waiting for it, and Phase 8 delivers a complete, coherent unit: *everything
that produces telemetry*.

This is consistent with the standing position that **the guide's phase boundaries are not
authoritative** (see [[APIM IaC - known traps and gotchas]] G16 — the guide is AI-generated).

## 3. Decision — the availability monitor stays in Phase 8, not with the alerts

This one was genuinely open, and the first instinct was wrong.

Phase 8A's checklist includes *"Add availability monitor: Yes"*, so it belongs to Phase 8 on the
guide's own terms. But an availability monitor implies alerting, and alerting was going to be its own
phase — so the question was whether the two were too tightly coupled to separate.

**They aren't.** Two findings settled it:

1. **No dependency on alerting work.** Our repo uses pre-existing platform-wide action groups via
   `get_all_action_groups_for_environment()`. In sandbox that returns `None` at every severity — so
   the web test's alert has nothing to wait for. Confirmed on deploy:
   `Creating log alert: ..., actions: None`. In higher environments it picks up the platform action
   groups automatically, with no Phase 8 change.
2. **The alerts section is much bigger than it looks.** It isn't two rules, it's **seven** — 5xx
   rate, capacity, GET p95, POST p95, unavailability p95, failure rate, and any 429 — plus a
   post-deploy verification step. Merging that into Phase 8 would have blown well past the
   30-minute review budget.

**Decision:** availability monitor in Phase 8; the seven alert rules get their own phase.

## 4. Decision — ping APIM's built-in status endpoint

The monitor needs a URL that proves the gateway is alive without needing a key or a token.

APIM publishes exactly that: `/status-0123456789abcdef`. It's unauthenticated by design, it doesn't
touch Core, and it doesn't consume a subscription quota. Pinging a real API operation instead would
have meant embedding a credential in a monitoring resource and generating fake traffic through the
Entities API.

**Trade-off, stated honestly:** this proves *APIM* is up. It does **not** prove Core is reachable or
that Entities actually returns data. That's a deeper health check and a separate decision.

> [!note] The URL is composed, not read from the resource
> It's built from the service name — `https://{name}.azure-api.net/status-0123456789abcdef` — rather
> than read off `apim_service.gateway_url`, because that's a Pulumi `Output` and can't be dropped
> into an f-string. This assumes the default Azure hostname, which holds for BasicV2.
> **The deploy confirmed the composed URL was correct.**

## 5. Decision — settings are fixed constants, not configuration

Sampling rate, verbosity and the four header lists are hard-coded constants in
`apim_spike_helpers.py`, not per-customer config.

They're operational tuning that should be identical everywhere. Making them configurable invites
customer-specific drift in exactly the data we'd rely on to debug a customer-specific problem. If we
later need to raise sampling to chase an issue, that's a code change and a deploy — which is the
right level of ceremony for something that changes what we record about live traffic.

## 6. Decision — pass the connection string straight through

The old APIM code on `develop` staged the Application Insights connection string in a secure APIM
**named value** first, then pointed the Logger at that. We don't.

The value is already held in Pulumi state by the lookup that fetches it, so wrapping it in a named
value adds a resource to manage without putting the secret anywhere it isn't already.

> [!success] Azure agreed — and did it for us
> The deploy came back with the Logger's credential set to
> `{{Logger-Credentials--6a75e533ce6cc21d688d315f}}`. **APIM automatically created its own named
> value** for the connection string. So the value is properly stashed, and we didn't have to write
> or maintain the resource that does it. This validates the decision after the fact.

We also use a **connection string rather than an instrumentation key** — the key-based form is on
Microsoft's retirement path.

## 7. What we deliberately did *not* copy from the old code

The old VP APIM implementation still exists on `origin/develop` at
`Products/Spoke/APIM/infra/apim_resource_helpers.py` (lines 717 and 769). It was genuinely useful for
**shape** — it confirmed `apim.ApiDiagnostic` is the right class, that `diagnostic_id` must literally
be `"applicationinsights"`, and that `"W3C"` works as a plain string. That closed three unknowns
without a deploy.

**Its values are another matter.** Three things in it we did not carry forward:

| Old code did | Guide says | We did |
| --- | --- | --- |
| Logged **8192 bytes** of request *and* response bodies | `0` | `0` |
| Logged **client IP** | `False` | `False` |
| Invented a SKU-derived sampling ladder | flat 20% | flat 20% |

The first two are the important ones. Request and response bodies through the Entities API contain
customer data; 8 KB of it per sampled request, sitting in Application Insights, is a privacy exposure
nobody asked for. Same for client IPs.

> [!warning] This isn't hypothetical drift
> That configuration was **written and shipped once already**. Encoding `bytes: 0` and
> `logClientIp: false` as named constants with a comment explaining *why* is what stops it happening
> a second time. This is the strongest single argument for Phase 8 being in IaC rather than a
> checklist.

---

## 8. The PO talking point — what this replaces

Straight from the guide, about the header configuration:

> *"All these headers below need to be pasted in individually (yes very painful) due to the APIM UI
> taking it all as a literal string if done together."*

That is **39 headers across four lists**, entered one at a time, through a portal form, **per API,
per customer**. Miss one and you don't find out until you're trying to debug a live incident and the
header you need isn't there.

In IaC it's four Python lists, written once, applied identically everywhere, and covered by a unit
test. This is a good, concrete example of the manual-to-IaC value story — it isn't about saving
typing, it's that **the fortieth customer gets exactly the same configuration as the first**.

---

## 9. What got deployed — verified against the guide

Deployed to `grant-sbx` on 2026-08-07. **7 resources created, 0 errors, 16 seconds.**

Every Phase 8B setting checked against the guide from the actual deploy output, not assumed:

| Guide requires | Deployed | |
| --- | --- | --- |
| Sampling 20% | `percentage: 20, samplingType: fixed` | ✅ |
| Verbosity Error | `verbosity: "error"` | ✅ |
| Always log errors True | `alwaysLog: "allErrors"` | ✅ |
| Log client IP **False** | `logClientIp: false` | ✅ |
| Support custom metrics False | `metrics: false` | ✅ |
| Correlation protocol W3C | `httpCorrelationProtocol: "W3C"` | ✅ |
| Payload bytes **0** (all four sections) | `bytes: 0` ×4 | ✅ |
| Frontend request headers (13) | 13, in order | ✅ |
| Frontend response headers (11) | 11, in order | ✅ |
| Backend request headers (8) | 8, in order | ✅ |
| Backend response headers (7) | 7, in order | ✅ |

Plus: Logger `loggerType: applicationInsights` bound to the hub instance; diagnostic setting
streaming `GatewayLogs` + `AllMetrics`; web test `provisioningState: Succeeded` against the composed
status URL, every 5 minutes from 4 European locations, expecting HTTP 200 with SSL checking on.

**Guide's Enable Advanced Options checkbox** was correctly ignored — it's a UI-only toggle that
reveals the per-pipeline settings in the portal form. It has no API equivalent, and we set those
settings directly.

### Test coverage

**29 unit tests, all passing**, of which 5 are new for Phase 8. The privacy-critical one is
`test_entities_api_diagnostic_never_logs_payload_bodies_or_client_ips` — named so that anyone who
later loosens those settings has to consciously delete a test that says why they exist.

The web test assertion was **mutation-tested**: the expected URL was deliberately corrupted and the
test failed as expected. That matters because of the vacuous-capture failure mode in
[[APIM IaC - known traps and gotchas]] G14/G19, where an assertion can pass while proving nothing.

---

## 10. Still to confirm

- [ ] **Is the availability test actually green?** `provisioningState: Succeeded` only means the test
      was *created*. It needs ~10 minutes of runs before App Insights → Availability shows a result.
      If it's red, `/status-0123456789abcdef` isn't returning 200 through the gateway and we need a
      different URL.

## 11. Watch items — not defects

- **Log Analytics is using the legacy table layout.** The setting deployed with
  `logAnalyticsDestinationType: "AzureDiagnostics"` — the older shared-table format rather than
  resource-specific tables. That's what the shared `logging_interface()` helper does for *every*
  component, so we're consistent with the house pattern rather than special. Only worth revisiting
  if the seven alert queries later turn out to want the newer format.
- **The web test and its alert live in the hub resource group** (`sre-ds-uks-lwr-sh-rg`), because
  that's where App Insights is. Names are client-prefixed so there's no collision, but at 100+
  customers every availability test and alert accumulates in one shared hub RG. Worth a sanity check
  with SRE before wider rollout. See G21.
- **The alert notifies nobody in sandbox** (`actions: None`). Expected — sandbox has no action
  groups. It will pick up platform action groups automatically in higher environments, but that
  behaviour is **unproven until we deploy somewhere that has them**.
- **The diagnostic setting names itself `{service}-diag`**, not the guide's *"Default Metrics"*. The
  house helper wins; the name is cosmetic.

## 12. Traps found while building this

Both are written up in full in [[APIM IaC - known traps and gotchas]]:

- **G21** — `create_availability_web_test()` has no `return` statement. Assigning its result binds
  `None`, and the `# noqa: F841` on its internals hides the fact. Also: one web test per component
  maximum, because it overrides the Pulumi provider.
- **G22** — because of G21 there's no resource handle to anchor a test assertion on, so the test
  anchors on the *next* resource created. That makes the ordering inside `__configure_diagnostics()`
  load-bearing, which is now flagged with a comment in the code.

---

## 13. Update (2026-08-19) — the seven production alert rules are now built

Phase 8 originally delivered *everything that produces telemetry*. The guide's final section,
**Setup alerts in production**, defines seven rules that consume it. Those are now built too, in
`Products/Spoke/ApimSpike/infra/apim_spike_alerts.py`, following the CognitiveSearch pattern
(`Spoke/CognitiveSearch/infra/search_component_log_alerts.py`): a dedicated alerts module with the
KQL held in separate `.kql` files under `infra/kql/`.

| Rule | Type | Severity | Window / Freq | What it catches |
| --- | --- | --- | --- | --- |
| 5xx rate > 1% | Log | 1 Error | PT5M / PT5M | The API is erroring for customers |
| Capacity > 85% | Metric | 2 Warning | PT10M / PT1M | The gateway is running out of headroom |
| Unavailabilities p95 > 30s | Log | 2 Warning | PT5M / PT1M | The slowest endpoint has degraded |
| Endpoint failure rate > 1% | Log | 2 Warning | PT5M / PT1M | Any failure, not just 5xx |
| GET p95 > 5s | Log | 3 Info | PT5M / PT1M | Reads getting slow |
| POST p95 > 15s | Log | 3 Info | PT5M / PT1M | Writes getting slow |
| Any 429 | Log | 3 Info | PT5M / PT1M | A customer is being rate limited |

### Three deliberate deviations from the guide

1. **Alerts live in `dayshape-alerting`** in the platform management subscription, not the customer's
   resource group. That is what `LogSearchAlert`/`MetricAlert` default to and what every other
   component does.
2. **Shared platform action groups**, not a per-customer group emailing `forth-team@dayshape.com`.
   Matches every other component, and goes quiet in sandbox/dev for free. Trade-off accepted:
   platform on-call is less familiar with the API than the Forth team would be.
3. **Severity-mapped routing** — Error to `p1`, Warning to `p2`, Informational to `p2_4` — rather
   than routing everything to `p1` as the two existing in-repo examples do. Keeps latency blips and
   429s off the paging channel.

### How the thresholds work

All six log rules are configured as `threshold=0`, `GreaterThan` on *count of rows*. The real
threshold lives **inside the KQL**, which filters down to only the breaching rows. So the Azure rule
reads "alert if the query returned anything at all". This is the shape the guide uses and it is
worth knowing, because reading the Azure portal alone makes every rule look like it has no threshold.

> [!warning] Superseded by §14
> The `spk-apimgs-infra-apim_alert_*` keys described here were our own invention and no longer
> exist. See **§14** for the convention that replaced them.

### ⚠️ The biggest risk with this work

**A rule whose query matches nothing looks exactly like a healthy service.** Nothing in Azure tells
you a KQL filter is wrong — the rule just never fires, forever. Two facts were verified rather than
assumed before building:

- `ENTITIES_API_PATH = "entities"`, so the guide's `CleanUrl contains "/entities/"` filter genuinely
  matches our gateway traffic.
- `/unavailabilities` endpoints genuinely exist in `openapi.v1.json` (190 references).

That is still not enough. **Each query must be run against real Dreich gateway logs and shown to
return rows when it should.** That validation is outstanding and is not optional.

### Known-by-design quiet spots

- **Low-traffic customers are silent.** The 5xx and failure-rate queries require `Total >= 20`
  requests in the window. A newly onboarded customer will not trigger them. Intended, but should be
  a documented expectation rather than a surprise.
- **Sandbox and dev page nobody.** Every action group tier resolves to `None` there, so the rules
  are created but wired to nothing. Same behaviour as the availability alert above.
- **Recurring cost.** Six log alerts evaluating every minute, per customer, is an ongoing Azure
  Monitor cost that scales with API adoption.

### Trap found while building this

**G46** — `assertPulumiOutputCustom` is just `return output.apply(assertion_function)`. It does not
await. Assertions inside it *do* run and *do* fail the test, but they surface asynchronously during
`wait_for_rpcs()`, which means (a) you cannot wrap one in `self.assertRaises`, and (b) if you assert
on a mock capture, the capture is only guaranteed populated if you anchor on an Output belonging to
that same resource or a later one. Both bit during this work. Where a wrapper class stores its own
config on `self` — as `MetricAlert` does — asserting on that directly is simpler and has no timing
dependency at all.

---

## 14. Update (2026-08-19) — alert config brought onto the estate convention

§13 shipped with two things we had invented rather than inherited. Both are now corrected against
`Modules/ContainerApp/infra/function_app_alert.py`, which is the reference implementation.

### What was wrong

1. **Our own key naming.** `apim_alert_capacity_percent`, `apim_alert_window_minutes` and four
   others. Every other component uses `alert-<rule>-<field>`. Anyone grepping a customer config for
   `alert-` would have found every component's settings except ours.
2. **No override hooks at all.** Every other component lets a customer change `action_group_id`,
   and most expose `enabled`, `threshold`, `window_size`, `evaluation_frequency` and `severity`. We
   exposed none, so calming one noisy rule for one customer meant a code change for all of them.

### The convention now followed

Keys are `spk-apimgs-infra-alert-<rule>-<field>`. Hyphens separate the three parts; underscores sit
inside a part.

| | |
| --- | --- |
| **Rules** | `5xx_rate`, `get_latency`, `post_latency`, `unavailabilities_latency`, `failure_rate`, `rate_limited`, `capacity` |
| **Fields** | `enabled`, `severity`, `threshold`, `window_size`, `evaluation_frequency`, `action_group_id`, `query` |

`severity` accepts either a name (`WARNING`) or a Sev number (`2`). A blank or `[[empty_string]]`
value means *unset*, not *empty*.

> [!important] Defaults live in code, not in `infra_defaults.yml`
> This looks wrong and is not. **No yml anywhere under `Products` declares an `infra-alert-` key.**
> Reporting is the exact precedent — it has both an `infra_defaults.yml` and its own alert module,
> and declares none of its alert keys in the yml. Ours documents the convention in comments and
> declares nothing.

### The trade-off we accepted

The numeric thresholds — the 1% failure rate, the 5/15/30 second latency targets, the 20-request
floor — are now **literals inside the KQL**, as they are for every other component. Changing one
means overriding the whole query via `alert-<rule>-query`, not setting a number.

**That is a real step down in usability for the knob most likely to be turned.** It was chosen
deliberately: it is how ContainerApp and CognitiveSearch already work, and consistency across the
estate was judged worth more than a nicer surface on one component. Worth recording that it was a
choice rather than an oversight. The `{apim_resource_id}` and `{api_path_filter}` placeholders are
substituted into overrides too, so a query set in shared config stays scoped to one customer.

### A latent bug this removed

See **G47**. Every query carried `| where TimeGenerated >= ago({window_minutes}m)` while the rule's
`window_size` was separately hardcoded to `PT5M` — two settings for one concept, only one of them
reachable from config. Set the window to 15 and the query kept looking back 5. Deleting the time
filter fixed it by construction and matched the estate at the same time: **no sibling component's
KQL filters on time**. `window_size` alone governs the range, which is how CognitiveSearch runs the
same query shape at `PT5M` and at `PT15M`.

### Verification

117/117 ApimSpike tests; 409 passed / 6 skipped across ApimSpike, Core, AppCfg, CognitiveSearch,
Alerts and ContainerApp; `ruff check` clean. With no config set the behaviour is exactly what §13
describes — same seven rules, same thresholds, same severities, same routing.

---


## Related


- [[APIM IaC - known traps and gotchas]] — **G21**, **G22** (found here), **G14**/**G19** (the test
  traps this phase had to work around), **G16** (why the guide's phase boundaries aren't binding)
- [[APIM IaC - OpenAPI source of truth (phase 7)]] — the Entities API this diagnostic attaches to
- [[APIM IaC - working rules for implementation]] — the automate-everything rule this phase follows
- `Products/Spoke/ApimSpike/infra/apim_spike_component.py` — `__configure_diagnostics()`,
  `__create_availability_web_test()`
- `Products/Spoke/ApimSpike/infra/apim_spike_helpers.py` — the Phase 8 constants and header lists
- `Products/Modules/AppInsights/infra/app_insights_interface.py` — `create_availability_web_test()`
- `Products/Modules/LogAnalyticsWorkspace/logging_interface.py` — `logging_interface()`
- `origin/develop:Products/Spoke/APIM/infra/apim_resource_helpers.py` lines 717, 769 — the old
  implementation: useful for shape, **not for values**

