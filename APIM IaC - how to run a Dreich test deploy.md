---
title: APIM IaC - how to run a Dreich test deploy
created: 2026-08-11
status: no gates left - the client secret gate was removed on 2026-08-13
tags: [apim, iac, devo-28368, testing, dreich]
---

> [!warning] Audited 2026-10-07
> Component name is now `Apim` (not `Auth,ApimSpike`). Registry list: `ENV_STARTER_COMPONENTS` in `Products/scripts/domain/config_sync_service.py`. A component missing from the registry is silently skipped with a green pipeline.

# How to run a Dreich test deploy

Working notes for getting the APIM component onto a real environment. Based on Richard Donkin's
guidance, then checked line by line against the pipeline code.

> [!warning] Read the blockers first
> Started as **three** gates, then became four. As of 2026-08-13: gate 2 turned out to be a non-issue,
> **the client secret gate has been removed by a code change**, and gate 1 is with Marcus.
> See [[#What will stop you]].

---

## The short version

**Two pipelines, and you may only need the second.**

| | Pipeline | Why | Needed? |
| --- | --- | --- | --- |
| 1 | **Dreich release** — `Dayshape.Deploy.Internal-Dev-Dreich` | Puts the Dayshape app itself on Dreich, so APIM has something real to point at | Only if Dreich is stale |
| 2 | **Manual infrastructure deploy** — `IACVer.Orchestrate.Deploy` | Deploys **just the APIM component** from the spike branch | ✅ Yes |

Dreich already exists and is deployed, so in practice **this is a one-pipeline job.**

---

## Why not just run the Dreich release pipeline?

It was my first assumption, and it's the heavier option.

The Dreich release pipeline deploys **everything** — the app, the online help, the notification and
email services, the integrations, the reporting service, and it runs database migrations and stops
and restarts services along the way. Infrastructure is one job inside all that.

The manual infrastructure pipeline does the infrastructure only, and takes a **`components`** filter
so it can target `ApimSpike` and nothing else.

> [!tip] Same underlying code either way
> Both call the identical orchestration template. The manual one is a smaller doorway to the same room — much less that can go wrong, and much less to sit and watch.

---

## What will stop you

### 🔴 1. Dreich has not been told APIM exists

**This will not fail loudly — it will look like it worked.**

The deploy does not work from the components in the code. It works from a list held in the
**environment registry**: what this environment is expected to have. `components=ApimSpike` is a
**filter on that list**, not an instruction to deploy.

`ApimSpike` has never been registered against Dreich. So the filter matches nothing, **nothing
deploys, and the pipeline goes green.**

> [!danger] Expect a green run with no APIM
> No warning, no error. The natural reading is *"the deploy worked but APIM did not appear"* — and
> you would go hunting through Pulumi and Azure for a problem that is not there.

Registration is not self-service either. New environments get a fixed starter set that does not
include APIM, and the CLI that adds one is hard-blocked to the *dev* registry — Dreich is in the real
one.

**Ask Richard/SRE:** please register `ApimSpike` as an infra target for `drch` / `dv`.

> [!success] In hand as of 2026-08-12
> **Marcus is adding `ApimSpike` to the GSOT for Dreich.** Two things worth confirming with him,
> because both are easy to get wrong and both produce the same silent green run:
>
> 1. **It must go in the real registry** — `ds-env-config-sqldb`, *not* `ds-env-config-devdata-sqldb`.
>    The pipeline defaults to the real one.
> 2. **The component name must be exactly `ApimSpike`** — same capitalisation. The match is literal.

There is no way around this one. It is the single true prerequisite.

→ [[APIM IaC - known traps and gotchas|G31]]

#### How to check it actually landed

There is a ready-made view for this — `dbo.EnvComponentRolloutStatus`
(`scripts/apis/dsenv/views/env_component_rollout_status.sql`). It answers the registration question
and the deployment-history question in one go:

```sql
SELECT ClientId, Env, ComponentName, InfraTargetStatus,
       IacVersioningMode, LastDeployedDate, DeploymentSucceeded
FROM dbo.EnvComponentRolloutStatus
WHERE ClientId = 'drch' AND ComponentName = 'ApimSpike';
```

`InfraTargetStatus` returns `present` or `MISSING`.

> [!warning] "No rows" does not mean "missing"
> The view builds its component list with `CROSS JOIN (SELECT DISTINCT ComponentName FROM dbo.InfraTargets)`.
> If `ApimSpike` has never been registered against **any** environment, the name isn't in that list
> and you get **no row at all** rather than a `MISSING` row. Read an empty result as "not registered
> anywhere yet", not as a query mistake.

Grant can't run this himself (no database login — see below), so it's a question for whoever holds
the access.

#### How the registry actually works

It is a **SQL table**: `dbo.InfraTargets` in `ds-env-config-sqldb` on
`ds-platform-sqlserver.database.windows.net`. One row per environment per component.

Registering APIM for Dreich = **one row**: Dreich's `EnvId` + `ComponentName = ApimSpike`.

> [!warning] There are two registries, and they are not the same one
> `ds-env-config-devdata-sqldb` (**dev**) and `ds-env-config-sqldb` (**real**).
> `orchestrate-dev` always uses **dev**. Pipelines always use **real**.
> This is why the local sbx run worked without anyone registering anything — and why it is
> **not** evidence the pipeline route works.

#### Four ways to get that row in

| | Route | Who | Trade-off |
|---|---|---|---|
| **A** | SRE inserts the row directly | SRE | Fastest, no code. Manual, undocumented, invisible to review |
| **B** | Relax the dev-only guard on `ds-env-cli add-target-infra` | Us + review | The command already exists for exactly this; the block is one line. But that guard is presumably deliberate |
| **C** | Add APIM to `ENV_STARTER_COMPONENTS` and re-run **ConfigSync** | Us, in this repo | Fully in code, reviewable, idempotent. **But every environment then gets APIM** — right for the real rollout, too broad for a spike |
| **D** | Run `orchestrate-dev` locally | Us | Registers in the **dev** registry only. Does **not** help Dreich |

Route **C** is the only one that is self-service *and* leaves a reviewable trail — but it is a
product decision, not a test convenience.

**Correction:** an earlier version of this page said there was *no* self-service route. That was
too absolute — **C** exists.

#### Can Grant do it himself? Verified 2026-08-12

The table lives in `ds-platform-sqlserver` / `ds-env-config-sqldb`, resource group `ds-platform-db`,
subscription `a6128f37…` (**not** Sandbox — switch subscription in the portal or you will not see it).

Three things are needed to write that row. Tested each against live Azure:

| Need | Status |
|---|---|
| Network reach through the SQL firewall | ✅ **Already open** — rule `Grant1` matches the current IP, and there is a `SqlServer-FirewallContributor` role scoped to this server |
| A login on the database | ❌ **Denied.** Entra token reaches the server and SQL rejects it — `Login failed for user '<token-identified principal>'`. Not a member of `Forth`, the server's Entra admin group |
| SQL username/password from Key Vault | ❌ **Denied** on `dopprodkv` — no `secrets/get` |

For contrast, `dopdevkv` **is** readable. That is the whole explanation for the sbx run: SQL
authentication using credentials from the *dev* vault, against the *dev* registry.

> [!note] The gap is one permission, not a pile of them
> `secrets/get` on `dopprodkv` for `dsEnvConfig-ConnString`. That is the only missing piece.
> It is an **access-policy** vault (RBAC disabled, 19 policies), so the grant is a policy entry.

**But think about what that permission is.** The connection string is a SQL login with read/write over
the production golden source of truth — every environment, every component, not just Dreich. Asking
for it to unblock a spike is a much larger request than the thing it unblocks.

| | Ask | Size of ask |
|---|---|---|
| 1 | SRE runs one `INSERT` | Smallest. Two minutes of someone's time, no new access |
| 2 | `secrets/get` on `dopprodkv` | Self-serve forever — and full write over every environment's registry |
| 3 | A scoped SQL user limited to `dbo.InfraTargets` | Middle ground. More setup for SRE, far less privilege |
| 4 | Route C — starter set + **ConfigSync** pipeline | No personal access needed; the pipeline's identity already has it. But it touches app config and variable groups too, and changes every future environment |

**Portal route** *(for whoever does hold the access)*: Azure Portal → switch to the subscription above
→ **SQL databases** → `ds-env-config-sqldb` → **Query editor** → sign in → run:

```sql
SELECT Id, ClientId, Env, EnvName FROM dbo.EnvConfig WHERE ClientId = 'drch';

INSERT INTO dbo.InfraTargets (EnvId, ComponentName) VALUES (<that Id>, 'ApimSpike');
```

There is a unique constraint on `(EnvId, ComponentName)`, so a repeat insert errors rather than
duplicating. `ComponentName` must be exactly `ApimSpike`.

### 🟢 2. The branch override — RESOLVED, it works

**Checked properly 2026-08-12. My earlier worry here was wrong.**

I had assumed `iacBranchOverride` might be silently skipped if IaC versioning is switched on for
Dreich, which would have meant a green pipeline and no APIM. That isn't what the code does.

The override is resolved **first**, before the versioning mode is even looked at
(`iac_version_resolver.py`, `_resolve_override`). When it's set on an internal environment it returns
"no version, use a repo checkout" and **the GSOT versioning mode is bypassed entirely**. The code
comment calls this *"the override always wins"* behaviour.

**So we don't need to know Dreich's versioning mode.** That question can be dropped.

Is Dreich internal? The resolver counts two cases:

| Case | Rule |
| --- | --- |
| Current internal | `env_group` is `sbx` or `int` |
| **Legacy internal** | `client_group_id` is `sre` or `dev` **and** `env_group` is `lwr` |

Dreich is `dev-drch-ds-lwr-dv` → client group `dev`, env group `lwr` → **legacy internal**. Permitted.

> [!caution] One honest caveat
> That reading comes from decomposing the environment name, not from reading the GSOT row. If the
> run fails with `IacOverrideNotPermittedError`, this assumption is why. It's a loud, clear failure —
> not a silent one.

Two smaller details, both confirmed fine:

- The default value is `-`, not empty. Both the resolver call and the checkout step explicitly treat
  `-` as "not set" (`[ "$OVERRIDE_BRANCH" = "-" ] && OVERRIDE_BRANCH=""`). No risk of it trying to
  check out a branch literally called `-`.
- The override also overrides the Core artifact commit, and warns in the log when it does.

### ✅ 4. Nothing supplies the Entra client secret — **no longer applies**

**Found 2026-08-12 as a hard stop. Removed entirely on 2026-08-13.** Kept here because the app
registration facts below are still needed, and because the fix is worth knowing about.

Phase 4 needs a client secret belonging to the Core app registration. The code used to read it from an
environment variable, `APIM_IDP_CLIENT_SECRET`, which nothing set — so the run reached Phase 4 and
stopped.

**The APIM component now creates that secret itself,** as part of the deployment, and replaces it
every 30 days. There is no environment variable, no queue-time variable, no manual portal step and no
`env:` block edit. See [[D1 - The developer portal sign-in secret]].

> [!success] Three things went away with it
> - **The manual step** — nobody needs to mint a secret in the portal before a test run
> - **The YAML edit** — the run no longer has to come from the spike branch for this reason
>   *(the branch override is still a reason on its own)*
> - **The 12-month clock** — see [[APIM IaC - known traps and gotchas|G7]], now closed


#### Which app registration, and what is already on it?

**Investigated 2026-08-12.** Still worth keeping — the identifiers are needed for checking the result
of a run, and there is one leftover to clean up.

| | Finding |
| --- | --- |
| **App registration** | `dev-drch-ds-lwr-dv` |
| **Application (client) ID** | `7e65f249-82dc-4224-8ff4-15e8c0be2346` |
| **Object ID** | `bb9394dd-d0d4-43aa-9ded-256d19d4b31f` |
| **Existing secret** | ⚠️ One, named **`apim-auth`** — created 16 Jul 2026, expires 16 Jul 2027 |
| **Created by IaC?** | ❌ No. `apim-auth` appears nowhere in the repo — it was made by hand in the portal |
| **Registered owners** | None |
| **Grant's active directory roles** | None |
| **Grant in `PrivilegeElevationEnabled`** | ✅ Yes — PIM elevation is available to him |

**`apim-auth` is now litter and should be deleted** once the deploy has created its own. It was made
by hand during Grant's July APIM setup, isn't referenced by anything in the repo, and having a
hand-made credential sitting alongside a managed one is exactly the kind of thing that confuses the
next person. Deleting it needs PIM elevation — *App registrations → `dev-drch-ds-lwr-dv` →
Certificates & secrets*.

> [!note] Adding a secret was never the risky part
> An app registration can hold several credentials at once, and adding one does not disturb the
> others. That is also *why* the APIM component is allowed to add its own — see
> [[D1 - The developer portal sign-in secret]].


### ⚪ 3. There's no dry run

The orchestrator can do a preview — show what would change, change nothing. **The pipeline doesn't
offer it.** The first run is a real one.

→ [[APIM IaC - known traps and gotchas|G32]]

---

## Two things the ADO UI will ask you that are easy to miss

**1. Pick the Dayshape Core Release artifact.** The run panel shows a *Resources* section with the
`ClientRelease` pipeline. It supplies `CORE_SOFTWARE_VERSION`, which the API definition needs. Leave
it on the latest `minor-release` build unless there is a reason not to.

**2. Expect permission prompts on the first run.** The first time this pipeline is used with Dreich's
settings it will pause with *"This pipeline needs permission to access a resource before this run can
continue."* — once for the **service connection** and once for each **variable group** it has not used
before. Click **View → Permit**. If you cannot, someone with owner rights on those resources has to.

This is normal ADO behaviour, not a sign anything is wrong — but it is a stop-and-wait if you are not
the right person.

---

## Settings — the full list

| Field | Value |
|---|---|
| Branch to run the pipeline from | `spike/DEVO-28368-apim-investigation` |
| `client_id` | `drch` |
| `environment` | `dv` |
| `serviceConnection` | `Dayshape.DevOps.Orchestrator.Internal` |
| `iacBranchOverride` | `spike/DEVO-28368-apim-investigation` |
| `components` | `Auth,ApimSpike` — see the note below |
| `targetEnvironment` | `dev-drch-ds-lwr-dv` |
| `useDevGsot` | unticked |
| `destroy` | `-` |
| `approverEmail` | `-` |

`targetEnvironment` is the environment name from the Dreich pipeline definition
(`dev-drch-ds-lwr-dv`). It selects the variable groups, so a wrong value here means missing config
rather than a clean error.

> [!warning] `components` is a complete list, not a starting point — **G34**
> Naming `ApimSpike` deploys **only** `ApimSpike`. Its manifest declares `Storage` and `Auth` as
> dependencies, but the orchestrator **silently discards** any dependency not already in the list
> rather than adding it to the run (`component_ordering.py` L37-41). Dependencies control *ordering*
> within a run; they do not pull anything in.

> [!danger] ⛔ Why the 2026-08-12 run really failed — **G35**, and it is not fixed by code
> An earlier version of this guide blamed G34. **The full log disproved that** — the run was
> `all components`, so nothing was dropped. The real error was:
>
> ```
> No applications found matching filter: "displayName eq 'drch-dv-spk-auth'"
> ```
>
> Phase 4 looks the Core sign-in record up **by name**. On Dreich that name does not exist: the
> record is still called **`drch-dev-Spoke-auth`**, an older naming convention. Thirty other
> environments already use the current name — Dreich has simply not had `Auth` redeployed since it
> changed.
>
> *(It failed at 11:02 for a second reason too: `ApimSpike` had **no** manifest dependencies at that
> point — they were added at 12:50 — so it ran **before** `Auth`. That half is now fixed.)*
>
> **What this means for your run:** `Auth` **must** be in it. Running `ApimSpike` alone will fail
> with the same error. Running `Auth` renames the record to the current convention — verified safe,
> it is an **in-place rename**, the identifier does not change.

> **First deploy into a fresh environment:** `Auth` must have run at some point, because it declares
> the developer portal's return addresses. If it never has, use `Auth,ApimSpike`. After that,
> `ApimSpike` alone is fine — return addresses do not expire. **Not yet true on Dreich** — see G35
> above.

> [!tip] Why `Auth,ApimSpike` and not just `ApimSpike` for **this** run
> Auth has already been deployed on Dreich with the **old** code, so the credential and secure-store
> entry it used to create are still sitting there. Including `Auth` is what removes them, and that
> cleanup is one of the things this run is meant to prove.
>
> The order does **not** need changing — the orchestrator runs **Auth → APIM**, which is correct.
> Auth clears the old value first, APIM mints the replacement second, and portal sign-in is down for
> the **few minutes between the two**. On a sandbox that is a non-event.
>
> `ApimSpike` alone would also work and would have **no** sign-in gap, but it leaves the old value
> behind for a later run to tidy up. See [[D1 - The developer portal sign-in secret|D1 — the
> changeover]].
>
> ⚠️ **What not to do: deploy `Auth` on its own.** That removes the old value with nothing to
> replace it, and sign-in stays broken until APIM next runs.

---

## Before you run

- [ ] **Clear the "do not delete" flag on the old sign-in secret — G36.** Do this **first**, or the
      run stops on `Auth` before it does anything. One command, once per environment; you will need
      `source Products/tools/enable-pulumi.sh <client> <env> ds` first, or Pulumi cannot reach its
      state. Full syntax in [[APIM IaC - known traps and gotchas|G36]]. Only needed where the old
      code ran — **grant-sbx and Dreich**, nothing else
- [ ] **Check the APIM name is free, then wait — G37.** A previously deleted APIM reserves its own
      name for **48 hours** and blocks the rebuild:
      ```
      az rest --method post \
        --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.ApiManagement/checkNameAvailability?api-version=2022-08-01" \
        --body '{"name":"<apim-name>","type":"Microsoft.ApiManagement/service"}'
      ```
      `false` is a definite stop — list the soft-deleted services and either purge the entry
      (`--method delete` on the `id` returned) or wait out its `scheduledPurgeDate`.
      **`true` does not mean safe.** It only says the name is untaken; Azure can still be dismantling
      the old gateway and will reject the deploy with `ServiceLocked`. **Allow 30–45 minutes after
      any delete or purge before redeploying** — no API reports when teardown is actually done
- [ ] **Delete the hand-made APIM on Dreich.** Built manually while learning; the deploy expects to
      own what it finds, and leaving it risks a clash. **Note this also starts the 48-hour clock
      above** — purge it straight after deleting, or the deploy that follows will be blocked by it
- [ ] Confirm `ApimSpike` is registered against `drch`/`dv` in the **real** GSOT *(gate 1 — with Marcus)*
- [ ] ~~Confirm IaC versioning is off for `drch`~~ — **not needed**, the branch override bypasses it *(gate 2, resolved)*
- [ ] ~~Create a client secret on `dev-drch-ds-lwr-dv` via PIM elevation~~ — **not needed**, the deploy
      creates its own as of 2026-08-13 *(gate 3, removed)*
- [ ] ~~Add it as a **secret** queue-time variable and map it in the `env:` block~~ — **not needed**,
      same reason
- [ ] Push the spike branch
- [ ] Set aside **~45 minutes** — a fresh APIM is slow to create

## After you run

- [ ] Check the APIM component created a secret named **`apim-devportal-signin`** on
      **`drch-dv-spk-auth`** (the record `Auth` will have renamed — see **G35**), and that portal
      sign-in works
- [ ] **G35 follow-up:** confirm whether the portal's **"Try it"** call against the real API is
      accepted. Dreich's Core is configured to trust the *legacy* record `dev-drch-ds-lwr-dv`, not
      the one APIM uses. If it is rejected, that is G5/G35 — not a fault in this change
- [ ] Confirm the **old** value Auth used to create is gone — both the credential on the sign-in
      record and the `apim-devportal-client-secret` entry in the secure store
- [ ] **Delete the leftover hand-made `apim-auth` secret** — note it is on the **legacy**
      `dev-drch-ds-lwr-dv` record, not the one APIM now uses. Needs PIM elevation
- [ ] Re-run with no code change and confirm the secret is **not** replaced. A secret that churns on
      every deploy is as broken as one that never rotates

---

## What good looks like

1. An APIM instance appears in Dreich's resource group
2. The API is published and visible in the developer portal
3. **The developer portal is styled** — that's Phase 10, the content copy, and the part most likely
   to fail elsewhere
4. **An API call actually works end to end** — the real prize, and the thing sandbox could never show

> [!success] Phase 10 should pass here
> `…Orchestrator.Internal` can read the content master. Note **why**: it has blanket admin over the
> sandbox subscription for unrelated reasons, not because anyone granted it.
>
> **The same step would fail in production**, which has no access at all.
> → [[APIM IaC - known traps and gotchas|G30]] · [[D2 - The master documentation site|D2]]

---

## Related

- [[D5 - Getting access to test on a real environment]]
- [[APIM IaC - known traps and gotchas]] — G29, G30, G31, G32
- [[APIM IaC - rollout and testing strategy]]
