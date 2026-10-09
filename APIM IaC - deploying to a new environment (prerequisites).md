---
title: APIM IaC - deploying to a new environment (prerequisites)
created: 2026-08-11
status: draft - to be finished and published as part of the real ticket
audience: whoever deploys APIM to an environment for the first time
tags: [apim, iac, devo-28368, release, handover, permissions, rbac]
---

# Deploying APIM to a new environment — prerequisites

> [!abstract] What this is
> The **release/handover page**. Everything that must be true *before* APIM is deployed to an
> environment for the first time, and who does each part.
>
> Written because several of these are **not code**. They can't be fixed by re-running the pipeline,
> and each one fails in a way that doesn't point at itself.

> [!warning] Draft
> Captured now so it isn't rediscovered under release pressure. To be finished and published
> alongside the real implementation tickets.

---

## Why this page exists

Almost everything in this work is automated. These are the handful of things that aren't — and every
one of them, if missed, produces a failure that looks like something else:

| If this is missed | What you actually see |
| --- | --- |
| The content-master grant | Deploy fails part-way, complaining about *the customer's* environment |
| Wrong role chosen | Portal appears with all the right words and **no images** |
| Component not registered | Deploy **succeeds and does nothing** — no APIM, no error |
| Sign-in secret expired | Portal sign-in silently stops working — no deploy involved |

None of these say *"you forgot a prerequisite."* Hence a checklist.

---

## 1. The content-master grant 🔴 blocks production today

### What's going on, plainly

Each customer's API documentation site is **copied from one master copy** we maintain
(`vprk2-sbx-spk-apim`, in our own sandbox subscription). To copy it, the account running the
deployment must be allowed to **read** that master.

That account isn't a person — it's one of **four** Azure DevOps service connections, one per tier.

### Where we actually stand

Checked against live Azure on 2026-08-11:

| Service connection | Covers | Access today |
| --- | --- | --- |
| `Dayshape.DevOps.Orchestrator.Internal` | Internal *(incl. Dreich)* | ⚠️ yes — **by accident** |
| `Dayshape.DevOps.Orchestrator.Sandbox` | Sandbox | ⚠️ yes — **by accident** |
| `Dayshape.DevOps.Orchestrator.Lower` | Lower customer envs | ❌ **none** |
| `Dayshape.DevOps.Orchestrator.Upper` | 🔴 **Production** | ❌ **none** |

"By accident" is the important word. **Nobody granted those two access to the master.** They hold
blanket administrator rights over the whole sandbox subscription for unrelated reasons, and the
master happens to live in it.

> [!danger] Why this is worse than a simple missing permission
> - **The two that work are the two we test with.** Our testing passes; production fails on the
>   first real customer, part-way through a deploy.
> - The access relied on is **far broader than needed** — it includes *write*, so a misconfigured
>   cleanup step could erase the master. There's no version history and no backup.
> - It's **incidental**, so it could be tightened for perfectly sound unrelated reasons and silently
>   break every APIM deployment. Nothing records the dependency.

### What to do — a purpose-built role

Not the off-the-shelf `API Management Service Contributor`. That grants write, and it's effectively
what the two working connections already have by inheritance — i.e. the thing we want to stop
depending on.

A dedicated read-only role grants exactly what the copy needs and nothing more:

```jsonc
{
  "Name": "Dayshape APIM Portal Content Reader",
  "Description": "Read-only access to the APIM developer portal content master, so deployments can copy it to customer environments. Deliberately excludes write.",
  "Actions": [
    "Microsoft.ApiManagement/service/read",
    "Microsoft.ApiManagement/service/contentTypes/read",
    "Microsoft.ApiManagement/service/contentTypes/contentItems/read",
    "Microsoft.ApiManagement/service/portalConfigs/read",
    "Microsoft.ApiManagement/service/portalConfigs/listMediaContentSecrets/action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/58d847ff-52c8-4127-b663-1232b8fbbd48"
  ]
}
```

All five confirmed to exist. Assign to **all four** connections, scoped to the **`vprk2` resource
itself** — not the resource group, not the subscription.

> [!note] Why the fifth line matters more than it looks
> Text and styling are ordinary reads. **Images are not** — they come back via an *action* that
> hands over a temporary link. Azure treats an action as separate from a read, so read-only roles
> don't include it.
>
> This is why the obvious candidate, `API Management Developer Portal Content Editor`, **does not
> work** despite being named after exactly this job. Anyone reasoning from role names alone picks
> it, and gets a portal with no logos. → G27

**Owner:** SRE, with Ecosystem
**Open question:** applied by hand or defined as code? → [[D2 - The master documentation site|D2 Part 3]]

### How to actually apply it

Two steps, needing **two different rights**. Worth knowing, because the first is a one-off and the
second is the bit that recurs.

> [!warning] Step 1 needs `Owner` or `User Access Administrator`
> `Contributor` **cannot** do it — it explicitly subtracts `Microsoft.Authorization/*/Write`. Nor can
> `Role Based Access Control Administrator`, despite the name: that covers role *assignments* only.

**Step 1 — define the role. Once ever.**

Save the JSON above as `apim-portal-content-reader.json`, then:

```bash
az role definition create --role-definition @apim-portal-content-reader.json
```

The `AssignableScopes` line in the JSON is what limits where it can be used — as written, anywhere
within the sandbox subscription. That is deliberately broader than the resource itself, so the role
survives the master ever being rebuilt or moved within that subscription; the *assignment* in step 2
is what pins it to one resource.

Record the `name` field it returns (a GUID) — that is the role's permanent ID. It goes in
`Products/Modules/AzureRoleDefinitions.py` alongside the other seven Dayshape custom roles:

```python
# ── Dayshape custom roles (tenant-specific, not Azure built-ins) ──────────
DAYSHAPE_APIM_PORTAL_CONTENT_READER = "<guid>"
```

**Step 2 — assign it. Once per service connection, and again for any new one.**

Needs `Role Based Access Control Administrator` — which Ecosystem already holds on Sandbox.

```bash
SUB=58d847ff-52c8-4127-b663-1232b8fbbd48
RG=vprk2-sbx-spk-apim-rg
SVC=vprk2-sbx-spk-apim

az role assignment create \
  --assignee-object-id "<service principal object id>" \
  --assignee-principal-type ServicePrincipal \
  --role "Dayshape APIM Portal Content Reader" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$SVC"
```

> [!important] Scope it to the **service**, not the resource group or subscription
> The `--scope` above ends at the APIM service itself. Assigning at resource-group or subscription
> level would hand out these rights on everything in it, which defeats the point of a narrow role.

Then run the verification below as that identity. Allow a minute or so — Azure takes time to
propagate a new assignment, and an immediate check can fail misleadingly.

> [!note] If this is ever moved into code
> The house pattern is custom role → **security group** → scoped to the single resource, as in
> `Modules/AppCfg/infra/app_cfg.py`. Note that **only the assignment** would move; nothing in
> `Products/` creates a role definition, and all seven existing custom roles are defined out-of-band
> and referenced by GUID. → [[D2 - The master documentation site|D2 Part 3]]

### Verify it, don't assume it

Run this per connection **before** relying on it. It reads and changes nothing:

```bash
SUB=58d847ff-52c8-4127-b663-1232b8fbbd48
RG=vprk2-sbx-spk-apim-rg
SVC=vprk2-sbx-spk-apim
az rest --method post \
  --url "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$SVC/portalconfigs/default/listMediaContentSecrets?api-version=2021-08-01" \
  --query containerSasUrl -o tsv > /dev/null \
  && echo "PASS" || { echo "FAIL - missing listMediaContentSecrets on vprk2"; exit 1; }
```

A `403` here is precisely the failure a real deployment would hit — caught in seconds rather than
half-way through a customer deploy.

---

## 2. The environment must be told APIM exists 🔴 silent failure

The deployment does **not** work from the components in the code. It works from a list held in the
**environment registry**: what this environment is expected to have.

If APIM has not been registered against the environment, **nothing deploys, nothing errors, and the
pipeline reports success.**

> [!danger] This one lies to you
> Every other item on this page fails visibly. This one produces a clean green run that did nothing.
> The obvious reading is *"the deploy worked but APIM did not appear"*, which sends you looking in
> entirely the wrong place.

Registration is not automatic. New environments are seeded with a fixed starter set that does not
include APIM. There are four routes to add it — SRE inserting the row directly, relaxing the
dev-only guard on the `add-target-infra` CLI, adding APIM to the starter set and re-running
**ConfigSync**, or a local `orchestrate-dev` run (which writes to the *dev* registry only and does
not help a real environment).

Which route we take is a decision, not a detail — see G31.

**So adding a component to an environment is not purely a code change**, and the step that makes it
real has no tooling the team can run. Applies to **every** environment's first APIM deploy.

**Owner:** SRE · → G31

---

## 3. The master instance must be protected

`vprk2-sbx-spk-apim` stops being a sandbox toy the moment this ships. **Every customer deployment
reads from it.**

- [ ] Named owner recorded
- [ ] Marked do-not-delete, exempt from sandbox cost cleanups
- [ ] Understood: no version history, and APIM's own backup **does not cover portal content**

If it's deleted or paused, **every** APIM deployment stops working. → G24, G26

**Owner:** SRE

---

## 4. The portal sign-in secret — ✅ **no longer a prerequisite**

**Resolved 2026-08-13.** The developer portal sign-in relies on a credential that expires. That used
to mean someone had to create it before the first deploy and diary its renewal.

**The deployment now creates it and replaces it every 30 days**, so there is nothing to arrange
before standing up a new environment and nobody to name as its owner.

- [x] ~~Reaches production by an agreed route~~ → the deploy creates it. See
      [[D1 - The developer portal sign-in secret]]
- [x] ~~Named owner for renewal, with expiry diarised~~ → renews itself

> [!warning] One thing to be aware of, not to action
> Renewal happens **during an APIM deployment**. An environment left undeployed for more than twelve
> months will still drift. Customers get 5–6 deploys a year, so this is only a concern for
> long-dormant internal environments.

**Owner:** nobody — that is the point.

---

## Pre-flight checklist

Before the first APIM deploy to any environment:

- [ ] Custom role defined and assigned to **all four** connections, scoped to `vprk2`
- [ ] Read-only check passes for the connection this environment uses
- [ ] `ApimSpike` registered as a component for the target environment
- [ ] `vprk2` has a named owner and do-not-delete marking
- [ ] Sign-in secret in place, with renewal owned

---

## Related

- [[APIM IaC - portal content permissions (phase 10)]] — the full detail behind §1
- [[APIM IaC - known traps and gotchas]] — G24, G26, G27, G30, G31
- [[D2 - The master documentation site]]
- [[APIM IaC - how to run a Dreich test deploy]]
