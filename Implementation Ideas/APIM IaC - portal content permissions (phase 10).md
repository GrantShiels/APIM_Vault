---
tags:
  - APIM
  - IaC
  - phase10
  - permissions
  - rbac
ticket: DEVO-28368
status: verified against Azure — grant needed before production
date: 2026-08-11
updated: 2026-08-11
---

> [!warning] Audited 2026-10-07
> Belongs to DEVO-30004 / DEVO-31423. Role-creation needs Owner/UAA (D2).

# APIM IaC — portal content permissions: what the deploy needs to be allowed to do

> [!abstract] The whole thing in four sentences
> Copying the developer portal from our master instance onto a customer's needs the deployment
> to be allowed to **read** that master. A `Reader` role is **not enough**, and the role whose
> name sounds exactly right does not work either. If we get this wrong the copy half-succeeds —
> the customer gets every page and **no images** — and the error will not point at the cause.
> Four deployment identities need the grant, not one.

Part of [[APIM Findings Summary]] · Follows [[APIM IaC - developer portal content (phase 10)]] ·
Trap recorded as G27 in [[APIM IaC - known traps and gotchas]]

**Everything below was verified against live Azure on 2026-08-11, not inferred from documentation.**

---

## Part 1 — Why permissions are suddenly a question

While Phase 10 was a person running a script on their laptop, permissions were invisible: the
person was already signed in and already had access to everything.

Automating it changes that. The copy now runs as a **service account** inside a deployment
pipeline, and that account only has the access we have explicitly given it. It has to reach
*across* from the customer's environment into the sandbox that holds the master content — which
is the one and only place a customer deploy steps outside the customer's own subscription
(see G24).

So the question "what exactly does it need to be allowed to do?" has to be answered before
this can run anywhere near a customer.

---

## Part 2 — The trap

The tool copies two different kinds of thing, and — this is the important bit — it fetches them
by **two different mechanisms**:

| What | How it's fetched | Kind of permission |
| --- | --- | --- |
| Pages, text, styling | `GET .../contentTypes/...` | An ordinary **read** |
| Images, logos, media | `POST .../portalconfigs/default/listMediaContentSecrets`, which hands back a temporary storage link | **An action** |

The second row is the trap. In Azure's permission model an *action* is a separate thing from a
*read*, and read-only roles do not include it.

From Microsoft's own toolkit (`scripts.v3/utils.js`, at the tag we pin, `2.34.0`):

```js
getStorageSasUrl() {
    const response = await this.httpClient.sendRequest(
        "POST", `/portalconfigs/default/listMediaContentSecrets`);
    return response.containerSasUrl;
}
```

That maps to the Azure permission:

```
Microsoft.ApiManagement/service/portalConfigs/listMediaContentSecrets/action
```

### Which roles actually grant it

Checked against Azure's live role definitions — every plausible candidate:

| Role | Grants it? |
| --- | --- |
| `Reader` | ❌ No |
| `API Management Service Reader Role` | ❌ No |
| `API Management Developer Portal Content Editor` | ❌ **No** |
| `API Management Service Operator Role` | ❌ No |
| `API Management Service Contributor` | ✅ Yes |
| `Contributor` | ✅ Yes |

Note the third row. The role named after the exact job we are doing does not grant it. Anyone
reasoning about this from role names alone — including a future us — would pick that one and it
would not work.

> [!danger] The failure is partial, and it arrives late
> Under a read-only role the text and styling copy **successfully**. The run then fails when it
> reaches the images. The customer is left with a portal carrying every correct page and no
> logos, and the error message concerns the customer's deploy rather than the source instance
> it could not read.
>
> This is the shape of problem that gets found by a customer rather than by us.

---

## Part 3 — Where the master lives

| | |
| --- | --- |
| Subscription | `58d847ff-52c8-4127-b663-1232b8fbbd48` — **"Dayshape Sandbox"** |
| Resource group | `vprk2-sbx-spk-apim-rg` |
| Resource | `vprk2-sbx-spk-apim` |
| SKU / region | BasicV2 · West Europe |

Worth being explicit about one thing, because it is easy to assume otherwise: **this subscription
is ours.** It is a Dayshape sandbox, not a Voodoo Park tenant. Granting access is entirely within
our control — there is no third party to negotiate with and nothing to wait for.

That is good news for delivery, and it sharpens the point already made in G24: once this ships,
`vprk2` is a **production dependency**. Every customer deploy reads from it. It must not be
deleted, paused, renamed, or swept up in a sandbox cost-cleanup, and it needs a named owner.

---

## Part 4 — What to grant on the master

Two ways to do this.

### Option 1 — `API Management Service Contributor`, scoped to the `vprk2` resource

One command, an off-the-shelf role, works immediately.

> [!caution] What it also grants
> It grants **write** on our content master. The tool has a `cleanup` step whose job is to
> **wipe** whichever instances are listed as targets. If that target list were ever generated
> incorrectly and named `vprk2`, a Contributor-level identity would be *permitted* to erase the
> master — the thing our technical authors write into.
>
> A read-only identity could not do this, however wrong the configuration was.

### Option 2 — a custom role ⭐ recommended

Grants precisely what the copy needs and nothing else:

```
Microsoft.ApiManagement/service/read
Microsoft.ApiManagement/service/contentTypes/read
Microsoft.ApiManagement/service/contentTypes/contentItems/read
Microsoft.ApiManagement/service/portalConfigs/read
Microsoft.ApiManagement/service/portalConfigs/listMediaContentSecrets/action
```

All five confirmed to exist via `az provider operation show`. No separate storage permission is
needed — the temporary link handed back by that last action carries its own access with it.

**Why I would pick this one:** not for tidiness. The source is our single content master, with
no version history and no backup (APIM's own backup does not cover portal content — see G26).
Least privilege turns "cleanup wipes the master" from *unlikely* into *impossible*, and the
extra work is a single role definition.

### On the target — the customer's own APIM

Needs read, write and delete on content, plus the same media action. That is covered by
`API Management Service Contributor` — and the deployment identity **already creates the APIM
instance itself**, so it will almost certainly hold this today.

**Expected to need no change.** Confirm on the first run rather than assuming.

---

## Part 5 — Which identities need it

Deployments run as Azure DevOps service connections, and there are **four**.

> [!danger] Verified against live Azure, 2026-08-11 — two have it by accident, two not at all
> | Service connection | Covers | Access to `vprk2` today | How |
> | --- | --- | --- | --- |
> | `…Orchestrator.Internal` | Internal *(incl. Dreich)* | ✅ | `Contributor`, **inherited from the whole subscription** |
> | `…Orchestrator.Sandbox` | Sandbox | ✅ | `Contributor` + `Owner`, inherited |
> | `…Orchestrator.Lower` | Lower customer envs | ❌ **none** | — |
> | `…Orchestrator.Upper` | 🔴 **Production** | ❌ **none** | — |
>
> **Not one role assignment is scoped to the master itself.** Both working identities work by
> side-effect of broad subscription rights held for unrelated reasons.

This is worse than a plain missing permission, in three ways:

1. **The two that work are the two we test with.** Dreich and sandbox pass; **production fails on the
   first real customer**, part-way through a deploy.
2. The access being relied on is **subscription-wide `Contributor`** — far broader than the five
   rights needed, and it includes exactly the write access that makes the "cleanup wipes the master"
   scenario in Part 4 possible.
3. It is **incidental**, so it could be tightened for entirely sound unrelated reasons and silently
   break every customer deployment. Nothing anywhere records that APIM depends on it.

> [!important] This settles Option 1 vs Option 2
> Option 1 was "grant `API Management Service Contributor`". But `Internal` and `Sandbox`
> effectively have that already by inheritance — and it is precisely what we would want to *remove*.
>
> **Option 2, the custom role explicitly assigned to all four, is now clearly the right answer.** It
> closes the two gaps *and* removes the reliance on inherited `Contributor` for the two that
> currently work by luck.

Scope each assignment to the **`vprk2` APIM resource itself** — not the resource group, not the
subscription.

→ Recorded as G30 · decision [[D2 - The master documentation site|D2]]

---

## Part 6 — How we prove it, rather than hope

### ✅ Step 1 — confirm the mechanism works at all *(done)*

Both calls were run successfully against `vprk2` using an ordinary Dayshape login:

- `GET .../contentTypes` → returned **9 content types**
- `POST .../portalconfigs/default/listMediaContentSecrets` → returned a **valid storage link**

So the approach itself is sound. **Whether the deployment identities have the same rights is now
answered for two of the four — see Part 5. Two of them do not.**

### 🟡 Step 2 — test as each service principal *(partly done)*

**Already established by role-assignment inspection** *(Part 5)*: `Lower` and `Upper` have no access
at all, so they will fail. `Internal` and `Sandbox` hold inherited `Contributor`, which *should*
cover it.

What inspection **cannot** tell us is whether inherited `Contributor` genuinely satisfies the media
call in practice. This job settles that. It calls the media action and nothing else, so it is
**read-only and changes nothing**:

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: 'Dayshape.DevOps.Orchestrator.Upper'   # repeat per connection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      SUB=58d847ff-52c8-4127-b663-1232b8fbbd48
      RG=vprk2-sbx-spk-apim-rg
      SVC=vprk2-sbx-spk-apim
      az rest --method post \
        --url "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$SVC/portalconfigs/default/listMediaContentSecrets?api-version=2021-08-01" \
        --query containerSasUrl -o tsv > /dev/null \
        && echo "PASS - media access OK" \
        || { echo "FAIL - missing listMediaContentSecrets on vprk2"; exit 1; }
```

A `403` here is the *exact* failure a real deployment would hit, caught in seconds instead of
part-way through a customer deploy.

### ⬜ Step 3 — make the deployment itself fail early and legibly

Add the same check to the deployment hook **before** it begins copying, so a missing permission
produces:

> `Cannot read portal content from vprk2 — check access`

rather than a half-finished portal and a misleading error. This is the same principle as G25:
never let a bad read reach the point where it can do damage.

---

## Part 7 — Actions

- [x] ~~Decide Option 1 vs Option 2~~ — **settled: Option 2, the custom role.** Verifying access
      showed two identities already hold inherited `Contributor`, which is the over-broad access we
      want to stop depending on *(Part 5)*
- [ ] **Define the custom role** — name, the five actions, assignable scope *(draft in
      [[APIM IaC - deploying to a new environment (prerequisites)|the prerequisites doc]])*
- [ ] Apply it to **all four** orchestrator service principals, scoped to the `vprk2` resource only
- [ ] Run the Step 2 check against `Upper` **before** any production use
- [ ] Add the pre-flight check into the deployment hook
- [ ] Record a **named owner for `vprk2`** and mark it do-not-delete — raise with SRE (reinforces G24)
- [ ] **Carry this into the release/handover documentation** so whoever deploys APIM to a new
      environment knows the grant is a prerequisite —
      → [[APIM IaC - deploying to a new environment (prerequisites)]]

---

## Part 8 — Open questions

> [!question] Should the grant be codified as infrastructure, or applied by hand?
> A Pulumi role assignment would be reproducible and reviewable. But this is a permission on a
> **shared master instance**, not on a customer environment, so it does not sit naturally inside
> a per-spoke deployment — every customer's deploy would be asserting a permission on something
> none of them own. Possibly belongs with the hub, or possibly stays a documented one-off.
> Needs a short conversation with SRE.
>
> **Now tracked for decision as [[D2 - The master documentation site|D2 Part 3]].**

> [!success] Does the customer portal need anything *removed* after the copy? — **answered**
> No. Confirmed during Phase 10 planning: it is a **straight copy** of `vprk2` with nothing
> stripped. The customer's own sign-in settings are preserved by the copy; everything else is
> intended content.

---

## For the Product Owner — the plain version

We copy the customer-facing API documentation site from one master copy onto each customer's own
environment. To do that, the automated deployment needs permission to read that master copy.

The catch we found: the permission that *looks* correct doesn't cover images. Without the right
one, a customer would get a site with all the right words and none of the logos — and the error
wouldn't make it obvious why.

We've confirmed exactly which permission is needed and proved it works. What's left is granting
it to the four deployment accounts and running a quick read-only check against the production one
before we rely on it. Small piece of work, but it has to happen before 26.5 goes wide.

---

## Related

- [[APIM IaC - developer portal content (phase 10)]] — the design this permission serves
- [[APIM IaC - known traps and gotchas]] — G24 (sandbox as production dependency), G25 (partial reads),
  G26 (unsupported v2→v2 migration), **G27 (this trap)**
- [[APIM IaC - portal content capture dry run]] — the read-only capture that proved the access works
- [[APIM Findings Summary]]
