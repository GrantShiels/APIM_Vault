---
tags: [apim, iac, secrets, key-vault, hmac, devo-28368, decision]
updated: 2026-08-10
status: superseded by the code — built and merged (commit 2831472)
related-ticket: DEVO-28368
---

> [!warning] Audited 2026-10-07
> Belongs to DEVO-28372 (First Review). Core's check exists; the APIM-side signing policy is not in the branch yet.

# APIM HMAC signing secret — where does it come from, and how do both sides get the same value?

> [!warning] Superseded by the code
> Phase 5 is **built, merged and deployed** (commit `2831472`). The design here is the reasoning that led to it.
>
> **Where this document and the code disagree, the code is right.** Kept for the reasoning behind
> the decisions, not as a description of how things behave today.

> Written before implementing Phase 5.
> **This is a decision record.** It sets out what the problem looked like, what investigation
> actually found, the options weighed, and what was chosen. Written up first so the reasoning
> survives independently of the code.

---

## What Phase 5 is, in plain English

APIM sits in front of the Dayshape API. We want Dayshape to be able to prove a request genuinely
came *through* APIM, rather than someone going round the side and hitting the API directly.

The mechanism is a shared password. APIM uses it to stamp a signature onto every request it
forwards. Dayshape uses the same password to check that stamp. No match, no entry.

Phase 5 is the job of creating that password and making sure both sides end up holding the
identical value.

---

## The question, in one sentence

> **Who creates the signing secret, where does it live, and how does the same value reach both APIM and the Dayshape application?**

---

## Why this looked like it would be hard

On the face of it this is the [[APIM IaC - client secret decision|Phase 4C client secret problem]]
all over again — a secret that has to exist in two places, with the setup guide instructing a human
to copy and paste it.

The guide's Phase 5 reads:

1. Run a PowerShell one-liner to generate a random string
2. Save it in Keeper under the customer's entry
3. Paste it into APIM as a Named Value
4. Paste it into the core ADO variable group as `ApimHmacSecret`
5. Redeploy, or hand-edit the config file

Five steps, three of them copy-and-paste, repeated per customer. That does not survive automation,
and at 100+ customers it does not survive a human either.

### But it is a much easier problem than 4C, and it is worth being precise about why

The 4C client secret is genuinely hard for reasons that have nothing to do with copy-and-paste:

- **Azure generates it**, not us — so automation cannot choose the value
- **Azure shows it once** and then it is unreadable forever, even to its creator
- **It belongs to Core's app registration**, which the `Auth` component owns — so the real question
  is one of *ownership*, not storage

**The HMAC secret has none of those properties.** It is a random string that we invent. Nothing in
Azure generates it, nothing hides it from us, and it does not belong to any app registration. There
is no ownership argument to have.

So the only real question was **delivery** — how the same string reaches two different systems. That
turned out to be a much smaller problem than expected, for the reasons below.

---

## What investigation actually found

Three discoveries, each of which shrank the job.

### 1. The Dayshape half already exists

`Dayshape/Dayshape/Controllers/proxy/Filters/RequireApimOriginAttribute.cs` was added by **Anton
on 2026-01-08** (commit `782bdfb37ef`, *"Add APIM gateway verification configuration settings"*).

It is complete, has 30 unit tests, and is already applied to `BaseProxyController` via a
`[RequireApimOrigin]` attribute. It reads three settings:

| Setting | Purpose | Default in code |
| --- | --- | --- |
| `RequireApimOrigin` | On/off switch | `false` |
| `ApimHmacSecret` | The shared password | none |
| `ApimTimestampToleranceSeconds` | Allowed clock drift | 300 (5 minutes) |

I checked its signing method line by line against the policy printed in the setup guide. They match
exactly — same three headers (`X-APIM-Timestamp`, `X-APIM-Nonce`, `X-APIM-Signature`), same string
to sign, same HMAC-SHA256, same base64url encoding of the result.

**So Phase 5 is not building a feature. It is supplying the one missing input to a feature that is
already built and tested.** That is a materially different, and smaller, piece of work than the
guide implies.

### 2. Half the Key Vault plumbing already exists — but not the half that matters

I expected to have to build a route from "secret in Key Vault" to "value the Dayshape app can
read". A route exists, and `Spoke/GenericSync` already uses it:

- `config_interface.set_config(secret_id=...)` writes a Key Vault **reference** into App
  Configuration — a pointer to a vault entry, not the value itself
- Every read already passes `AzureAppConfigurationKeyVaultOptions`, so those pointers are resolved
  back into real values automatically and transparently

> [!warning] Corrected 2026-08-10 — this route does not reach Core
> Phase 5 was originally built on this, and it was **wrong**. App Configuration is read by the
> modern services. **Core is not one of them.** It runs on a VM as the legacy IIS application and
> reads plain-text settings from `Dayshape.config` on disk. Nothing bridges App Configuration to
> that file — I searched twice.
>
> The real route is in [[#How the secret actually reaches Core]] below. The consequence for the
> secret is significant: **Core needs a literal copy**, so the "one copy in existence" property
> claimed further down only holds for the APIM side.

### 3. The secret is used as plain text, not decoded

`RequireApimOriginAttribute` does:

```csharp
var keyBytes = Encoding.UTF8.GetBytes(secret);
```

It does **not** base64-decode the value. So the guide's `[Convert]::ToBase64String(...)` one-liner
is simply a convenient way to produce a random printable string — the base64-ness carries no
meaning to either side.

Small detail, real benefit: it means we are free to pick a character set, and can avoid characters
that would need escaping when the value is later referenced inside the Phase 7C policy XML.

---

## What the codebase already does

Everything Phase 5 needs already exists as a pattern somewhere in `Products/`.

| Need | Existing example | Notes |
| --- | --- | --- |
| Generate a random secret | `Modules/SqlServer`, `Modules/VirtualMachine` | `pulumi_random.RandomPassword`, with `additional_secret_outputs=["result"]` so it stays out of logs |
| Store a secret in Key Vault | `Modules/KeyVaultSecret` | Also clears any soft-deleted secret of the same name first — a trap it has already solved |
| Write into another component's vault | `Spoke/Certificate` | Uses `StorageHelpers.get_storage_kv_info()` to reach the spoke's shared vault |
| Write a Key Vault reference into config | `Spoke/GenericSync` | `set_config(secret_id=...)` |
| Write config belonging to another component | `Spoke/LanguagePack` | `set_config_for_different_component(...)` |
| Grant a principal vault access declaratively | `Modules/KeyVault/key_vault_security.py` | `get_kv_access()` returns an `entra.GroupMember` |

Nothing here is novel. That is a good sign — and it is the strongest argument that this approach is
the conventional one rather than an invention.

---

## The options

### Option 1 — Guide-faithful: a human generates it

Exactly as written. Person runs the one-liner, saves to Keeper, pastes into APIM and the ADO
variable group. Pulumi reads it from an environment variable, the same way the spike currently
handles the 4C client secret.

> [!note] Update 2026-08-13 — the comparison no longer holds, which strengthens the rejection
> The 4C client secret **no longer uses an environment variable**. The APIM deployment now creates
> it directly and replaces it every 30 days — see [[D1 - The developer portal sign-in secret]].
> So `APIM_IDP_CLIENT_SECRET` is gone from the codebase entirely, and this option would be
> **reintroducing** a pattern we have just finished removing rather than matching an existing one.

| Pros | Cons |
| --- | --- |
| Matches the guide word for word | Repeats the `APIM_IDP_CLIENT_SECRET` env-var pattern that Richard's review already marked **Major** |
| No new machinery at all | A manual step per customer, at 100+ customers |
| | Two hand-entered copies that can silently drift apart |
| | Nothing in `Products/Pipelines/` supplies the variable, so a pipeline deploy fails before Pulumi starts |

**Rejected.** It reproduces a known-bad pattern to solve a problem that does not require it. The
only reason 4C tolerates an environment variable is that Azure generates that secret and hides it —
which does not apply here.

### Option 2 — Pulumi generates it, APIM only

Pulumi creates the secret and puts it into APIM. The Dayshape side stays a documented manual step
until Phase 7 makes it testable.

| Pros | Cons |
| --- | --- |
| Smallest possible change | Leaves the two halves able to drift, which is the whole failure mode we are trying to remove |
| Spike-sized and quick | Defers the interesting question rather than answering it |
| | Half a solution is arguably worse than none — it looks finished |

**Rejected.** The delivery problem *is* the problem. Solving everything except that solves nothing.

### Option 3 — Pulumi generates it, a deployment hook delivers it

Pulumi generates the secret, stores it in the spoke's shared Key Vault, points APIM at it, and an
ApimSpike deployment hook delivers it to the Dayshape side.

| Pros | Cons |
| --- | --- |
| No human, no copy-paste, nothing to keep in step | Imperative rather than declarative |
| Touches only `ApimSpike` — small blast radius | Each future secret would repeat the same code |
| Existing precedents in the repo | |

> [!note] Where the hook writes — corrected 2026-08-10
> This was originally implemented as a write to **App Configuration**, on the assumption Core read
> from there. It does not. The hook now writes to the client's **`_Core` Azure DevOps variable
> group**, following the Svix webhooks precedent. The choice of *a hook* was right; the choice of
> *destination* was not. See [[#How the secret actually reaches Core]].

### Option 4 — Pulumi generates it, a new declarative config provider delivers it

As Option 3, but instead of a hook, add a `keyvault` provider to `Modules/Config` so the delivery
can be declared as a CSV row alongside the existing `graph`, `gsot` and `variablegroup` providers.

| Pros | Cons |
| --- | --- |
| Declarative and reusable for every future secret | Changes code that **every component** runs through |
| The tidier long-term answer | Requires a pre-existing bug to be fixed first (see below) |
| Consistent with how all other config is expressed | Larger review surface, and not ours to own |

---

## What was decided

**Option 3 — the deployment hook — with Option 4 recorded as a follow-up.**

The reasoning:

- Both options produce the *same outcome* for the customer. The difference is purely how the
  delivery is expressed.
- Option 4 modifies shared machinery every component depends on, during a spike, to satisfy a
  single consumer. That is the wrong order — prove the mechanism works end to end first, then
  generalise it.
- Option 4 is also **currently blocked by a bug** (below), which is not ours to fix on this ticket.
- Option 3 has two working precedents in the repo, so it is not a novel pattern.

Once Phase 7 proves the whole chain — secret → APIM → signature → Dayshape check — promoting the
hook to a declarative provider is a small, well-understood change with a proven mechanism behind it.

### Two supporting decisions

**Which Key Vault?** The spoke's shared vault, owned by `Spoke/Storage`. `Spoke/Certificate` already
writes into it from another component using `StorageHelpers.get_storage_kv_info()`, so this is an
established route, and it avoids standing up a vault that would exist to hold one secret.

**Does APIM hold its own copy of the secret, or point at the vault?** **Point at the vault.** APIM's
Named Values support a `key_vault` reference as an alternative to a literal value. This requires
giving the APIM service a managed identity and read access to the vault — slightly more machinery,
but it means APIM never holds a copy that can drift, and a rotated secret reaches it without a
redeploy.

> [!note] Corrected 2026-08-10
> This section originally claimed there would be **exactly one** copy of the secret in existence.
> That holds for APIM, but **not** for Core, which needs a literal copy on disk. Two copies exist;
> only one of them follows the vault. That asymmetry is what drives the deploy-order constraint
> above.

**How do we handle the gap between granting access and using it?** **A 180-second
`pulumiverse_time.Sleep`, matching `Modules/AgwCertificate`.** Azure does not apply an access grant
everywhere the instant it is made, and Phase 5 both grants access and reads the vault in the *same*
deployment. A read attempted too early is refused, which leaves the named value holding an error
while the deployment still reports success — a silent failure.

A bespoke alternative was built and rejected: a Pulumi dynamic provider that asked APIM whether its
last read had worked and retried until it did. It functioned correctly, but came to roughly **400
lines across 5 files with 24 tests**, against **8 lines** for the sleep. `AgwCertificate` waits the
same 180 seconds after granting the same role on the same vault; `ContainerAppsEnv` and `SqlDb` do
the same for their own grants. Consistency with three existing components, and far less code to
maintain, outweighed the theoretical precision of checking.

---

## Confirmed working (2026-08-06)

Deployed to `grant-sbx`. The group membership and the named value were created in the **same run** —
the exact race the wait guards against — and APIM reported `lastStatus.code: "Success"` against the
named value, its own confirmation that it read the secret out of Key Vault. `provisioningState` came
back `"Succeeded"`. Whole component took 3m46s, 3m of which was the wait.

So the APIM half is proven end to end: generate → store → grant → wait → reference. What that run
did **not** prove is that the value reached Core — the hook reported success writing to App
Configuration, which Core never reads. See below.

---

## How the secret actually reaches Core

> **Corrected 2026-08-10.** The original Phase 5 delivery was wrong. This section replaces it.

### What was wrong

The hook wrote a Key Vault reference into **App Configuration**, under Core's namespace. That is a
real store, and the write genuinely succeeded — which is why the mistake was not obvious. But Core
is a **VM-hosted legacy IIS application**. It reads its settings from `Dayshape.config` on disk, and
nothing carries an App Configuration value into that file.

Two things confirmed it independently:

- Searching for any bridge between App Configuration and Core's config file found **nothing**, twice.
- `component_configs.csv` shows `variablegroup` is a config **provider** *feeding* App Configuration.
  So the variable group sits **upstream**. The hook was writing against the direction of flow.

### What is right

The setup guide had it correct all along. Step 5.4:

> *Add to the core variable group: `RequireApimOrigin` = true, `ApimHmacSecret` = \<HMAC-Secret\>.
> Then deploy so these update the dayshapeconfig file in braid.*

The chain, verified end to end:

1. Core's deploy lays down the `client-config` shards into `C:\braid\App_Data\configs`
2. `Dayshape.Core.Deploy.yml` runs an `updateconfig@1` step — the **secret** substitution path,
   which explicitly targets `Dayshape.config`
3. That step fills in `<add key="ApimHmacSecret" value="" />` from the `_Core` variable group
4. Core reads the file at runtime

**Critical detail:** the substitution steps only *overwrite existing* `<add>` nodes — they never
create new ones. This works only because `common/configs/Dayshape.config` in the `client-config`
repo already ships both keys.

### The precedent we copied

**Svix webhooks.** `Dayshape.IAC.PostDeploy.Svix.yml` already writes four values — two of them
secrets — into `$(LegacySpokeRef)_Core` after an IaC deploy, using `-DeleteVariablesFirst $false` so
the group's other variables survive. Its `Webhooks.config` has the same empty-placeholder shape as
`ApimHmacSecret`. Same problem, same group, same solution.

So this is not a new pattern. It is **the** pattern.

### How it is implemented

No pipeline YAML change was needed. `scripts/domain/config_sync_service.py` already builds
`{...}_Core` and uses the repo's own `scripts/domain/devops/variable_groups` module, so the hook uses
the same sanctioned Python API. No Azure DevOps REST calls of our own, no PowerShell.

The group name is derived, not looked up — the bridging script that seeds these groups builds it from
the same parts:

```
{client_group_id}-{client_id}-ds-{env_group}-{env}_Core
```

Two behaviours worth knowing, both covered by tests:

- **Azure DevOps replaces the whole group on update.** The hook fetches, merges, and writes back, so
  siblings like `JwtPrivateKey` survive.
- **A failed write raises rather than warns.** `update()` returns `None` on failure and only logs.
  Left unchecked, a silently missing secret would look like a successful deployment and then 403
  every request. See [[APIM IaC - known traps and gotchas|G23]].

### Environments with no Core

Sandbox and spike environments frequently run APIM with **no Core at all**, and those environments
have no variable groups either — confirmed on `grant-sbx`, which has four infra targets (AiAgent,
ApimSpike, Auth, Storage) and nothing in the Library.

So the hook asks first whether `Core` is an infra target for the environment. If it is not, there is
nothing that reads the secret, and it skips. If it *is*, a missing group is a genuine
misconfiguration and still raises.

That distinction is safe because the **same service creates both**: `config_sync_service` adds `Core`
to `ENV_STARTER_COMPONENTS` and creates `{env_prefix}_Core` in the same call. A client environment
cannot have one without the other.

### Permissions

Nothing to grant. The write runs under the pipeline's own identity (`SYSTEM_ACCESSTOKEN`), which is
how the six existing callers of the shared variable-group tool already work.

---

## ⚠️ Deploy order: APIM before Core

This falls straight out of the above, and it needs stating plainly because getting it wrong is not a
degraded service — it is a **total outage of the Proxy API**.

**The two sides hold the secret differently:**

| Side | How | Picks up a new value |
| --- | --- | --- |
| APIM | Points **at** the Key Vault entry | Automatically, no redeploy |
| Core | A **literal copy** on disk | Only on its next deploy |

Core cannot follow the vault. The config file is plain text and the application does not resolve
vault references, so it takes a copy at deploy time and then holds it, frozen, until it deploys again.

**Why that makes order matter:**

`RequireApimOrigin` is already `true` for every customer, and the filter **fails closed**. So Core
with a missing or stale secret rejects **everything** APIM sends it.

- **Build a new environment Core-first** → the variable group has no `ApimHmacSecret` yet, so Core
  comes up unable to verify anything, and 403s all traffic.
- **Rotate the secret** → APIM starts signing with the new value immediately; Core keeps checking the
  old copy until it is redeployed. That is a 403 window lasting **until the next Core deploy**, which
  could be days.

**So: deploy APIM first, and treat a rotation as requiring a Core redeploy.**

Recorded as **G23** in [[APIM IaC - known traps and gotchas]]. It also gives a concrete answer to one
strand of **G18**, where Sam and Richard left "Core first or APIM first" open — for the signing
secret, it is now settled.

---

## This answers an open question from the client secret decision

[[APIM IaC - client secret decision]] ends with an unresolved investigation item:

> *Does APIM read the secret via **Key Vault directly**, or does it get a managed identity and use APIM's **native Key Vault integration for Named Values**? (Not yet investigated — flagged.)*

**Phase 5 answers it, and will be the first place in the repo that mechanism is used.** A
system-assigned managed identity on the APIM service, granted Key Vault Secrets User via
`get_kv_access()`, with a `key_vault`-backed Named Value, is precisely that integration.

That is useful beyond Phase 5: if the mechanism works here, it becomes available as an option for
the still-open 4C client secret question too. Worth revisiting that decision once this is deployed
and proven.

---

## What we are deliberately not doing

**We are not switching verification on.**

`RequireApimOrigin` stays as it is. The policy that actually *applies* the signature to outbound
requests is **Phase 7C**, which does not exist yet. Turning the switch on now would leave Dayshape
demanding a signature that nothing is producing — and the filter fails closed, so the Proxy API
would start rejecting everything.

The consequence is that **the Named Value Phase 5 creates will be unused until Phase 7C lands.**
That is expected and correct, but it should be stated plainly rather than presenting Phase 5 as a
working feature in its own right.

---

## Open questions

### ✅ RESOLVED — `Dayshape.config` ships with verification on

**Resolved 2026-08-10.** The file I originally looked at was the in-repo template. The one that
actually ships is `common/configs/Dayshape.config` in the **`client-config`** repo, and it reads:

```xml
<add key="RequireApimOrigin" value="true" />
<add key="ApimHmacSecret" value="" />   <!-- must match Pulumi hmacSecret -->
<add key="ApimTimestampToleranceSeconds" value="300" />
```

So `RequireApimOrigin` is **already `true` for every customer** — it is not a gap, and nothing needs
turning on. The empty `ApimHmacSecret` is deliberate: it is a placeholder for the substitution step
to fill from the variable group, exactly as `WebhooksToken` is for Svix.

The comment claiming the default is `false` refers to the **code** default in
`RequireApimOriginAttribute`, which applies only when the setting is absent entirely. Not a mistake.

This also means the fail-closed risk in the deploy-order section above is real for every customer,
not just ones who have opted in.

### 🟠 A latent bug blocks the declarative route

`Modules/AppCfg/app_cfg_interface._set_config_raw()` contains, in its Key Vault branch:

```python
config_setting = SecretReferenceConfigurationSetting(
    key=key,
    label=label,
    secret_id=config_value,   # always None when this branch runs
)
```

That branch is only reached when `config_value` is falsy, so `secret_id` is always `None`.
`_set_config_raw()` therefore cannot write a Key Vault reference at all. Separately,
`ComponentConfigManager.manage_appcfg()` only ever passes `config_value`, never `secret_id`.

Our hook goes through `_set_config()`, which is correct — so this does not block Phase 5. It does
block Option 4.

- [ ] **Raise with Infra-Platform** alongside the request for a `keyvault` config provider.

### 🟡 Rotation

APIM picks up a regenerated secret automatically, because it references the vault. **Core does not**
— it holds a copy taken at its last deploy. So rotating leaves a window in which APIM signs with the
new value and Core rejects it, lasting until Core next deploys.

- [ ] Decide whether rotation is coupled to a Core release, or whether it needs a mechanism of its
      own. Until then, treat rotation as a release-window activity. See the deploy-order section
      above and **G23**.

---

## Related

- [[APIM IaC - client secret decision]] — the harder sibling problem; Phase 5 answers one of its
  open investigation items
- [[APIM IaC - known traps and gotchas]] — **G7** (secret expiry with no owner) is the failure mode
  this approach avoids by having nothing that expires; **G23** carries the deploy-order constraint
- [[APIM IaC - outstanding questions]]
- [[APIM IaC - ticket breakdown]]
- `Dayshape/Dayshape/Controllers/proxy/Filters/RequireApimOriginAttribute.cs` — the Dayshape half
- `pipelines/Templates/IAC/PostDeploy/Svix/Dayshape.IAC.PostDeploy.Svix.yml` — the precedent for
  writing secrets into the `_Core` variable group from an IaC post-deploy
- `pipelines/Templates/Core/Dayshape.Core.Deploy.yml` — the `updateconfig@1` step that fills
  `Dayshape.config` from that group
- `Products/scripts/domain/devops/variable_groups.py` — the sanctioned in-repo API the hook uses
- `Products/scripts/domain/config_sync_service.py` — the reference implementation of `_Core` group
  naming
- `Spoke/Certificate/infra/certificate_component.py` — reaching the shared spoke vault
