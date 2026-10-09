---
tags: [apim, iac, devo-28368, planning, tickets]
updated: 2026-08-24
status: current
---

# APIM IaC — ticket breakdown

Eight tickets to take the APIM work from a proven spike to production. Each is framed as an outcome
— *"you can now do X"* — rather than as a chapter of the setup guide. That is deliberate: it makes
each one independently demonstrable and independently valuable, which is what the
[[https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/5484314734/GSS+IAC+Componentisation+FDP|GSS IAC FDP]]
does with its milestones.

> [!warning] This is not "one component" work
> The APIM commits touch **Auth, Core, CoreLegacyVM, AppCfg, AppInsights, Globals, DynamicProviders
> and the Core release pipeline**. Any ticket that scopes review and testing to the component folder
> alone will under-scope it.

---

## The rule that shapes everything

> [!important] Nothing from the spike is merged or moved
> Every ticket is implemented **fresh**, on its own branch off `iac-main`, written to production
> standard from the start. The spike branch is never merged and no code is lifted from it.

Each ticket is written as if we are building this for the first time — because we are. What the spike
gives us is **knowledge, not code**: proof that every piece can be made to work, and a record of every
trap we hit getting there. Read it before starting a ticket; do not copy out of it.

Two things follow:

- **Every ticket carries what the spike taught us.** The traps are named in each ticket by gotcha
  number. That list is the main defence against paying for the same bug twice — several were bugs
  that produced a *completely green deploy with every API call broken*.
- **Where the codebase already solves a similar problem, follow it.** Those patterns are named in
  each ticket. We are not inventing new approaches for problems the estate has already answered.

### Spike shortcuts that must not appear in the production build

| Shortcut taken in the spike | Handled by |
| --- | --- |
| Component named `ApimSpike`, short name `apimgs`, modules `apim_spike_*` | **1** — the new component is `APIM`, once the old one is deleted |
| Region hard-pinned to `westeurope` | **1** — becomes a config setting defaulting to the hub region |
| `skip_role_assignment=True` | **1** — left on |
| ARM called directly via `local.Command` in `interface/` | **1** — dynamic provider, or dropped if it is a no-op on our tier |
| `openapi.v1.json` (309 KB) bundled in the repo, plus an artifact fallback | **2** |
| Auth's portal redirect URIs declared with no environment switch | **5** |
| Portal clone script copied into the component's hooks folder | **6** |

---

## The tickets

| # | Ticket | Ends with *"you can now…"* | Blocked by |
| --- | --- | --- | --- |
| **1** | Foundation — the production APIM component | …deploy an empty, correctly-named APIM through the orchestrator | — |
| **2** | The OpenAPI spec supply chain | …import the spec the Core build produced, with no copy in the repo | — *(parallel with 1; long lead)* |
| **3** | The API surface, routing and maintenance responses | …call the API through the gateway, and get a clear answer mid-deploy | 1, 2 |
| **4** | Request signing and Core configuration | …prove Core accepts signed and rejects unsigned | 1, 3 |
| **5** | Developer portal — sign-in | …sign in to the portal with a Dayshape account | 1 |
| **6** | The master documentation site *(mostly non-coding)* | …deploy a branded portal from a durable, owned master | 5 |
| **7** | Observability and alerting | …see traffic, and get paged when it breaks | 1, 3 |
| **8** | Rollout, handover and per-customer enablement | …turn APIM on for a named customer, hands-off | all |

---

## 1 — Foundation: the production APIM component

> **Setup guide:** Phases 1 and 2 — *Create An API Management Service*, *Clean Up Default Resources*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

Branch from `iac-main` and build a **new** component called `APIM` from scratch, with the spike open
beside you as a reference. Everything else in this epic builds on this ticket.

### What "done" looks like

An `APIM` component on a branch off `iac-main` that deploys an **empty** APIM instance through the
orchestrator. That is all. No API, no policies, no signing, no portal — those arrive in later
tickets. The old proof-of-concept component is gone, nothing is pinned to a single region, and
nothing is switched off to make deploys quicker.

### The asks

**Remove the old APIM component.** First, because it holds the name we want.
`Products/Spoke/APIM/` on `iac-main` is the VDP proof-of-concept SRE told us not to build on (see
[[Decision - do not build on existing APIM IaC]]). It is **not** orphaned — three things reference
it:

- the 21 files under `Products/Spoke/APIM/`
- its registration in `Products/Modules/Globals.py` — the `product_component_base(name="APIM",
  short_name="apim", …)` line and its entry in the components dict below it
- its debug entry in `Products/.vscode/launch.json`

Its last commit looks recent, but that was a bulk manifest sweep across many components rather than
anything APIM-specific.

**Build the new component and name it `APIM`.** Empty instance, nothing else.

**Make the region a config setting that falls back to the hub region.** *Differs from the spike,
which pinned `westeurope`.*

**Leave role assignment on.** *Differs from the spike, which set `skip_role_assignment=True`.*

**No direct Azure calls from the component.** The spike clears Azure's default API and products —
guide Phase 2 — by shelling out through a `local.Command` in `interface/apim_arm.py` and
`apim_defaults.py`. That breaks our rule that *"Pulumi components should not interact with Az APIs
directly"*.

> *Suggested approach: check first whether those defaults are even created on our tier. If not, drop
> the step. If they are, write it as a dynamic provider — worked examples in `DynamicProviders/Apim/`.*

### Traps not to repeat

- APIM resources return far fewer outputs than you pass in. Unit tests written assuming symmetry fail
  in confusing ways.
- Azure holds a deleted APIM name for several hours. Destroy-and-recreate while testing and you are
  blocked, with an error that does not say why.
- Child resources under APIM need an explicit name or the whole deploy is rejected.

> [!warning] These remain live and unrelated
> A "delete anything APIM" sweep would break working things:
> `Products/Pipelines/Tools/SRE-Tagging/main.py` (matches the `microsoft.apimanagement` resource type
> generically, nothing to do with the component) · `Dayshape.Apim.Local/` ·
> `Dayshape.Apim/CustomWidget/` · Core's `RequireApimOriginAttribute`, `APIMRateLimit*`,
> `ProxyApiMapper*` · `Dayshape/Dayshape/Tools/ApimContentClone/`.

---

## 2 — The OpenAPI spec supply chain

> **Setup guide:** no phase of its own — it supplies the OpenAPI file that Phase 7 imports.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

**Start this early, alongside ticket 1.** It has a dependency nothing else can shorten, and it is
the reason this is not folded into ticket 3 — the bundled fallback cannot be deleted until the
publish pipeline is on `develop` and a Core build has actually produced an artifact. Kept separate,
ticket 3 is unblocked sooner.

- Build the publish step into the Core release — a script and pipeline definition on `develop`, wired
  into `Dayshape.Core.Release.yml`. The spike proved the shape of this end to end.
- Have the component read the published artifact from `dsartifactrepo`, and validate that every HTTP
  operation carries a stable identifier APIM can derive names from.
- **No bundled `openapi.v1.json` and no fallback path in the production build.** Anything interim can
  only be removed **once the pipeline is on `develop` and a build has produced an artifact** — until
  then the component has nothing to read.

**Non-coding**

- Confirm who owns the `dsartifactrepo` blob store, and what the retention policy is for
  `openapi/{version}/`. **A spec that ages out is a deploy that fails.**

---

## 3 — The API surface, routing and maintenance responses

> **Setup guide:** Phases 3, 6 and 7 — *Configure Service-Level Policy*, *Create Products*,
> *Create entities API with Version Set*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

- Entities API, version set, product, developers group, and the policies — service-level, rate limit,
  CORS, user-oid.
- **Maintenance responses.** Dayshape shows a *"down for maintenance"* page when Core is deploying.
  **The gateway has no equivalent**, and the reality is worse than expected: Core's maintenance mode
  is a **port swap, not a shutdown**, so a second IIS site takes over 443. APIM's call reaches IIS,
  finds no such path, and returns a **404 with an HTML page**. A 404 means *"this endpoint doesn't
  exist"* — a well-behaved integration will **not retry it**.
  Agreed shape: **two distinct 503 responses** — `DAYSHAPE_MAINTENANCE` when Core is deploying,
  `GATEWAY_MAINTENANCE` when APIM is — so a support ticket tells us where to look. Plus an honest
  third state: if APIM itself is genuinely gone, nothing can answer at all.
  Design sketch: [[APIM IaC - maintenance responses (design sketch)]]. Policy call is
  [[D10 - What customers see during a deploy window|D10]].

> [!danger] Learned the hard way — **G39**
> APIM's version set uses the `Segment` scheme, so it strips **both** the API path and the `v1`
> before forwarding — while Core's proxy routes are `proxy/v1/...`. A backend of `/proxy` therefore
> delivers `/proxy/grades`, and Core answers *"Proxy API version segment 'grades' is not supported"*.
>
> Fixed on the spike by deriving the backend from the same version constant the API is given, so the
> two cannot drift when v2 arrives. **This produced a fully green deploy with every call broken.**

---

## 4 — Request signing and Core configuration

> **Setup guide:** Phase 5 — *Create HMAC Signing Secret*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

The most intricate ticket, and the one with the most ways to fail silently. Two separate mistakes here
produced a fully successful deploy with every API call failing.

APIM generates the HMAC secret into Key Vault, and Core reads it through an **App Configuration Key
Vault reference**. **Nothing is written to an Azure DevOps variable group.** This is the established
house pattern — LanguagePack, Solver and CognitiveSearch all hand values to Core this way — and it
was confirmed by Sam Clifford on 2026-08-18 and verified against `ConfigurationFactory.cs`.

Choosing that route removes four separate problems at once:

| Problem | Status |
| --- | --- |
| `403` — no permission to write the variable group (**G42**) | Gone. Nothing is written to Azure DevOps |
| Core takes its copy before APIM writes it (**G40**) | Gone. Nothing is copied |
| Rotation needs a Core redeploy | Gone. Unversioned reference, picked up on next start |
| Secret sits in plain text in a variable group | Gone. It only ever lives in Key Vault |

**Scope**

- The component creates the Key Vault reference as a Pulumi resource — stateful, so Pulumi removes it
  on destroy.
- The `AppCfgKeyValueSet` extension for an optional `secret_id`, which emits the `content_type` and
  `{"uri": ...}` an App Configuration Key Vault reference needs.
- Vault read access for all three Core hosting variants — `CoreLegacyVM`, `Core` and `CoreAppService`.

> [!danger] Learned the hard way
> - **G40** — *"Core" is three component names* (`Core`, `CoreLegacyVM`, `CoreAppService`) and an
>   environment registers only the one it uses. Checking for one silently skipped the secret and
>   403'd every API call. The only trace was a single information line in a 3,800-line log.
> - **G41** — variable groups are named from the **raw** environment values, not the remapped ones
>   used for naming Azure resources.
> - **G43 / G44 / G45** — why App Config is the right route; how vault access is actually granted
>   (**membership of an Entra group named after the vault**, not a role assignment — and member
>   *listings* return empty for our accounts whether or not there are members, while membership
>   *checks* return the truth); and why the stateful resource beats the imperative SDK call.
> - **The key name.** App Config strips the `spk-core-software-` prefix and turns dashes into colons,
>   so `spk-core-software-Dayshape-ApimHmacSecret` arrives at Core as `Dayshape:ApimHmacSecret` —
>   which is what `RequireApimOriginAttribute.cs` looks for. **Getting it wrong fails silently.**
>   Keep the test that fails if the name drifts.

**One constraint that survives the redesign**

Core reads App Configuration **once at startup**, not on refresh. So the requirement is *"the value
must exist before Core next starts"* — and if APIM is deployed where Core is already running,
**Core needs a restart**.

**Non-coding**

- **Rotate the Dreich HMAC secret** that leaked into a build log. Spike cleanup, tracked against
  Grant — deliberately not an ask on the ticket.
- Get the Core-restart requirement into the enablement runbook (ticket 8).

---

## 5 — Developer portal: sign-in

> **Setup guide:** Phase 4 — *Configure Entra ID Identity Provider and Authorization Server*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

- Entra app registration and the portal sign-in secret. Pulumi has no native support, so this needs a
  dynamic provider. `Products/DynamicProviders/AGW/Certificate/` is the closest existing model on
  `iac-main` — the delegates / models / provider split, with tests — and the spike's
  `apim_portal_signin_provider` is a working example of the same shape.
- [[D1 - The developer portal sign-in secret|D1]] is **resolved** — the secret renews itself, so
  there is no expiry to manage. Carried as a note on the ticket, not an ask.
- **Add the portal's redirect URIs to `auth_component.py`, behind an `apim_enabled` guard.** On
  `iac-main` there is nothing there today — this is new on Auth, not a change to something existing.
  The spike declared them unconditionally so a spike environment worked without a separate config
  step. Do not do that here: it leaves every environment that deploys Auth carrying sign-in URLs for
  a gateway it does not have.

> [!warning] Learned the hard way — **G6**
> `azuread.ApplicationRedirectUris` is **authoritative** for the type it manages. With the guard in,
> deploying Auth for an APIM-enabled customer from anywhere the flag has not resolved to true
> will **delete the portal's redirect URIs** and break portal sign-in until the next correct run.
>
> Worth questioning as part of this ticket whether an APIM feature flag belongs on Auth at all. It is
> easy to forget on a new environment, and it fails looking like an APIM fault rather than a missing
> setting.

---

## 6 — The master documentation site *(mostly non-coding)*

> **Setup guide:** Phases 9 and 10 — *Publish Developer Portal*, *Import developer portal content
> from sandbox instance*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

**The largest remaining risk, and barely any of it is code.**

Every customer's documentation portal is copied from **one master instance — `vprk2`, in a personal
sandbox subscription, with no owner.** That is deliberate, not an accident: the portal is edited
through a drag-and-drop visual editor, so there is no file format to author in — the instance *is*
the design tool. Our technical authors were given edit rights in July 2026 specifically so they could
write customer-facing documentation there, so this is the **publishing pipeline for customer
documentation**, not a one-off bootstrap.

We checked whether this could be avoided entirely. **It cannot** — Microsoft holds the images in
storage we don't own and can't reach except through a temporary link. Written up at
[[APIM IaC - portal content, why there is no IaC-native route]]. **Ownership is the mitigation, not
avoidance.**

> [!danger] The day this ships, that sandbox becomes production
> Not "production-like" — actually production. Every customer deployment reads from it. If it
> disappears:
> - **No customer can be deployed or updated** until it's rebuilt
> - The content **cannot be recovered** — no version history, and Azure's own APIM backup does not
>   cover portal content
> - Rebuilding means our technical authors redesigning every page by hand in a browser
>
> There is currently **nothing** stopping that — no owner, no tag, no lock. *(**G24**)*

**Scope — the site itself**

- [[D2 - The master documentation site|D2]] **Part 1** — who owns it, and move it somewhere durable.
  Tag, lock and name it so it cannot be swept in a cost exercise. The estate already applies a
  read-only lock to production resource groups via `DayshapeComponent.generate_management_lock()` —
  match that convention, but note it only fires for resource groups the framework deploys, and
  `vprk2` is not one of them.
- **D2 Part 2** — what permission the deployment identity gets to read it.
- **D2 Part 3** — whether that grant lives in code or is applied by hand.
- **Repoint** the clone configuration off `vprk2-sbx-spk-apim`.
- **A backup and versioning story, because there is none today.** The desired state is whatever is
  live on that instance at the moment of the deploy. Two customers deployed a week apart can receive
  different portals, with no way to tell afterwards and no way to roll a customer back.

**Scope — the content copy**

- **Give the clone script one home.** `Dayshape/Dayshape/Tools/ApimContentClone/clone-apim-portal.ps1`
  is the original and is the only copy on `iac-main`. Decide deliberately whether that stays its home
  or it belongs inside the component — the spike ended up running a modified second copy, and two
  copies of an unsupported migration script is the worst outcome.
- **Decide whether a failed branding copy should fail the deployment.** Today a cosmetic problem stops
  the whole run. On 2026-08-17 a Dreich deploy was lost to a single momentary `Bad Gateway` reading
  the master, after Auth, APIM and everything else had already succeeded; replaying the identical call
  straight afterwards worked. The spike settled on retrying transient failures (three attempts, 5s then
  15s) but not permission failures, which worked well. **A warning, with the customer keeping the
  portal they already had, may be the better trade.**
- [[D7 - Accepting an unsupported content migration|D7]] — accepting an unsupported migration.

---

## 7 — Observability and alerting

> **Setup guide:** Phase 8 — *Configure Application Insights Diagnostics*, plus the unnumbered
> *Setup alerts in production* section at the end of the guide.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

Fully designed and proven on the spike, so the unknowns here are small. **The substance of this ticket
is validation, not design.**

- Phase 8 diagnostics — logger, API diagnostic, diagnostic setting, and an availability web test that
  pings the gateway from four locations outside Azure.
- The **seven alert rules** from the setup guide — `5xx_rate`, `get_latency`, `post_latency`,
  `unavailabilities_latency`, `failure_rate`, `rate_limited` and `capacity`. Six are log queries; the
  seventh, capacity, is a metric rule. Full detail in
  [[APIM IaC - diagnostics and monitoring (phase 8)]] §13.
- **The per-customer tuning surface.** Every field of every rule — `enabled`, `severity`, `threshold`,
  `window_size`, `evaluation_frequency`, `action_group_id`, `query` — should be overridable in App
  Config using the estate-wide `alert-<name>-<field>` convention. **ContainerApp, Reporting and SqlDb
  all do exactly this — follow them.** Defaults live in **code, not in `infra_defaults.yml`** — again
  the estate pattern, with Reporting as the precedent. Without this, calming a noisy rule for one
  customer is a code change for all of them.

> [!danger] The validation is the substance of this ticket, not a formality
> **A rule whose query matches nothing is indistinguishable from a healthy service.** Azure will not
> tell you a KQL filter is wrong — the rule simply never fires, forever.
>
> Two assumptions were verified before building (`ENTITIES_API_PATH = "entities"` so the `/entities/`
> filter matches real traffic; `/unavailabilities` genuinely exists in the spec). **That is not
> enough.** Every query must be run against real gateway logs and shown to return rows when it should.

**Two deviations to confirm rather than silently inherit**

1. Alerts route to **shared platform action groups**, not a per-customer group emailing
   `forth-team@dayshape.com` as the guide suggests. Consistent with every other component, and quiet
   in sandbox/dev for free. Trade-off: platform on-call is less familiar with the API than the Forth
   team would be.
2. **Severity is mapped** — Error pages, Warning goes to `p2`, Informational to `p2_4` — rather than
   everything routing to `p1` as the two existing in-repo examples do.

> [!warning] Learned the hard way — **G21**, **G22**, **G46**, **G47**
> The availability web-test traps (no return value; one web test per component maximum; the assertion
> has to anchor on a later resource) and the async test-assertion trap.
>
> **G47** is the one most easily reintroduced: the queries must carry **no time filter of their own**.
> An `ago(...)` line inside the KQL duplicates the rule's `window_size`, and only one of the two is
> reachable from config — widen the window and the query silently keeps reading the old range. No
> component in the estate filters on time inside the query.

**Non-coding**

- **Nobody has costed this.** Six log alerts evaluating every minute, per customer, is a recurring
  Azure Monitor cost that scales with API adoption.
- Confirm platform on-call are willing to receive these.
- Low-traffic customers are **silent by design** — the 5xx and failure-rate queries require 20+
  requests in the window. Intended, but it should be a stated expectation rather than a surprise.

---

## 8 — Rollout, handover and per-customer enablement

> **Setup guide:** Phase 11 — *Post-Deployment Verification*.
> [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)

- The per-customer enablement runbook, and **retiring the manual Confluence guide** that this whole
  piece of work replaces.
- Document the **Core restart** requirement from ticket 4.

> [!important] The smoke test is the point of this ticket
> **Twice now we have had a fully successful pipeline run with every API call broken** — first the
> routing (**G39**), then the signing secret (**G40**). Both were found by hand, both after a green
> deploy.
>
> **One authenticated call through the gateway asserting a 200** is cheap, and it is the only thing
> that would have caught either.

**Where the smoke test belongs**

`after_deployment` in `Products/Spoke/<component>/hooks/component_deployment_hooks.py` — eleven
components already use it. `_Template` shows the bare shape; LanguagePack is the worked example.

**Non-coding**

- [[D5 - Getting access to test on a real environment|D5]] — access to test on a real environment.
- Handover to whoever runs customer enablement.

---

## The non-coding workstream, in one place

| Item | Ticket | Who is needed |
| --- | --- | --- |
| Update three docs pointing at the old component *(parked — deliberately not on ticket 1)* | 1 | — |
| Blob store ownership and retention for published specs | 2 | SRE |
| [[D10 - What customers see during a deploy window\|D10]] — deploy-window behaviour | 3 | PM · TL · SRE |
| Rotate the leaked Dreich HMAC secret | 4 | Grant |
| Whether an APIM feature flag belongs on Auth at all | 5 | TL · Auth owner |
| [[D2 - The master documentation site\|D2]] — ownership, move, permission, grant location | 6 | SRE · Security · PO · TL |
| [[D7 - Accepting an unsupported content migration\|D7]] — unsupported migration | 6 | PM · TL |
| A backup/versioning story for portal content | 6 | SRE · PO |
| Cost of alert evaluation per customer | 7 | SRE |
| Platform on-call accepting these alerts | 7 | SRE |
| [[D5 - Getting access to test on a real environment\|D5]] — access to a real environment | 8 | SRE |
| Retire the manual Confluence runbook | 8 | PO |

> [!important] For the Product Owner
> **Roughly half the remaining risk is not code.** It is ownership, permissions and decisions — and
> every one of them needs somebody outside this team.

---

## Sequencing

- **1 and 2 start together.** 2 has a lead time nothing else can shorten — the bundled spec cannot be
  deleted until a real Core build has published one.
- **3 and 5 can run in parallel** once 1 lands. They touch different parts of the component.
- **4 needs 3** — there is no point proving signing until a call routes correctly.
- **6 should start its decisions early even though its code comes late.** D2 needs SRE, Security, the
  PO and a TL; that conversation has a longer lead time than the work does.
- **8 is last by definition.**

---

## Open questions

- **APIM pricing is still unverified.** BasicV2 is the agreed tier, but nobody has confirmed the
  per-customer cost, and it scales with every customer that moves across.
- **No performance targets exist**, and no load test has been run. The latency thresholds in ticket 7
  came from the setup guide, not from a measured baseline.
- **Should the seven alert rules be per-customer or shared** once this is productionised?
- **Does Infra Platform want the AppCfg `secret_id` change as it stands?** It extends a shared module,
  and they own `/Products/`.
- **Three repo docs carry stale paths to the old component** — `docs/adr/004-apim-origin-validation.md`
  (line 209), `docs/runbook/001-Local-Development-APIM-Proxy.md` (lines 172, 205) and
  `docs/runbook/002-APIM-Nswag-OpenApi-generation.md` (line 11). All three describe live things and
  are still maintained, so only the paths are wrong. **Deliberately left out of the tickets for now.**
  When picked up: the ADR should not be edited — a decision record is a dated snapshot and the
  decision it records still stands. The two runbooks each become accurate at a different point
  (NSwag at ticket 2, the signing policy path at ticket 4), so they are not a single bulk edit.

---

## Related

- [[_Decisions needing sign-off - start here]] — the decisions awaiting sign-off
- [[APIM IaC - known traps and gotchas]] — the trap register the spike produced
- [[APIM IaC - diagnostics and monitoring (phase 8)]] — ticket 7 in full
- [[APIM IaC - maintenance responses (design sketch)]] — ticket 3
- [[APIM IaC - portal content, why there is no IaC-native route]] — why ticket 6 cannot be avoided
- [[Decision - do not build on existing APIM IaC]] — why the old component was deleted
- [[APIM IaC - outstanding questions]] — the wider open-questions register
