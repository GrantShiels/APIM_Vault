---
tags: [apim, iac, devo-28368, needs-decision, permissions, rbac, ownership]
decision-id: D2
absorbs: [D2, D3, D4]
updated: 2026-08-13
status: needs-decision
sign-off: SRE (Richard Donkin) · Security · Product Owner · Tech Lead
blocks: first real customer deploy
---

> [!warning] Audited 2026-10-07
> Moving the master site is now ticket DEVO-31423 (split from DEVO-30004). Ownership and permission questions below are still open.

# D2 — The master documentation site

> [!abstract] One instance, three questions — all for the same conversation
> Every customer's API documentation site is copied from **one master instance**. Three decisions
> hang off that single fact, and they are best taken together because they are the same
> conversation with the same people.
>
> | | Question | Sign-off |
> | --- | --- | --- |
> | **Part 1** | **Who owns it**, and how do we stop it being deleted? | PO · TL · SRE |
> | **Part 2** | **What permission** does the deployment get to read it? | SRE · Security |
> | **Part 3** | **Where does that grant live** — in code, or applied by hand? | SRE |
>
> *This page previously existed as three documents — D2, D3 and D4. They were combined on
> 2026-08-13. Part 1 is the one the Product Owner needs; Parts 2 and 3 are for SRE and Security.*

---

## Background — what the master is, and why it's a dependency

Our technical authors design the API documentation — pages, styling, logos — in a browser, on a
single API Management instance called **`vprk2`**, in the **Dayshape Sandbox** subscription. Every
customer's site is a copy of it.

**This is deliberate, not an accident.** The portal is edited through a drag-and-drop visual editor,
so there is no file format to author in — the instance *is* the design tool. Voodoo Park confirmed
the intended workflow in June 2026: any styling or static content change is made on `vprk2` and then copied out with the content scripts.

**It is in active use by Dayshape staff.** Our technical authors were given edit rights in July 2026
specifically so they could write customer-facing documentation there. So this is not a one-off
bootstrap that copies a fixed design — it is the **publishing pipeline for customer documentation**,
and the content behind it changes whenever the authors change it.

> [!info] We checked whether this could be avoided entirely
> We looked hard at whether the documentation could be held in our own source control and  deployed like code. **It cannot** — Microsoft holds the images in storage we don't own and can't reach except through a temporary link. *(Written up separately at
> [[APIM IaC - portal content, why there is no IaC-native route]].)*
>
> So the dependency is real and unavoidable. That is why Part 1 exists — **ownership is the
> mitigation**, not avoidance.

---

# Part 1 — Who owns the master? ⬜

> [!question] What we need decided
> The master **currently has no owner** and lives in a sandbox subscription. **Who owns it, and how do we make sure it isn't deleted?**

## Why this matters

**It was set up as a sandbox.** Sandboxes get cleaned up. They get swept in cost-reduction
exercises, paused when someone is trimming spend, or deleted when someone can't identify what
they're for.

> [!danger] The day this ships, that sandbox becomes production
> Not "production-like" — actually production. Every customer deployment reads from it. If it
> disappears:
>
> - **No customer can be deployed or updated** until it's rebuilt
> - The content **cannot be recovered** — there is no version history, and Azure's own APIM backup
>   does not cover portal content
> - Rebuilding means our technical authors redesigning every page by hand in a browser
>
> There is currently **nothing** stopping this happening. No owner, no tag, no lock. The name gives
> no clue on its own — when Jonathan asked about editor access in July, even Voodoo Park's own
> engineer had to go and look it up.

Two further consequences that nobody has had to think about while the copy step was manual:

- **A sandbox becomes a shared production dependency.** If it is deleted, renamed, moved or simply
  unreachable, every customer deploy that includes portal content fails.
- **There is no record of what shipped.** The desired state is whatever is live on that instance at
  the moment of the deploy. Two customers deployed a week apart can receive different portals with
  no way to tell after the fact, and no way to put a customer back to a previous version.

*(Recorded as **G24**.)*

## The options

### Option A — a named person owns it
**For:** clearest accountability · easy to arrange today
**Against:** people move teams and leave · one person can't cover holidays or out-of-hours

### Option B — a named team owns it ⭐
Most likely SRE, who already own the APIM work after handover.
**For:** survives individuals leaving · fits the existing handover plan · SRE already cover
out-of-hours
**Against:** "owned by a team" can mean owned by nobody unless it's written down properly

### Option C — move it out of the sandbox subscription entirely
Treat it as production infrastructure and put it where production things live.
**For:** removes the root cause rather than mitigating it · gets it inside normal production
protections automatically
**Against:** the most work · needs a rebuild or migration of the instance · **would need doing
before launch**, so it's on the critical path

### ⭐ Option D — snapshot the content into a build artifact
*Added 2026-08-13, prompted by a question from Andy L (TL, Infra Platform).*

Capture the master once when a release is cut, publish the snapshot as a build artifact, and have
each customer deploy read **the artifact** rather than the live instance.

**For:** customer deploys never touch Sandbox at all, so the cross-subscription grant in Part 2 is
only needed on **one build identity, not on production** · gives a **versioned record of exactly
what each customer received**, which fixes the "no record of what shipped" problem above · the
sandbox stops being a production dependency, which substantially defuses Part 1
**Against:** an extra publish step to build and maintain · content only moves at release cadence
rather than continuously, which changes the authors' feedback loop

> [!note] This is not a novel pattern here
> We have already prototyped this exact shape for the OpenAPI specification, modelled on
> `Dayshape.LanguagePack.Publish.ps1`, which is existing production code. The prototype is PoC
> quality, but the pattern is established.

## Our view

**Option B, plus the cheap protections regardless of which option is chosen.** Whatever else is
decided, these cost almost nothing and should happen anyway:

- A **resource lock** so it can't be deleted accidentally
- **Tags** saying what it is and that it must not be removed
- A line in the runbook explaining what breaks if it goes

Option C is the genuinely correct answer and we'd support it — but it's real work on the critical
path, and Option B plus a lock removes most of the risk for a fraction of the effort.

**Option D is worth a serious look** because it is the only option that addresses the *shape* of the
problem rather than mitigating its consequences — and it makes Parts 2 and 3 dramatically simpler by
removing production's need for any access at all.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

# Part 2 — What permission does the deployment get? ⬜

> [!question] What we need decided
> The automated deployment needs permission to read the master. **Do we grant a broad off-the-shelf permission, or a narrow custom one — and who approves it?**

## Why this matters

Previously a person did the copy while logged in, so permissions never came up. Now it runs
automatically, as a service account, which only has the access we explicitly give it.

> [!danger] The failure mode is genuinely nasty
> The copy fetches **words** and **images** by two different mechanisms. A read-only permission
> covers the words but **not the images**.
>
> So with the wrong permission the copy *half succeeds*: the customer gets a documentation site with
> every page correct and **no logos at all**. The error message points at the customer's deployment
> rather than the master it couldn't read.
>
> **This is the shape of problem a customer finds before we do.**

### The part that catches people out

We checked every plausible permission against live Azure. The one **named after the exact job we are doing** — `API Management Developer Portal Content Editor` — **does not work.**

| Permission                                       | Covers images?                             |
| ------------------------------------------------ | ------------------------------------------ |
| `Reader`                                         | ❌                                          |
| `API Management Service Reader Role`             | ❌                                          |
| `API Management Developer Portal Content Editor` | ❌ **← the obvious choice, and it's wrong** |
| `API Management Service Operator Role`           | ❌                                          |
| `API Management Service Contributor`             | ✅                                          |
| `Contributor`                                    | ✅                                          |

Anyone reasoning from names alone — including a future us — picks the wrong one. *(Recorded as
**G27**.)*

## Good news first

The master lives in **our own Dayshape Sandbox subscription**. It is not a Voodoo Park tenant and
there is no third party involved. **Granting this is entirely within Dayshape's control** — no
negotiation, nothing to wait for.

> [!success] ⭐ Verified 2026-08-13 — customer environments are in the *same tenant*
> Prompted by Andy L asking how permissions could possibly work for a customer environment
> "completely unrelated to the sandbox".
>
> Every subscription we have — every customer Lower and Upper/Prod, the regional Hubs, Platform, and Dayshape Sandbox — sits in **one Entra tenant** (`201e1d9f-…`). There is exactly one distinct tenant ID across the whole estate.
>
> This matters a great deal:
> - **Different tenants** would mean no shared identity and no shared access control. You'd need
>   cross-tenant trust or two sets of credentials. Genuinely hard.
> - **Different subscriptions in one tenant** means a single service account can simply be *granted*
>   access to both. It is an ordinary permission grant.
>
> **So the copy process works across subscriptions without anything special** — provided the grant below is actually made. The script switches subscription mid-run, which is fine inside one tenant.

## The options

### Option A — the off-the-shelf role (`API Management Service Contributor`)
One command, standard role, works immediately.

**For:** fastest · no custom role to maintain · well understood
**Against:** it grants **write** access to our content master

> [!caution] Why the write access is a real concern
> The copy tool has a cleanup step whose job is to **wipe** whichever sites are listed as targets.
> If that list were ever built incorrectly and named the master, an identity with write access would be *permitted* to erase it.
>
> The master has **no version history and no backup**. Losing it would be a genuine incident.

### Option B — a custom permission with exactly five rights ⭐
Grants precisely what the copy needs and nothing more. All five confirmed to exist in Azure.

**For:** the cleanup step becomes **incapable** of touching the master, however wrong the config ·
turns an unlikely disaster into an impossible one · the extra effort is one role definition
**Against:** a custom role is a thing we now own and maintain · marginally more setup

## What the five permissions actually do

The role is written out ready to apply in
[[APIM IaC - deploying to a new environment (prerequisites)]]. What follows is what each line is
*for*, so the grant can be reviewed on its merits rather than taken on trust.

| Permission | What it allows | Why the copy needs it |
| --- | --- | --- |
| `service/read` | See that the master exists | Without it the instance is invisible and nothing inside it can be addressed |
| `contentTypes/read` | List the *kinds* of content the site holds | The tool asks what exists rather than assuming, so nothing is missed if the portal gains a new kind |
| `contentTypes/contentItems/read` | Read the pages, text, menus and layouts | This is the documentation itself |
| `portalConfigs/read` | Read the site's own settings | Part of what makes the published copy match the master |
| `portalConfigs/listMediaContentSecrets/action` | Ask for a temporary link to the uploaded files | **The images.** See below — this is the one that catches people out |

> [!important] The whole list is read-only, by construction
> There is no `write` and no `delete` anywhere in it. Azure permissions are an allow-list: anything
> not named is refused.
>
> This is the direct answer to *"could the cleanup step ever wipe the master?"* Under this role the
> wipe isn't *refused* — it is **unrepresentable**. That is a stronger guarantee than a safety check
> in our own code, because it holds even if our code is wrong.

The fifth line is why the obviously-named role doesn't work. Text comes back from an ordinary read.
**Images do not** — the deployment has to *ask* for a short-lived link to the storage they live in,
and Azure classes that as an *action*, not a read. Roles built around "read" therefore miss it.

*Re-verified against Azure's own operations list on 2026-08-12 — all five exist and are spelled as above.*

## ⚠️ This is four permission changes, not one

Deployments run through **four** separate Azure DevOps service connections.

> [!danger] Verified against live Azure, 2026-08-11 — and the result is worse than expected
> | Connection | Covers | Access to the master | How |
> | --- | --- | --- | --- |
> | `…Orchestrator.Internal` | Internal *(incl. Dreich)* | ✅ | `Contributor`, **inherited from the whole subscription** |
> | `…Orchestrator.Sandbox` | Sandbox | ✅ | `Contributor` + `Owner`, inherited |
> | `…Orchestrator.Lower` | Lower customer envs | ❌ **none** | — |
> | `…Orchestrator.Upper` | 🔴 **Production** | ❌ **none** | — |
>
> **There is not a single role assignment scoped to the content master itself.** Every access above
> is incidental — a side effect of broad subscription-level rights that exist for other reasons.

### Why this is the exact trap this document was written about

The two connections that work are the two we'd test with. **Our testing on Dreich will pass, and
production would fail on the first real customer.**

It is also worse than a plain missing permission:

- The access we *do* have is **`Contributor` across the entire subscription** — far broader than the
  five rights the copy needs, and it includes the write access that makes the "cleanup wipes the
  master" scenario possible.
- It is **incidental**, so it could be tightened for unrelated good reasons and silently break every
  customer deployment. Nothing records that APIM depends on it.

> [!important] This strengthens the case for Option B
> Option A said "grant `API Management Service Contributor`". But `Internal` and `Sandbox`
> effectively already have that by inheritance — and it's precisely what we'd want to *remove*.
>
> **The tidiest outcome is a custom role explicitly assigned to all four**, which fixes the two that
> are missing and makes the two that work no longer dependent on inherited Contributor.

> [!note] Option D in Part 1 would change this picture substantially
> If customer deploys read a published artifact rather than the live master, then **production never
> needs access at all** and this becomes a single grant on one build identity.

## How we'd prove it rather than hope

**Already partly done.** The four service principals were resolved in Entra and checked against the
master's role assignments directly — that's where the table above comes from. What it can't tell us
is whether inherited `Contributor` genuinely satisfies the image call in practice.

A one-off pipeline job per connection settles that. It calls the image permission and nothing else,
so it is **read-only and changes nothing**, and turns a mid-deploy customer-facing failure into a
five-second pass/fail.

We have already confirmed the approach works using an ordinary Dayshape login — 9 content types
returned, valid image link returned.

> [!tip] Run it against `Upper` before anything else
> `Upper` is production and currently has **no access at all**. It is the one that matters, and the
> check costs seconds.

## Our view

**Option B.** Not for tidiness — because the thing being protected has no backup and no undo, and
the protection costs us a single role definition.

Under Option A the safeguard is "our configuration is always correct". Under Option B the safeguard
is "the permission does not exist". The second is worth the extra hour.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

# Part 3 — Where does the grant live? ⬜

> [!question] What we need decided
> Once Part 2 decides *what* to grant, this decides *how it is applied and kept*. Written into our
> infrastructure code, or applied once by a person and written down?

## Why this is not obvious

Everything else about APIM is being moved out of a manual runbook and into code — so the instinct is
to put this permission in code too. But this one doesn't fit the shape of the rest.

Our infrastructure code is organised **per customer environment**. This permission is a grant on a
**shared master instance that no customer owns**. Expressed as normal per-customer code, **every
customer's deployment would assert a permission on a resource that belongs to none of them.**

## ⭐ Verified 2026-08-11 — the obvious option is impossible, not merely awkward

Three independent findings, any one of which rules out putting this in per-customer code:

**1. Two of the four identities have no rights in that subscription at all.**
`Lower` and `Upper` — **`Upper` is production** — hold nothing on the sandbox subscription. A
customer deployment running as `Upper` **cannot** create a permission there. It cannot see the
resource.

**2. The one that does have rights still cannot grant permissions.**
`Internal` holds `Contributor`, which explicitly **excludes** permission management
(`notActions: Microsoft.Authorization/*/Write`). That's deliberate on Microsoft's part — an identity
able to grant itself permissions is effectively unlimited. Only `Sandbox` could do it, and `Sandbox`
never deploys a customer.

**3. It is one shared resource, and there are many customers.**
Every customer's deployment would try to manage the *same* permission on the *same* resource, each
stack believing it owns it. They would fight, and unpicking that is worse than the original problem.

> [!note] Also worth knowing — permissions aren't instant
> Azure takes time to propagate a new permission. The codebase already knows this: `ContainerAppsEnv`
> grants access then **deliberately waits 60 seconds**. So even a self-granting deploy would race
> itself — sometimes passing, sometimes not. That is a worse failure than a clean one.

**This is not "IaC vs not IaC". It is *whose* IaC.**

## ⭐ Verified 2026-08-12 — the question splits cleanly in two

*Found by trying to create the role and being refused.*

Defining a role and granting it need **different Azure rights**:

| Step | Happens | Azure right needed | Can our pipelines do it? |
| --- | --- | --- | --- |
| **Define** the role | **Once, ever** | `roleDefinitions/write` — `Owner` / `User Access Administrator` | **No.** No deployment identity holds it, and giving one it would be a very broad grant |
| **Assign** it to a connection | Once per connection | `roleAssignments/write` — `Role Based Access Control Administrator` | **Possibly** — this is what the live question is really about |

### Our own codebase has already answered the first half

We own **seven** custom Dayshape roles. Not one of them is *created* in `Products/` — there is no
role definition anywhere in the repository. Each is recorded as a bare identifier in
`Modules/AzureRoleDefinitions.py` and then *assigned* in code.

> [!success] So the existing house pattern is: **definitions by hand, assignments in code.**
> That isn't a new precedent we'd be setting — it's the one we already follow everywhere else.
> Defining a role is a one-off act; asserting it on every deployment would be repeating a thing that
> cannot change.
>
> **The definition is out-of-band whichever option is picked. The only live question is where the
> *assignment* lives.**

### The house pattern for the assignment itself

Where we do assign custom roles, the shape is: custom role → **security group** → scoped to one
specific resource *(e.g. `Modules/AppCfg`)*. Worth following rather than assigning to each service
principal individually.

## ⚠️ Creating the role needs a right nobody on the team has

On the Sandbox subscription, Ecosystem holds `Contributor` and `Role Based Access Control
Administrator`. **Neither covers defining a role:**

- **`Contributor`** looks like it should, since its actions are `*` — but it explicitly subtracts
  `Microsoft.Authorization/*/Write`.
- **`Role Based Access Control Administrator`** is named as though it covers all of RBAC, but its
  actions are `roleAssignments/write` and `roleAssignments/delete` only. **Assignments, not
  definitions.**

> [!tip] The useful half of this finding
> **We can already do the assignments ourselves.** Only the definition needs someone else — and it
> happens *once ever*, not once per environment or per customer.
>
> So the ask is small and bounded: either grant Ecosystem `User Access Administrator` on Sandbox, or
> have SRE run the one create command and hand back the role's ID.

## The remaining options

### Option A — per-customer infrastructure code
> [!failure] Ruled out by the evidence above — retained so the reasoning is visible
> Production has no rights in that subscription, `Contributor` cannot grant permissions at all, and
> many customer stacks would contend over one shared resource.
>
> There is also a hidden cost: to let a deployment grant this permission, the deployment must itself
> be allowed to manage permissions — **we'd be widening access in order to narrow access.**

### Option B — apply it once by hand, documented
**For:** simple · no elevated rights needed anywhere · matches the fact that it genuinely is a
one-off on a shared resource
**Against:** manual, so it can be forgotten or silently removed · not visible in code review · a new
service connection would need someone to remember

### Option C — put it with the shared/hub infrastructure ⭐
Not in per-customer code, but wherever we already manage things shared across all customers.
**For:** codified **and** asserted once rather than per customer · no elevated rights in customer
deployments · the right conceptual home
**Against:** **we don't yet know if a suitable place exists** — that's an SRE question · likely the
most upfront work

## Our view

**Option C if a suitable home exists, otherwise Option B.**

We genuinely don't know whether Option C is available; that is why this is a question for SRE rather
than a decision we're presenting.

**Whichever is chosen, the deployment should check the permission works before it starts copying** —
so a missing grant produces *"cannot read the content master — check access"* rather than a
half-built customer site. That part isn't a decision; we'll build it either way.

*A suggestion for discussion, not a recommendation to rubber-stamp.*

---

## What we need from you

### Part 1 — ownership *(Product Owner · TL · SRE)*
- [ ] **Who owns it** — a person, a team, does it move, or do we snapshot it (Option D)?
- [ ] Can we apply a **delete lock** now, regardless of the wider decision? *(cheap, reversible, no downside)*
- [ ] Does this belong on the sandbox cost-cleanup exclusion list, and who maintains that?
- [ ] Is anyone already responsible for it that we haven't found?

### Part 2 — the permission *(SRE · Security)*
- [ ] **Option A or Option B?**
- [ ] **Who creates the role definition?** We can't — it needs `Owner` or `User Access
      Administrator` on Sandbox. Either grant Ecosystem that, or run the one create command for us
      and send back the role's ID. *(One-off, not per environment.)*
- [ ] Who approves a permission change on the Sandbox subscription?
- [ ] Can we run the read-only verification job against **production** (`Upper`) before we rely on it?

### Part 3 — where the grant lives *(SRE)*
- [ ] **Where does the *assignment* live — B or C?** *(The definition is out-of-band either way.)*
- [ ] If C — is there an existing place for shared/hub-level permissions we should use?
- [ ] If B — who applies it, and where is it recorded so it isn't lost?
- [ ] How would we notice if this permission were removed six months from now?

## Sign-off

**Part 1:** Product Owner · Tech Lead · SRE — *the PO matters because the failure mode is a
customer-visible outage with a slow recovery, not a technical inconvenience.*
**Parts 2 and 3:** SRE (Richard Donkin) · Security · IaC reviewer

> [!note] This must outlive the decision itself
> Whatever is chosen has to land in the **release documentation** — the grant is a prerequisite for deploying APIM to *any* new environment, and it is not something the pipeline can fix by itself.
> Drafted at [[APIM IaC - deploying to a new environment (prerequisites)]], with the custom role
> written out ready to apply.

## Decision

*To be completed.*

---

## Related

- [[APIM IaC - portal content permissions (phase 10)]] — the full verification, with commands
- [[APIM IaC - portal content, why there is no IaC-native route]] — why we can't avoid this dependency
- [[D7 - Accepting an unsupported content migration]] — the same content, a different risk to it
- [[APIM IaC - known traps and gotchas]] — **G24** (sandbox as production dependency), **G25**
  (partial reads delete pages), **G27** (the role-name trap), **G30**
