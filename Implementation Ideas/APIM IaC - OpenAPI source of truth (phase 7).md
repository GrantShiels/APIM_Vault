---
tags:
  - APIM
  - IaC
  - decision
  - design
  - phase7
ticket: DEVO-28370
related: DEVO-28368
status: superseded by the code — built and merged (commit 656f350)
date: 2026-08-06
updated: 2026-08-07
---

> [!warning] Audited 2026-10-07 — paths in this note are wrong
> Real source on `iac-main`: blob `openapi/<CORE_SOFTWARE_VERSION>/openapi.v1.json` in `dsartifactrepo` (resolver `hooks/apim_openapi_resolver.py`). DEVO-31392 moves to `entities/openapi.v1|v2.json`. DEVO-28370's `Infrastructure/openapi.v1.json` path is also outdated.

# APIM IaC — the OpenAPI file: which one, which version, and how we keep it current

> [!warning] Superseded by the code
> Phase 7 is **built and merged** (commit `656f350`). This document previously said "ready to implement"; it has been implemented.
>
> **Where this document and the code disagree, the code is right.** Kept for the reasoning behind
> the decisions, not as a description of how things behave today.

> [!abstract] The whole thing in four sentences
> APIM doesn't know what our API looks like. We have to hand it a file — the **OpenAPI
> spec** — that lists every endpoint. Right now there are **two copies of that file in the
> repo, they've drifted apart, and the setup guide points at the wrong one**. On top of
> that, nobody has a way of knowing whether the file being handed to APIM actually matches
> the version of Dayshape that's running — which is the thing Sam Clifford asked us to
> prioritise.

Parts 1–6 explain the problem from scratch (written so it can be handed to a PO).
**Part 7 onwards is the proposed plan** — that's the part to review.

---

> [!success] ✅ Reviewed and endorsed — 2026-08-07
> Run past Sam Clifford, Richard Donkin and Marcus Thomson on 2026-08-07.
> **Sam endorsed the design outright** — *"Yeah, yeah, absolutely."*
>
> **The three questions this doc was blocked on are answered:**
>
> | Question | Answer |
> | --- | --- |
> | How often does the orchestrator run? | On **every customer version upgrade** — the Core deploy pipeline runs it. Also manually via the `IaC Deploy` pipeline. It reads a GSOT table of enabled components and does a **diff**, not a rebuild. |
> | Any objection to a publish step? | **No.** Sam wrote the language pack publish and confirmed it's the model to copy. |
> | Source of truth? | **Confirmed** — `Infrastructure/openapi.v1.json`, the one *not* in the APIM spoke folder. |
>
> **He framed the value more broadly than I did** — see §7.0 below. Publishing the spec as an
> artifact is a step towards **decoupling infrastructure deploys from Core deploys** (blue/green,
> deploying services in isolation), not just keeping a file in sync.
>
> **Two things this call added that aren't yet solved:** the deployment-window version skew
> (§7.6) and whether the publish belongs in its own CI pipeline (open question 2).
>
> Full notes: [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]]

---

## Part 1 — What the OpenAPI file actually is

Think of APIM as a receptionist sitting in front of Dayshape.

The receptionist doesn't know anything about Dayshape on their own. To do their job they need
a **directory** — a list of every service a customer can ask for, what information each one
needs, and what they'll get back.

The OpenAPI file *is* that directory. It's a single (very large) JSON file listing all 64
endpoints of our Proxy API.

Two things use it:

1. **APIM itself** — it reads the file and creates a matching operation for each endpoint, so
   it knows what to route where.
2. **The developer portal** — the "getting started" site customers log into. Every page of
   documentation they see is generated from this file.

So if the file is wrong or out of date, customers see the wrong documentation and APIM may
route calls incorrectly.

> [!info] Where the file comes from
> It's **generated** from our C# code by a tool called NSwag, which reads all the controllers and
> writes out the JSON. Nobody edits it by hand.
>
> The important consequence: **it's a build output, not source code.** Like a compiled DLL.

---

## Part 2 — Problem one: there are two copies, and they've drifted

The setup guide contradicts itself about where this file lives.

| Where in the guide | Which file it names |
| --- | --- |
| The variables table near the top | `Infrastructure/openapi.v1.json` |
| **Phase 7 — the bit we're implementing** | **`Products/Spoke/APIM/openapi_docs/entities.json`** |
| Section 12 (post-release updates) | `Infrastructure/openapi.v1.json` |

Two out of three say one thing. Phase 7 — unhelpfully, the one we need — says another.

Both files really do exist on `develop`. I diffed them:

| | |
| --- | --- |
| `Infrastructure/openapi.v1.json` | last regenerated **7 July** |
| `Products/Spoke/APIM/openapi_docs/entities.json` | last updated **29 June** |
| Lines that differ | **92** |

So `entities.json` is about five weeks behind. It missed a change where a field's description
gained the text *"(default value: FALSE)"*. Small in itself — but it proves the copy silently
falls behind, and nothing catches it.

### The bit that matters most

Of those 92 differing lines, **only 2 are a genuine difference**. The other **90 are noise**.

Here's why. The generator bakes example dates into the file, and it stamps them with the
**timezone of whoever's laptop generated it**. One file was generated in the UK (`+01:00`),
the other somewhere three hours ahead (`+03:00`).

```
-  "example": "2024-01-15T09:30:00+01:00"
+  "example": "2024-01-15T09:30:00+03:00"
```

Ninety lines of that. Zero meaning.

> [!warning] Why this matters more than it looks
> DEVO-28370 asks for *"a repeatable validation step"* to catch drift. The obvious
> implementation — compare the files, or compare checksums — **would fire on almost every
> regeneration and tell us nothing.** Ninety false alarms, two real ones.
>
> Any drift check has to ignore the noise, or the noise has to be fixed first. **The plan in
> Part 7 fixes it**, as a side-effect.

This is a known bug. Laszlo at Voodoo Park acknowledged it on 17 July and said a fix was
coming in PR #6303. **It hasn't landed** — the current file still has 45 of these.

### The easy part

`entities.json` is referenced by exactly two files, both inside `Products/Spoke/APIM/` — the
old APIM component **we already deleted from our branch**.

So on our branch `entities.json` is already dead. We don't have to migrate off it. We just
never adopt it.

And the source-of-truth question is **already answered** — we just have to follow the answer:

- **ADR-002** (status: Accepted) says output goes to `Infrastructure/openapi.v1.json`
- `nswag.v1.json` literally has `"output": "../../Infrastructure/openapi.v1.json"`

> [!success] Decision one — not really a decision
> **`Infrastructure/openapi.v1.json` is the source of truth.** ADR-002 already says so.
> The setup guide's Phase 7 is simply wrong, and it points at a folder that no longer exists
> on our branch. This is a guide correction, not a choice.

---

## Part 3 — Problem two: re-importing wipes the security policy

The guide admits this itself in section 7D.

The sequence is:

1. **Phase 7A/7B** — import the spec, APIM creates all the operations
2. **Phase 7C** — apply a policy to the API (this is where the HMAC signing from Phase 5
   finally gets switched on — the thing that stops people bypassing APIM and hitting Dayshape
   directly)
3. **Later — a new Dayshape release ships.** Someone re-imports the updated spec…
4. …**and step 2 is wiped.** The security policy is gone until a human remembers to redo it.

That's the "drift/regression risk" phrase in DEVO-28370. It's real.

> [!danger] And it has never actually been done
> Marcus Thomson — the SRE who performs the manual APIM setups — said on 23 July:
>
> > *"I don't know why that is there and we haven't used it in the past. **We also haven't
> > actually gone through a proper update to the OpenAPI doc after a new release either.**"*
>
> Nobody has ever performed the post-release spec update. Phase 7D and section 12 are
> **untested procedure**. That's the single biggest risk in this ticket.

### The good news

This one mostly solves itself once it's in Pulumi — see §7.5.3.

---

## Part 4 — Problem three: which *version* of the file? (the hard one)

This is the part Sam Clifford flagged in #infra-platform on 20 July:

> *"Open API spec selection was baked into the IaC — **this is the piece I'd prioritise**: how do
> we deploy the open API spec software onto provisioned IaC? IaC can be provisioned as part of a
> different lifecycle to software, so we need to make sure the spec that is used aligns with the
> core version that has been deployed."*

In plain English: **infrastructure and the application are deployed on different schedules.**

A customer might be running Dayshape **26.4**, while our infrastructure code has moved on to
**26.5**. If the IaC just grabs whatever spec file is sitting in the repo at that moment, APIM
gets handed a directory describing 26.5 — endpoints that don't exist on the customer's site
yet. The receptionist advertises services the building can't provide.

Today the IaC does exactly that. It reads a file from the repo with no version awareness at all.

### There's a proven pattern for this already

`Products/Spoke/LanguagePack` solves the identical problem. Language packs have the same
constraint: the translation files must match the running Dayshape version.

| Question | How LanguagePack answers it |
| --- | --- |
| How does it know the Core version? | An environment variable `CORE_SOFTWARE_VERSION`, set by the release pipeline |
| Where do versioned copies live? | Blob storage — `dsartifactrepo` → `release/languagepacks/v26.5.0.0/en-gb.zip` |
| Release vs pre-release? | Version matching `vNN.N.N.N` goes to `release`, anything else `releasecandidate` |
| What about running locally? | Falls back to the file in the repo — no cloud access needed |
| What if the version is missing? | Fails loudly rather than guessing |

There's also shared helper code — `get_dayshape_version()` in `Products/Modules/helper_functions.py` —
that already resolves the version.

> [!tip] What this means for us
> We wouldn't be inventing anything. We'd be copying a working, reviewed, in-house pattern —
> which is exactly what our working rules say to do.

### The catch — and the thing that makes it much smaller than it looks

**Nobody publishes the OpenAPI spec as a versioned artifact.** There is no per-version copy to
fetch. The shelf exists; nobody has put the file on it.

> [!note] Correction to an earlier version of this note
> I originally wrote that the spec "is never generated in a pipeline — only on a developer's
> laptop". **That is wrong, and the truth is much better news.**

The generation is wired into the build itself, not into any pipeline YAML — which is why a
search of the pipeline files finds nothing:

```
Dayshape.csproj:701
  <Target Name="GenerateOpenApi" AfterTargets="Build"
          Condition="'$(Configuration)' == 'Release' Or '$(GenerateOpenApiOnBuild)' == 'true'">
      → runs NSwag → writes Infrastructure/openapi.v1.json

Dayshape.Core.Build.yml:2-5     BuildConfiguration defaults to "Release"
Dayshape.Core.Release.yml:174   calls it without overriding
```

So **every release build already regenerates the spec on the build agent**, as an automatic
side-effect of compiling in Release. And look where the language pack publish sits:

```
174   Build Core            ← a fresh, correct spec is written here
179   Publish language pack
        ↑ that spec is sitting on disk, in the same job, at this exact moment
```

> [!success] What this means
> We are **not adding generation**. We are adding an **upload of something that already exists
> and is currently thrown away** when the agent is torn down.
>
> It also means the published spec would be generated on a build agent rather than someone's
> laptop — so **the timezone contamination from Part 2 disappears**, and a drift check becomes
> viable without waiting on PR #6303.

> [!tip] It also makes Alastair right
> *"It should result from the build process. It should not be something that we treat as source
> code."* — publishing it as a per-version artifact is literally that. The committed copy can
> eventually become a convenience for local development rather than the thing we deploy from.

---

## Part 5 — The background argument you should know about

There's an unresolved disagreement sitting underneath all of this.

**SRE's position** — Alastair Andrew, 17 July:
> *"The `openapi.v1.json` is an auto-generated file that should result from the build process.
> It should not be something that we treat as source code."*

**Sam Clifford agreed**, and added that the current IaC *"applies the openapi spec with no
consideration as to which version of core might be running"* — and that the team automating
APIM (us) would need to look into it.

**Voodoo Park's position** — Laszlo, 17 July:
> *"We don't want to ignore the file because it's needed for the SRE when they update the apim
> instances."*

So: Voodoo Park keep the file in the repo **because SRE need it**, while SRE say it shouldn't
be in the repo at all. Both are right about their own half. Nobody has reconciled it.

**Publishing it as a versioned artifact resolves the argument** — SRE get a proper build
artifact, and it stops being something people hand-copy.

---

## Part 6 — What DEVO-28370 was actually asking for

Having read the ticket against the evidence: **it's sound.** It names two real, specific defects
— the conflicting file references, and the 7D policy-overwrite risk. It's blocked by our spike,
which is correct sequencing.

What it *doesn't* capture is Part 4 — the version alignment problem. That came from Sam.

| What the ticket asks | Where we've got to |
| --- | --- |
| Define a single source of truth | ✅ Already decided in ADR-002 — `Infrastructure/openapi.v1.json` |
| Remove conflicting guidance | ✅ Identified — guide Phase 7 is wrong; `entities.json` is dead on our branch |
| Stop policy being lost on re-import | ✅ Solved by construction once declared in Pulumi (§7.5.3) |
| A repeatable drift check | ✅ Free once the spec is a Pulumi input (§7.5.1) |
| *(not in the ticket)* Spec matches deployed Core version | ✅ **The plan below** |

---

# Part 7 — The plan

**Publish the spec as a versioned build artifact, and have the IaC fetch the one matching the
deployed Core version — falling back to the repo file when running locally.**

## 7.0 Why this matters more than "keeping a file in sync"

This is the framing Sam gave on 2026-08-07, and it's a better argument than the one this doc
originally made.

> *"Right now we deploy infrastructure alongside the core application, because that's not where
> we want to be in a couple of years. In a perfect world we'd be able to do blue-green
> deployments and be able to deploy some services in isolation. And that's where we want to get
> to — and I think publishing this artefact and then consuming it will unblock that sort of
> future vision."* — Sam Clifford

**Today, infrastructure and Core are welded together.** The orchestrator runs as part of the Core
deploy pipeline, and one of the things it does is update container image tags so they match the
Core version being deployed. That works, but it means infrastructure can only move when Core
moves.

**An artifact breaks that link.** Once the spec is published under a version tag and fetched by
version, the IaC no longer needs to be running *at the same moment* as the Core build to get the
right file. It can run later, independently, and still be correct.

There's a second strategic thread. Sam mentioned that one of the reasons for having APIM at all
is to **break the Core application down** — pointing requests at new services behind the same
gateway. He traced that back to a conversation with Jonathan about a year ago and was careful to
call it *"an enabler, not a commitment."* Not our problem now, but it explains why the gateway
is worth doing properly.

> [!tip] Use this framing on the PR
> "Keeps the spec aligned with Core" is a maintenance argument and reviewers will weigh it as
> such. "First step towards deploying infrastructure independently of Core" is the reason Sam
> endorsed it.

---

### Why this over the alternatives

- It's the only option that actually answers Sam's point rather than deferring it
- Every piece copies something that already exists and works — nothing invented
- Stage 1 is free; the generation is already happening and being discarded
- It resolves the SRE-vs-Voodoo-Park argument rather than picking a side
- **It doesn't block us** — the local-file path works today

### Scope

**In scope:** publishing the spec per version; fetching the version-matched spec at deploy time;
local fallback; detecting when the deployed spec is out of date.

**Out of scope, deliberately:** changing how the spec is *generated* (NSwag config, processors,
ADR-002 all untouched); removing `openapi.v1.json` from source control (a later conversation once
the artifact path is proven); multi-version v2 APIM setup (must not be *blocked*, but Phase 7
imports v1 only); rate limiting via `x-apim-rate-limit` (mechanism exists but is dormant — no
controller carries the attribute).

**Newly out of scope, confirmed 2026-08-07:** API **versioning headers** in the Core application.
Sam: *"Hard, I would say. It's probably out of scope for this work, but just worth looking at,
because you'd need to change the actual APIs in the core application."* Recorded in §7.6 so the
idea isn't lost.

---

## 7.1 End to end

```
┌─ BUILD ──────────────────────────────────────────── already happens, no change ─┐
│  Dayshape.Core.Release.yml:174 → Dayshape.Core.Build.yml (Configuration=Release) │
│  Dayshape.csproj:701  GenerateOpenApi  AfterTargets="Build"                      │
│      → nswag.generate.ps1 globs NSwag/nswag.v*.json                              │
│      → writes Infrastructure/openapi.v{N}.json  on the build agent               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                    ↓  file is on disk, same job, same checkout
┌─ PUBLISH ─────────────────────────────────────────────────────────── new, small ─┐
│  Dayshape.Core.Release.yml:~183 → Dayshape.OpenApi.Publish.yml → .ps1            │
│      → az storage blob upload                                                    │
│        dsartifactrepo/{release|releasecandidate}/openapi/{versionTag}/openapi.v1.json │
└──────────────────────────────────────────────────────────────────────────────────┘
                                    ↓  weeks or months later
┌─ CONSUME ───────────────────────────────────────────── new, copies LanguagePack ─┐
│  ApimSpike component, during the Pulumi program                                  │
│      → resolve version   (CORE_SOFTWARE_VERSION)                                 │
│      → in DevOps: download the matching blob                                     │
│        locally:    read Infrastructure/openapi.v1.json from the repo             │
│      → hand the text to apimanagement.Api(value=...)                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Why the version tag is trustworthy

`Dayshape.Core.Release.yml` computes `versionTag` **once** and uses it for both the build and
the artifact publish:

| Release type | `versionTag` | Container |
| --- | --- | --- |
| Tagged release (`refs/tags/v*`) | `v26.5.0.0` | `release` |
| Beta (`refs/tags/beta/*`) | the tag | `release` |
| Anything else | first 8 chars of commit SHA | `releasecandidate` |

One pipeline run = one version = one build + one spec filed under that version. The spec
published under `v26.5.0.0` was generated from *exactly* the code that produced the 26.5.0.0
binaries.

---

## 7.2 Stage 2 — the publish step

> [!question] Open — its own CI pipeline, or piggyback the Core release?
> Sam said two things on 2026-08-07 that pull in slightly different directions.
>
> **On decoupling:** *"Currently the CI pipeline for the core application builds the core
> binaries… when it builds the API, that's a bit you're going to need to decouple. It shouldn't
> be built alongside the [core] project, it should be kind of its own CI pipeline so that we can
> build it and deploy it sort of separate."*
>
> **On how to build it:** *"Ideally you could copy what I did for the language packs. I might
> have piggybacked off of that pipeline, I can't really remember."* — he did piggyback; that's
> exactly what the code shows.
>
> **My reading:** a separate CI pipeline is the *destination*, and piggybacking on the Core
> release is the acceptable *first step* — same as language packs do today, and consistent with
> working rule 2 (copy the existing pattern). The tables below assume the piggyback.
>
> **This needs confirming before writing the files**, because it decides where they live and
> whether a separate pipeline definition is needed. Open question 2.

| File | Status | Size | Modelled on |
| --- | --- | --- | --- |
| `pipelines/Templates/Core/Dayshape.OpenApi.Publish.ps1` | new | ~40 lines | `Dayshape.LanguagePack.Publish.ps1` |
| `pipelines/Templates/Core/Dayshape.OpenApi.Publish.yml` | new | ~18 lines | `Dayshape.LanguagePack.Publish.yml` |
| `pipelines/Templates/Core/Dayshape.Core.Release.yml` | modified | ~4 lines | — |

Script shape — simpler than the language pack one, no temp dir, no `Compress-Archive`:

1. Validate `VersionTag` present, and matches `^v\d{2}\.\d+\.\d+\.\d+$` when `IsRelease`
2. **Glob** `Infrastructure/openapi.v*.json`; error if none found
3. Pick container: `release` if `IsRelease`, else `releasecandidate`
4. For each spec file: `az storage blob upload` to `openapi/{VersionTag}/{filename}`
5. Log the resulting blob path

> [!important] Glob, don't hardcode `v1`
> `nswag.generate.ps1` is deliberately version-agnostic — its own header says *"Adding a new API
> version requires only a new `nswag.v{N}.json` file; no changes to this script or the .csproj
> are needed."*
>
> If our publish step hardcodes `openapi.v1.json`, we silently break that promise the day v2
> lands: generation would produce v2, publishing would ignore it, and nobody would notice until
> APIM was missing an API version. **Mirror the glob.**

> [!note] Drop the dead parameter
> `Dayshape.LanguagePack.Publish.ps1` declares `$SubscriptionId` and **never uses it** — auth is
> via `--auth-mode login` and the `Dayshape.Platform.Artifact` service connection in the YAML
> wrapper. Don't carry it across. (Worth mentioning to whoever owns that script.)

---

## 7.3 Stage 3 — the consume step

### ⚠️ This is a loader, not a post-deployment hook

Worth being explicit, because the LanguagePack comparison makes "hook" the obvious guess and
**it's wrong**.

| | LanguagePack | OpenAPI spec |
| --- | --- | --- |
| What it does with the artifact | Copies blob → blob | Passes the **content** into a resource |
| When it can run | After everything is deployed | **During** the Pulumi program |
| Mechanism | Post-deployment hook | Plain function called while declaring resources |

`apimanagement.Api` takes the spec as its `value` input. Inputs must exist when the resource is
declared, so this cannot be deferred to a hook. The **version-resolution and artifact-fetch logic
copies near-verbatim** from `deploy_base_language_packs.py` — only the *placement* differs.

There's also direct prior art for the loader: `apim_openapi_loader.py` in the old
`Products/Spoke/APIM/` component (still on `develop`). Its `load_openapi()` already accepts
either a URL or a local path, and validates that every operation has an `operationId` — worth
lifting, since a missing `operationId` produces unstable APIM operation names.

### Behaviour

```
resolve_spec() →
    if not running in DevOps:
        read Infrastructure/openapi.v{N}.json from the repo (via find_repo_root())
    else:
        version = CORE_SOFTWARE_VERSION
        if missing or unexpanded ("$(...)"):  fail loudly — do not guess
        container = "release" if version matches vNN.N.N.N else "releasecandidate"
        download  openapi/{version}/openapi.v{N}.json
        if not found: fail loudly
    validate every operation has an operationId
    return the raw JSON text
```

Both failure modes are **hard failures**, matching LanguagePack. A missing version or a missing
artifact must not silently fall back to the repo copy in DevOps — that would reintroduce exactly
the drift this design exists to remove.

| File | Status | Modelled on |
| --- | --- | --- |
| `Products/Spoke/ApimSpike/infra/apim_spike_openapi.py` | new | `deploy_base_language_packs.py` + `apim_openapi_loader.py` |
| `Products/Spoke/ApimSpike/tests/apim_spike_openapi_unit_test.py` | new | `deploy_base_language_packs_test.py` |
| `Products/Spoke/ApimSpike/infra/apim_spike_component.py` | modified | — |

---

## 7.4 Where each configuration value lives

Applying the model agreed with Richard and Marcus ([[Meeting Notes - Richard and Marcus 2026-08-05]]):

> **App Config** — dynamic values that change per environment, per deploy, or over time
> **GSOT** — values that don't change *and* are shared across multiple components
> **Variables in code** — everything else

| Value | Example | Where | Why |
| --- | --- | --- | --- |
| Artifact storage account | `dsartifactrepo` | **Code constant** | Fixed. LanguagePack hardcodes it as `ARTIFACT_ACCOUNT_NAME` — see note |
| Release container | `release` | **Code constant** | Fixed; LanguagePack precedent |
| Release-candidate container | `releasecandidate` | **Code constant** | Fixed; LanguagePack precedent |
| Blob directory | `openapi` | **Code constant** | Fixed; mirrors `CONTAINER_LP_DIR` |
| Local fallback path | `Infrastructure/openapi.v1.json` | **Derived in code** | From `find_repo_root()`, as LanguagePack does |
| Deployed Core version | `v26.5.0.0` | **Environment variable** | `CORE_SOFTWARE_VERSION`, already set by the orchestrator |
| API version to import | `v1` | **Code constant (for now)** | Only v1 exists — see open question 5 |
| SAS token lifetime | 15 minutes | **Code constant** | LanguagePack precedent |

> [!success] The headline: this needs almost no new config
> Every value is either a fixed constant or already supplied by the environment. **Nothing new
> goes into App Config.** A good sign we're following the grain of the existing design rather
> than cutting across it.

> [!question] One tension to raise with Richard
> Strictly, `dsartifactrepo` fits the **GSOT** definition — it doesn't change *and* it's now
> shared by two components (LanguagePack and us).
>
> But LanguagePack hardcodes it, and working rule 2 says copy the existing pattern rather than
> invent a better one. Creating a GSOT entry for a single value that an existing component
> hardcodes would be a deviation.
>
> **Proposal:** hardcode it, matching LanguagePack, and raise "should the artifact account move
> to GSOT?" separately rather than deciding it inside this ticket.

---

## 7.5 Keeping track of updates

This is DEVO-28370's *"repeatable validation step"*. Most of it is free.

### 7.5.1 Pulumi's own diff is the drift check

Once the spec text is an **input to a Pulumi resource**, it's in state. So:

```
pulumi preview
    ~ azure-native:apimanagement:Api  ...-entities-api
        ~ value: (diff)
```

**`pulumi preview` tells you the deployed spec differs from the current one.** No checksum
tooling, no bespoke comparison script, no new job. It's the standard IaC mechanism, and it
satisfies the ticket's ask directly.

> [!warning] This only works if the timezone noise is gone
> Today the spec bakes in the generating machine's timezone, producing ~90 meaningless diff
> lines (Part 2). That would make `pulumi preview` report a change on essentially every
> regeneration — noise that trains people to ignore the diff.
>
> **Generating on the build agent fixes this**, which is why stage 2 earns its keep twice.

### 7.5.2 Recording which spec version is deployed

Useful for anyone looking at APIM and asking *"what's actually loaded here?"*.

**Proposal:** an `apimanagement.NamedValue` recording the resolved version — e.g.
`openapi-spec-version` = `v26.5.0.0`. Reuses the named-value pattern already established in
Phase 5, is visible in the portal without digging into Pulumi state, and gives SRE a straight
answer during an incident.

> [!caution] This is a deviation from the setup guide — raise it, don't sneak it in
> The guide doesn't mention any such named value. Per working rule 5, flag it and let the room
> decide. It's genuinely optional; the design works without it.

### 7.5.3 The re-import problem solves itself

Guide 7D warns that re-importing the spec **wipes the API policy**. True in the portal, where
import and policy are two separate manual acts.

In Pulumi they're declared **together**, in the same program:

```
Api(...)         ← the spec
ApiPolicy(...)   ← the HMAC / CORS policy
```

Re-running the deploy reapplies both. **The failure mode disappears by construction** — there is
no window in which the spec has been updated and the policy hasn't.

This is why Phase 7 and the guide's section 12 ("update existing Entities API with new OpenAPI
specification") are **one design, not two**. Section 12 *is* Phase 7, run again.

### 7.5.4 What we are deliberately not building

- **A file-comparison / checksum drift job** — `pulumi preview` already does it, and a bespoke
  check would be ~90 false positives per regeneration until the timezone fix lands
- **A scheduled sync** — the orchestrator run *is* the sync point (confirmed 2026-08-07)

---

## 7.6 The deployment-window problem — raised 2026-08-07, not solved

This came out of the call and doesn't have an answer yet. It sits **next to** this design rather
than inside it, but it's the natural follow-on so it's recorded here.

### What Sam actually raised

He opened by asking whether we'd ever want **two API specs live at once** — 26.4 and 26.5 side by
side. I pushed back, because if APIM advertises endpoints Core doesn't have yet, callers get a
404. After some back-and-forth the real concern turned out to be narrower:

> *"There's a race condition. Let's say there's a breaking change to one of the endpoints in the
> core application — then the API spec's going to change, it has to change at some point in time.
> So there will be a moment in time where APIM is newer or older than the core app it's talking
> to. So any integrations interacting with APIM might fail during that window when otherwise they
> wouldn't. So how do you prevent those failures?"*

**So it is not about supporting two versions permanently.** It's about the minutes or hours during
a deployment when APIM and Core disagree. Sam confirmed: *"during a deployment, it would happen."*

### Why it matters

APIM's audience is different from the app's. A person hitting a broken page retries. A
**customer's automated integration script** hitting an unexpected error may fail the job, retry in
a tight loop, or silently drop data. Sam's framing:

> *"You guys need to figure out who your users are. We don't want outages. If one of your users
> is a little script that's a customer's custom integration — how does that integration react to
> outages?"*

Richard put the operational version of the same question:

> *"How do we tell the customer's script that it needs to have an outage from now to now while we
> update everything, including the Core and the APIM?"*

### Options discussed — none decided

| Option | Raised by | Notes |
| --- | --- | --- |
| Return a meaningful **`503 Service Unavailable`** from APIM during a deploy | Sam | Gives integrations something unambiguous to back off on, rather than an assortment of 404s and timeouts |
| **Maintenance mode for the API** | Richard | Core already has maintenance mode plus a VM disable. APIM has no equivalent — it would keep serving while Core is down |
| **Versioning headers**, backwards compatibility, mark-obsolete-and-redirect | Sam | What Reporting already does. **Sam: probably out of scope** — needs changes to the Core APIs themselves |
| Decide the **deploy sequence** — Core first then APIM, or the reverse | Richard | **Left unresolved.** Needs an answer before rollout |

### Related warning

Richard mentioned a recent incident where a customer believed Stage was down for a day, when in
fact **cache headers** were still serving a maintenance response after the outage ended. Being
fixed separately — but it's the same class of problem: the gateway telling callers something the
app no longer means.

> [!question] What I'd suggest
> Treat this as a **separate ticket after Phase 7**, not part of it. It's a behavioural /
> product question ("what should a customer integration see during a deploy?") and answering it
> properly needs Gabriela as much as it needs SRE.
>
> Phase 7 doesn't make it worse — today APIM has no version awareness at all, so the artifact
> design is strictly an improvement.

### The unowned bit

Sam flagged something nobody appears to own:

> *"A whole piece of this work is how do we help customers build custom integrations, and how do
> we give them what they need to do it. APIM is part of the puzzle. But it's more work."*

Worth raising with Gabriela — it's a product question hiding inside an infrastructure ticket.

---

## Part 8 — Work breakdown

| # | Task | Status / proof |
| --- | --- | --- |
| 1 | `Dayshape.OpenApi.Publish.ps1` + `.yml` | **Implemented as PoC code.** Globs every `openapi.v*.json` and publishes under `openapi/{versionTag}/`. |
| 2 | Wire into the pipelines that produce a Core Release build | **Implemented.** Core Release and HighVolume publish; FrontEnd-only deliberately does not because it does not run NSwag. |
| 3 | Verify a release run actually publishes the blob | **Follow-up, not a PoC gate.** Check on the next real release run. |
| 4 | Artifact-store access for the IaC identity | **Closed.** `dsartifactrepo` already grants account-scoped *Storage Blob Data Contributor* to `pulumi.serviceprincipals.defaultpermissions`. No new role assignment or SRE action is needed. |
| 5 | `apim_spike_openapi.py` — version resolution, fetch, local fallback, `operationId` validation | **Implemented and unit tested.** |
| 6 | Unit tests | **Implemented.** Local/artifact resolution, failure paths, validation and component resource inputs are covered without Azure access. |
| 7 | Wire the loader into the component and declare the version set, API, policy and product link | **Implemented.** The PoC can now be deployed and proven from the local repo spec. |

The complete chain is now represented in code, but the proof is intentionally split:

- **PoC proof now:** local spec → Pulumi → versioned Entities API → policy → Default Product.
- **Supply-chain proof later:** release build → artifact blob → DevOps resolver. This waits for a
  real release run and does not block the spike.

### Testing

- **Unit tests** for version resolution — release vs release-candidate container selection,
  missing version, unexpanded `$(...)` variable, missing artifact. `deploy_base_language_packs_test.py`
  already sets `os.environ["CORE_SOFTWARE_VERSION"] = "v22.1.0.0"` — copy that approach.
- **Unit tests** for `operationId` validation, including a spec with one missing.
- **No Azure access in tests** — `ComponentUnitTest` forbids it; blob access must be mocked.
- **Expect G14 again** — `apimanagement.Api` will surface fewer outputs than inputs, so
  assertions on input-only properties need the capture hook. See [[APIM IaC - known traps and gotchas]] G14.

---

## Part 9 — Open questions

### Answered on 2026-08-07

| # | Question | Answer |
| --- | --- | --- |
| ~~1~~ | How often does the orchestrator run? Is `CORE_SOFTWARE_VERSION` set on every run? | **Answered — see below.** It runs on every customer version upgrade, plus manually. Version is set on the deploy path. |
| ~~2~~ | Any objection to adding the publish step to the Core release pipeline? | **No — Sam endorsed it.** *"Yeah, yeah, absolutely."* Superseded by new question 2 below (which pipeline). |
| ~~3~~ | Own CI pipeline vs piggyback on the Core release? | **Core release pipeline.** The spec is generated by its Release build, and the publish step runs immediately afterwards. |
| ~~4~~ | Do the other language-pack publish callers need the spec? | **HighVolume: yes; FrontEnd-only: no.** HighVolume performs a Core Release build. FrontEnd-only does not generate a spec and must not publish a stale checkout copy. |
| ~~5~~ | Does the IaC identity need a new artifact-store role? | **No.** The shared Pulumi identity already has account-scoped *Storage Blob Data Contributor* on `dsartifactrepo`. |

**On question 1 — the detail, because it changes how urgent this work is:**

The orchestrator has **two triggers**:

1. An **`IaC Deploy` pipeline** — manual, lets you pick which components to run. Used both for
   sandbox testing and for real production runs. *(Exact pipeline name unconfirmed — Sam
   bracketed it as "versioning" in the call. Worth checking.)*
2. The **full software deployment pipeline** — the same pipelines that deploy Core to a customer
   also run the orchestrator.

Which components run comes from **a table in the GSOT database** listing what's enabled per
customer environment. It's a **diff, not a rebuild** — existing components are checked for
changes rather than torn down.

There's already precedent for exactly our problem: the IaC **updates container image tags** so
web apps match the deployed Core version.

I put my assumption to Sam directly — *"a new version comes out, we run the deploy pipeline, and
that runs the orchestrator"* — and he confirmed: **"Yeah, that's it."**

> [!important] What this means for the urgency of this work
> In the **normal** case, spec and app move together and `CORE_SOFTWARE_VERSION` will be set. So
> the version-alignment work is **not** fixing a live breakage — it's making the abnormal cases
> safe (manual APIM-only deploys, re-runs, environments upgraded out of band) and unblocking
> independent infrastructure deploys later.
>
> Still worth doing, and still the right design. But don't oversell it as firefighting.

### Still open

| # | Question | Who | Blocks |
| --- | --- | --- | --- |
| 1 | Confirm the exact name of the `IaC Deploy` pipeline | Sam C / Andy L | Nothing — documentation accuracy |
| 2 | Should `dsartifactrepo` move to GSOT, or stay hardcoded per LanguagePack? | Richard | Nothing — can follow later |
| 3 | Is the `openapi-spec-version` named value wanted, or scope creep? | Richard / Marcus | Nothing — optional |
| 4 | Did PR #6303 (timezone fix) ever land? | VP / Laszlo | Nothing — stage 2 fixes it anyway |
| 5 | **Deploy sequencing during an upgrade** — Core first or APIM first? See §7.6 | Richard / SRE | Nothing here — blocks rollout |

---

## Part 10 — Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| ~~Permissions grant takes weeks~~ | ~~Artifact path can't be proven~~ | **Closed:** the shared Pulumi identity already has account-scoped artifact access |
| Publish step lands in the wrong pipeline | Rework | Question 2 — settle before writing the files |
| Spec published is stale because a dev forgot to regenerate | APIM advertises the wrong shape | **Generation is automatic in Release builds** — already mitigated by stage 1 |
| ~~`CORE_SOFTWARE_VERSION` not set on some orchestrator runs~~ | ~~Hard failure mid-deploy~~ | **Reduced.** Confirmed set on the deploy path. Fallback still needed for manual `IaC Deploy` runs |
| v2 arrives and publishing ignores it | APIM missing an API version | Glob rather than hardcode (§7.2) |
| APIM and Core disagree during the deployment window | Customer integrations fail unpredictably | **Unsolved — see §7.6.** Separate ticket; Phase 7 doesn't make it worse |

---

## A note on the setup guide

Sam volunteered this on 2026-08-07, unprompted:

> *"I am 99% sure that they built it from the IaC. So they built the IaC with AI, then they
> pointed AI at the IaC and said 'build me a doc', and then they handed it to us. The only human
> element is modifications me and Marcus have made to it. So you need to assume it's not
> something you should take verbatim. It's a starting point, but I wouldn't use that as the
> decision token for how you phase the work."*

**Relevant here because this doc leans on the guide in two places** — the phase ordering, and the
guide's choice of which spec file to import. Both remain reasonable, but "the guide says so" is no
longer evidence on its own. Where this doc reaches a different conclusion from the guide
(§7.1 on the spec path in particular), that's now the expected outcome rather than a deviation
needing justification.

Working rule 5 has been amended to match — see [[APIM IaC - working rules for implementation]].

---

## Related

- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — **where this was endorsed**; full notes on version skew, testing and rollout
- [[Meeting Notes - Richard and Marcus 2026-08-05]] — the config placement model used in §7.4
- [[APIM IaC - working rules for implementation]] — rule 5, amended after the guide's provenance came out
- [[APIM IaC - rollout and testing strategy]] — the environments theme from the same call
- [[APIM IaC - ticket breakdown]] — T7 is where this lands
- [[APIM IaC - known traps and gotchas]] — G14 applies to the `Api` resource; the timezone noise belongs here too
- [[APIM IaC - working rules for implementation]] — rules 1, 2 and 5 shape this design
- [[Meeting Notes - Sam Clifford on VP APIM IaC gaps]] — where the version-alignment point came from
- `docs/adr/002-openapi-generation-strategy.md` — the accepted decision on spec generation
- `docs/runbook/002-APIM-Nswag-OpenApi-generation.md`
