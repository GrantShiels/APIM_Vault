---
tags: [apim, iac, gotchas, devo-28368]
updated: 2026-08-19
status: living-doc
related-ticket: DEVO-28368
---

> [!warning] Audited 2026-10-07 — read with [[_START HERE - APIM handover]]
> Traps are still valid. Names are stale: `ApimSpike` = `Apim` on `iac-main`; "T3/T4/T7/T9, Ticket 1–8" map to
> 28371/28372/30005/30006 etc. (see start-here). G37 (name hold) conflicts: 48 h here, "several hours" in
> 28369, "up to 7 days" in a code comment — check before relying on it. G7/G38: Entra issues a 2-year
> secret and IaC replaces it every 30 days. Tier references to Basic v2 will change with Standard v2.

# APIM IaC — known traps and gotchas

> Part of [[APIM Findings Summary]] · A **running register of things that will bite us later** — latent
> traps found during the spike that are not yet a problem but will surface during implementation,
> testing or rollout.
>
> This is **not** the same as [[APIM IaC - outstanding questions]]. That register holds questions
> needing someone *else's* answer. This one holds things **we already know**, where the only risk is
> forgetting them.
>
> Status key: 🔴 will bite as-is · 🟠 mitigated in the spike, needs a real fix · ✅ handled, kept for
> the record.

## Register

| # | Trap | Status | Bites during |
| --- | --- | --- | --- |
| G1 | The `email` Graph scope is requested but never consented | 🔴 | Portal sign-in (4C) |
| G2 | Sign-in silently requires app-role assignment | 🔴 | First Try-It / portal login |
| G3 | The guide's policy XML is **not valid XML** as printed | 🟠 | Phase 3 deploy |
| G4 | `<base />` is rejected in the service-level policy | ✅ | Phase 3 deploy |
| G5 | Implicit flow is a deprecated dependency, not a choice | 🔴 | Long-term support |
| G6 | Running Auth without the flag strips the portal's redirect URIs | 🟠 | Any Auth deploy |
| G7 | The 4B/4C client secret expires after 12 months with no owner | ✅ | ~~12 months post go-live~~ — renews itself |
| G8 | The setup guide has confirmed omissions | 🟠 | Any phase |
| G9 | APIM cannot use its own app registration — ID token audience is fixed | ✅ confirmed | Any 4B/4C redesign |
| G10 | APIM child resources need an explicit name, or Azure rejects the deploy | ✅ | Phase 4C deploy |
| G11 | The guide's Phase 4D uses a portal API that the v2 tiers reject | ✅ | Phase 4D deploy |
| G12 | The Azure SDK rejects the success code Azure actually sends | ✅ | Phase 4D deploy |
| G13 | APIM may read the vault before its group membership has caught up | ✅ | Phase 5 first deploy |
| G14 | APIM resources return far fewer outputs than you pass in | ✅ | Any phase's unit tests |
| G15 | **Azure holds the APIM name for hours after deletion** | 🔴 | Any destroy-and-recreate |
| G16 | **The setup guide is AI-generated, not hand-written** | 🟠 | Any phase |
| G17 | **Secrets stored in APIM are not obfuscated in the portal** | 🔴 | Rollout / access review |
| G18 | **APIM keeps serving while Core is mid-deploy** | 🔴 | Every customer upgrade |
| G19 | A capture-based assertion must hang off the resource it is reading | ✅ | Any phase's unit tests |
| G20 | Single quotes inside a single-quoted policy attribute break the XML | ✅ | Any policy XML work |
| G21 | `create_availability_web_test()` returns nothing — assigning it gives you `None` | ✅ | Any component adding a web test |
| G22 | With no handle on a resource, the test must anchor on a *later* one | 🟠 | Phase 8 unit tests, any reorder |
| G23 | **Core holds a *copy* of the signing secret, so APIM must be deployed first** | 🔴 | Every deploy and every rotation |
| G24 | **Every customer deploy reads portal content from a sandbox in another subscription** | 🔴 | Phase 10, every customer deploy |
| G25 | **A partial read of the source portal would delete customer pages** | 🔴 | Phase 10 implementation |
| G26 | **Microsoft does not support portal content migration between v2 tiers — which is what we run** | 🔴 | Phase 10, today and automated |
| G27 | **A read-only role cannot copy the portal's images, and the obvious role does not either** | 🔴 | Phase 10, first deploy in any environment |
| G28 | **The "has anything changed?" fingerprint is not reproducible across machines, so the portal may be rebuilt on every deploy** | 🔴 | Phase 10, every customer deploy |
| G29 | The dev deploy command offers to register an unknown environment, and saying yes creates a wrong one | 🔴 | Any dev-container deploy |
| G30 | Two deployment identities can read the content master by accident, and production cannot at all | 🔴 | Phase 10, first real deploy |
| G31 | A component the environment has not been told about is skipped silently, and the deploy reports success | 🔴 | Any deploy of a new component |
| G32 | There is no dry-run through the pipeline | 🟠 | Every pipeline run |
| G33 | A branch override beats IaC versioning, but only on internal environments | ✅ | Any spike-branch test run |
| G34 | **Naming one component does not pull its dependencies in — it silently drops them** | 🔴 | Any partial deploy, including every Dreich test run |
| G35 | **Dreich's sign-in record has an out-of-date name, so APIM cannot find it** | 🔴 | Any APIM deploy where `Auth` is stale — this is what actually failed on Dreich |
| G36 | **"Do not delete" is inherited by children, so anything hung off the sign-in record can never be removed** | 🔴 | Any deploy that removes a resource parented to the Auth app registration |
| G37 | **A deleted APIM reserves its own name for 48 hours, and nothing in our code clears it** | 🔴 | Any rebuild of an environment within two days of tearing it down |
| G38 | The one-year expiry we ask for is ignored — Entra issues two years instead | 🟠 | Any `ApplicationPassword` we create |
| G39 | **APIM strips the version segment, Core insists on it, and nothing puts it back** | ✅ | Ticket 3 — routing. Green deploy, every call broken |
| G40 | **"Core" is three different component names, and checking for one of them 403s the whole API** | ✅ | Ticket 4 — signing. Green deploy, every call broken |
| G41 | The environment values are remapped, and the remap is wrong for anything Pulumi did not create | 🟠 | Any lookup of a pre-existing resource |
| G42 | The deploy identity can read variable groups but not change them | ✅ | Superseded by G43 |
| G43 | Core reads App Configuration directly, so the variable group is unnecessary | ✅ | Ticket 4 — the simpler route |
| G44 | **A Key Vault reference is only half the job — the reader needs vault access too** | 🔴 | Ticket 4. The Dreich group is still empty |
| G45 | There are two ways to write an App Configuration setting, and only one is owned by Pulumi | ✅ | Ticket 4 |
| G46 | A test assertion inside a Pulumi Output can pass by never running, and you cannot catch it failing | ✅ | Any component test asserting on an Output |
| G47 | A time filter inside alert KQL is a second look-back window, and config only reaches one of them | ✅ | Ticket 7 — alert tuning |
| G48 | **APIM should be deployed to West Europe, but the code defaults to the hub's region** | 🟠 | Any new environment (added 2026-10-09) |

---

## G1 — The `email` Graph scope is requested but never consented

**Found:** 2026-08-05, first `Spoke/Auth` deploy to `grant-sbx`. **Status:** 🔴

Phase 4B configures the developer portal to request `openid profile email`. The Auth deploy granted
admin consent for exactly two scopes:

```
claimValues: ["openid", "profile"]
```

`email` is missing. So the portal asks for a permission the app registration has not been consented
for, which typically produces a consent prompt or an outright sign-in failure.

This is not a surprise — a January 2026 message in `#project-api-voodoo` listed the app-registration
prerequisites the guide never captured, including **`email`** and **`User.Read`**. Both are absent
from `Spoke/Auth/infra/auth_component.py`, which grants only `openid` and `profile`.

**If portal sign-in misbehaves, look here first.** The fix is small — add the two scope IDs to
`application_msgraph_api_access` — but the Core app registration is shared and security-sensitive, so
it needs sign-off rather than a unilateral change.

---

## G2 — Sign-in silently requires app-role assignment

**Found:** 2026-08-05, same deploy. **Status:** 🔴

The service principal is created with `appRoleAssignmentRequired: true`, meaning **only explicitly
assigned principals can sign in at all**. Two assignments exist: the `All Dayshape Users` group and
the deployment service connection.

The failure mode is a generic Entra error along the lines of *"not assigned to a role for the
application"*, which reads like a broken configuration rather than a deliberate access control. It is
easy to lose an afternoon to this.

**Implication for customers:** every user expected to sign into the developer portal must be in an
assigned group. Whether that is the right group for *external* API consumers is an open design point
— the Dayshape-internal group almost certainly is not.

---

## G3 — The guide's policy XML is not valid XML as printed

**Found:** 2026-08-05, caught by `test_service_policy_is_valid_xml`. **Status:** 🟠

The Phase 3 policy embeds a C# expression to read the caller's `oid` claim. As printed in the setup
guide it contains two characters that are illegal raw inside an XML attribute:

- `&&` — must be `&amp;&amp;`
- the space char literal `' '` — must be `&apos; &apos;` inside a single-quoted attribute

Copy the guide's XML verbatim into IaC and **Azure rejects the entire policy**. The spike escapes
both and has a test that parses the generated XML so this cannot regress.

The wider lesson: the guide's code blocks are **screenshots of a working portal state**, not
validated source. Anything copied out of it needs parsing before it is trusted.

---

## G4 — `<base />` is rejected in the service-level policy

**Status:** ✅ handled, recorded so nobody "fixes" it back

`<base />` inherits policy from a parent scope, but the service-level policy **is** the top of the
scope chain. Including it produces a *"not allowed in global context"* error. It is only valid in
API- and operation-level policies.

The spike's policy deliberately omits it, and the omission is commented in
`Spoke/ApimSpike/infra/apim_spike_helpers.py` so it reads as intentional rather than missing.

---

## G5 — Implicit flow is a deprecated dependency, not a choice

**Status:** 🔴 — strategic, not immediate

The guide instructs setting the authorization grant type to **Implicit**, and both live instances use
it. The reason is only recorded in Slack (`#project-api`, 16 December 2025): Core validates an
**ID token**, whereas APIM and modern OAuth are built around **access tokens**. Authorization-code
flow returns an access token, which Core rejects; implicit returns an ID token, which it accepts.

Implicit flow is **legacy and deprecated**. We are matching the guide deliberately, but this is a
known dead end — the stated ambition in that thread is an API auth mechanism we can stand behind for
~5 years, with **client-credentials flow** (app-to-app) named as the likely direction.

**Do not treat "it works" as "it is finished."** This is a scheduled migration, not a solved problem.

---

## G6 — Running Auth without the flag strips the portal's redirect URIs

**Status:** 🟠 mitigated by the flag being set per-customer

`azuread.ApplicationRedirectUris` is **authoritative** for the type it manages. Auth now owns both the
SPA and Web lists, and only adds the developer-portal URIs when `apim_enabled` is true.

Consequence: if anyone deploys Auth for an APIM-enabled customer from an environment where that flag
has not resolved to true, **the portal's redirect URIs are deleted** and portal sign-in breaks until
the next correct run.

The flag lives in App Config at the `{clientid}-{env}` label, so it does persist — but it is set by
hand, not declared in version control. Making it declarative is a tracked follow-up on T3.

Related: the trailing-slash bug recorded in [[APIM IaC - ticket breakdown]] is the same class of
problem — two owners disagreeing about one character was enough to break customer authentication.

> [!note] This is now the *only* thing Auth still holds for APIM — 2026-08-13
> The portal's sign-in secret used to be created by Auth too. It has moved into the APIM component
> (see [[D1 - The developer portal sign-in secret]]), which removed a second, larger version of this
> same failure mode: an Auth deploy would mint a fresh secret and leave sign-in broken until APIM was
> next deployed.
>
> The redirect URIs deliberately **did not** move, because `azuread.ApplicationRedirectUris` really is
> authoritative and the SPA list holds Core's own address alongside the portal's. That list needs a
> single owner.
>
> The practical effect is that this trap is now more prominent rather than less: it is the last
> remaining way an Auth deployment can break portal sign-in, so making the flag declarative is worth
> more than it was.

---

## G7 — The 4B/4C client secret expires after 12 months with no owner

**Status:** ✅ **RESOLVED 2026-08-13** — the secret now renews itself. Kept as a record.

Phases 4B and 4C both need a client secret belonging to the Core app registration. The spike
originally read it from an environment variable and expected a human to mint it in the portal —
scaffolding with no production route.

The secret is **per-customer** and expires after **12 months**. Nothing owned creating or rotating
it. Left as-is, authentication would silently start failing roughly a year after go-live, one
customer at a time, with no warning.

### How it was resolved

**The APIM component creates the secret itself and replaces it every 30 days.** See
[[D1 - The developer portal sign-in secret]].

The agreed direction was originally for `Spoke/Auth` to mint it and file it in Key Vault, since Auth
owns the app registration. That was built on 2026-08-12 and then **revised on 2026-08-13**: minting
it in Auth meant every Auth deployment broke portal sign-in until APIM was next deployed. Creating it
in APIM removes the handover entirely and makes automatic renewal trivial, because
`azuread.ApplicationPassword` is not authoritative over the credential list and several components
already add credentials to app registrations they do not own.

> [!warning] ⚠️ One residual risk, deliberately accepted
> Renewal happens **during an APIM deployment**. An environment left undeployed for more than 12
> months still drifts and its portal sign-in will stop working. Customers receive 5–6 deploys a year
> so this does not affect them; the exposure is long-dormant internal and trial environments.

**Note on the earlier "worth checking first":** the suggestion that 4B might not need a secret at all
(implicit flow exists for clients that cannot hold one) was not pursued. Both resources take the
same value, and since it is now created automatically there is nothing to be saved by halving it.

---

## G8 — The setup guide has confirmed omissions

**Status:** 🟠 — treat the guide as a solid draft, not a specification

Two confirmed gaps, both from `#project-api-voodoo`:

- **January 2026** — a list of required app-registration settings (ID tokens, multitenant, Graph
  `email` and `User.Read`) that does not appear in the guide. See G1.
- **April 2026** — the guide's author on the username/password removal step (our Phase 4D):
  *"This was only a mistake that I forgot to mention in the setup guide."*

The guide remains the best source we have and the spike follows it deliberately. But where it is
silent, **assume something is missing rather than assuming nothing is needed** — and check Slack
before concluding the guide is complete on a given point.

---

## G9 — APIM cannot use its own app registration, because the ID token audience is fixed

**Found:** 2026-08-05, while evaluating whether APIM should own its identity. **Status:** ✅
**CONFIRMED 2026-08-13** — from Core's own source, not from a live token

The obvious tidy design is for APIM to create and own its own app registration, exactly as
`Spoke/AiAgent` does for its bot. It would satisfy both house rules at once: one owner per app
registration, and mint-the-secret-where-you-use-it. It is also Richard Donkin's Option 2.

**It does not work.** An OIDC **ID token's `aud` claim is the client ID of the
application that requested it** — that is fixed by the protocol, not configurable per request. Core
validates an **ID token** rather than an access token (see G5). So a developer portal signing users in
against an APIM-owned registration produces a token whose `aud` is APIM's client ID, and Core rejects
it.

This is very likely **why the setup guide configures the Core app registration** rather than creating a
dedicated one. It reads like an arbitrary choice; it is probably a constraint.

> [!success] Confirmed from source — 2026-08-13
> The suggested ten-minute token check below was never needed. Core's own code settles it more
> firmly than a single token would:
>
> - `AuthenticationExtensions.cs` sets `ValidateAudience = true` with
>   `ValidAudiences = { ClientId, api://{ClientId} }` — **Core accepts only its own client ID.**
> - The obvious workaround is closed too: `IdpConfigurationProvider` has exactly **two** hard-coded
>   identity-provider slots, and both are already used by real customer sign-in providers. There is
>   no third slot for APIM without a Core code change.
>
> **Note on attribution.** [[D1 - The developer portal sign-in secret]] originally credited this
> constraint to G9. The more precise framing is that **G5** is the root cause — Core validates ID
> tokens rather than access tokens — and G9 is the consequence for APIM specifically. Same outcome,
> but G5 is where the fix would have to happen.

**Consequences:**
- Sharing Core's app registration is not a preference we can revisit for tidiness — it is forced.
- Any "let APIM own its identity" refactor is blocked until Core accepts additional audiences, which
  is a Core change, not an IaC one.
- The secret-ownership question this created is now answered — APIM adds its own credential to Core's
  registration, which is permitted because passwords are not authoritative. See
  [[D1 - The developer portal sign-in secret]].

~~**How to confirm — ten minutes, no code.** Run "Try it" on the hand-built `vprk-sbx-spk-apim` developer
portal, open the HTTP request preview, and decode the bearer token.~~ *No longer needed — confirmed
from source above.*

---

## G10 — APIM child resources need an explicit name, or Azure rejects the deploy

**Status:** ✅ fixed and **proven on a real deploy** (grant-sbx, 2026-08-05 — the identity provider
was created successfully, which also confirmed the client secret was valid).

Every APIM child resource is addressed by a name that becomes the last segment of the URL Azure
writes to. Pulumi exposes that as a separate argument — `authsid` on an authorization server,
`identity_provider_name` on an identity provider, `api_id` on an API, `product_id` on a product.

If the argument is left out, Pulumi does not fail. It quietly falls back to the resource's own
logical name, which for us is prefixed and unique per customer. Azure then rejects it.

The identity provider is the sharpest version of this, because Azure only accepts six fixed names —
`aad`, `aadB2C`, `google`, `facebook`, `twitter`, `microsoft`. The name *is* the type. A first
deploy failed with:

```
Status=400 Code="ValidationError"
Message="Unknown identity provider type: grant-sbx-spk-apimgs-apim-entra-idp"
```

The give-away in the Pulumi diff is the two properties disagreeing:

```
identityProviderName: "grant-sbx-spk-apimgs-apim-entra-idp"   <- fell back to the resource name
type                : "aad"                                    <- what we actually set
```

**What to do.** Set the name argument explicitly on every APIM child resource. Assume it is required
even where the SDK marks it optional. This matters most for Phases 6 and 7, which add products and
APIs and have the same shape of argument.

**Guard.** `test_entra_identity_provider_created` asserts the resolved name is `aad`. This needed a
matching entry in the shared Pulumi mock (`Modules/component_unit_test.py`), which already maps the
URL-name input onto `name` for virtual machines, resource groups and private endpoints — the
identity provider now joins them. Without that entry the mock returns `None` and the assertion
cannot see the value.

---

## G11 — The guide's Phase 4D uses a portal API that the v2 tiers reject

**Status:** ✅ fixed and **proven on a real deploy** (grant-sbx, 2026-08-05 — clean run, 18s, the
script reported `developer portal username-and-password identity disabled`). This also confirms
`portalconfigs` is reachable on BasicV2, which until that run was expected rather than known.

APIM has **two developer portals**, and they have separate ARM APIs:

| Portal | ARM path | Available on |
| --- | --- | --- |
| Legacy | `portalsettings/signup` | Classic tiers only |
| Current | `portalconfigs/default` | All tiers, and the only portal the v2 tiers run |

Phase 4D asks us to delete the "Username and password" row from the portal's Identities blade. The
obvious ARM equivalent is `portalsettings/signup`, and on BasicV2 that fails:

```
Status=400 Code="MethodNotAllowedInPricingTier"
Message="Method not allowed in BasicV2 pricing tier"
```

The correct property on the current portal is `enableBasicAuth` on the portal config — Azure's own
name for the username-and-password identity.

```
PATCH .../service/{name}/portalconfigs/default?api-version=2024-05-01
If-Match: *
{"properties": {"enableBasicAuth": false}}
```

**Use PATCH, not PUT.** PUT on this entity is a create-or-update that replaces the whole
configuration, so it would silently wipe the portal's CORS, CSP, delegation and sign-up settings.
`test_the_whole_portal_config_is_not_replaced` guards this.

**Wider point now that Basic v2 is the agreed tier.** This is the second v2-tier behaviour difference
the guide does not mention, and it will not be the last — the guide was written against a classic
tier, so every step needs checking against the v2 tiers rather than trusted. Only four of ten phases
are built; expect more of these in Phases 5–10. Treat "the guide says do X in the portal" as a
starting point, not a specification. The tier decision itself is recorded as **Q4** in
[[APIM IaC - outstanding questions]].

---

## G12 — The Azure SDK rejects the success code Azure actually sends

**Status:** ✅ fixed and **proven on a real deploy** (grant-sbx, 2026-08-06 — clean run, 14s, the
provider reported `Set developer portal username-and-password sign-in to False`). Found on the first
real deploy of the dynamic provider, earlier the same day.

Phase 4D failed with:

```
Error creating ApimPortalSigninProvider: Operation returned an invalid status 'No Content'
```

Nothing was wrong with the request. **Azure applied the change and answered `204 No Content`**, which
is a success. The `azure-mgmt-apimanagement` SDK reported it as a failure anyway:

```python
# azure/mgmt/apimanagement/operations/_portal_config_operations.py, update()
if response.status_code not in [200]:
    map_error(...)
    raise HttpResponseError(...)
```

**Why it happens.** Microsoft auto-generates these SDKs from a machine-readable description of each
Azure API. That description lists `200` as the only success code for this operation. The live
service sends `204`. The generated code inherited the mismatch, so a successful write is reported as
an error. This is a Microsoft defect, not a fault in how we are calling it.

**Why the old script never hit it.** The deleted `apim_portal_settings.py` used `requests` and
checked the status itself, with a comment that already recorded the real behaviour:

```python
# 200 = updated, 204 = updated with no body returned.
if response.status_code in (200, 204):
```

`requests.raise_for_status()` also treats `204` as success. Moving to the official SDK — which the
repo's IaC rules require — is what exposed the SDK's stricter behaviour.

**The fix.** `set_basic_auth_enabled()` catches `HttpResponseError` and treats `200` and `204` as
success, re-raising everything else. Guarded by
`test_set_basic_auth_enabled_accepts_a_no_content_answer_as_success` and
`test_set_basic_auth_enabled_raises_when_the_write_genuinely_fails`.

**Wider point.** Expect this class of problem again in Phases 5–10. A generated SDK is only as
accurate as the description it was built from, so where one disagrees with the live service, trust
the service. Two practical habits: when an SDK call fails, check the actual status code before
assuming the request was wrong; and when replacing raw REST with an SDK, check whether the REST code
tolerated status codes the SDK will not.

---

## G13 — APIM may read the vault before its group membership has caught up

**Where it bites.** The first Phase 5 deploy, when the named value is created.

**What happens.** Phase 5 does three things in one deploy: it gives APIM its own identity, it puts
that identity into the security group that can read the vault, and it creates the named value that
points at the vault secret. APIM checks it can actually read the secret at the moment the named
value is created.

Azure does not apply group membership everywhere instantly. There is usually a short lag between
"APIM has been added to the group" and "Azure agrees APIM is in the group". If the named value is
created inside that window, APIM is told it has no access, and the named value is created carrying
an error rather than the secret.

**How to tell.** The named value exists in the portal but shows a Key Vault error along the lines of
"the client does not have permission to perform action Microsoft.KeyVault/vaults/secrets/getSecret".
The deploy itself may still report success, which is what makes this worth guarding against — the
signing secret is quietly missing and nothing says so.

**The fix.** A three-minute wait sits between granting APIM access and creating the named value, so
the grant has applied before APIM ever reads the vault. This is the pattern the repo already uses —
`Modules/AgwCertificate` waits the same 180 seconds after granting the same Key Vault Secrets User
role on the same vault, and `ContainerAppsEnv` and `SqlDb` do the same for their own grants.

The wait only applies when the resource is first created, so it does not slow down later
deployments.

**Why a fixed wait rather than checking.** APIM does record whether its last read worked, and it can
be told to try again, so a check is possible. It was built and then thrown away: roughly four
hundred lines against eight, to solve a problem three other components already solve with a sleep.
Consistency and less code to maintain won. If the race ever does slip past 180 seconds, the check is
in the branch history.

**Confirmed working.** First deploy of Phase 5 to `grant-sbx` on 6 Aug 2026 created the group
membership and the named value in the same run — the exact race this guards against — and APIM
reported `lastStatus.code: "Success"` on the named value, with `provisioningState: "Succeeded"`.
Total deploy 3m46s, of which 3m was the wait.

**Two things this is not.** Checked against Microsoft's named values documentation:

- **The vault firewall is not in play.** Microsoft requires "allow trusted Microsoft services" to be
  switched on where the vault firewall is enabled. The spoke vault is created with
  `public_network_access="Enabled"` and no network rules, so there is nothing to allow.
- **The permission model is right.** The vault is created with `enable_rbac_authorization=True`, and
  Microsoft's RBAC route asks for the Key Vault Secrets User role, which is what Phase 5 grants.
  Secrets User covers both get and list, which is what the documentation says the identity needs.

Also confirmed against the same documentation: the identity must be **system-assigned** if the vault
firewall is ever switched on, the secret URI must carry **no version** or rotation stops working,
and the secret must be **1–4096 characters**. Phase 5 satisfies all three.

---

## G14 — APIM resources return far fewer outputs than you pass in

**Found:** Phase 6, writing unit tests for the Default Product.

Several `azure-native` APIM resources accept an identifier as an **input** but never expose it as an
**output**. Concretely:

| Resource | Has no output for |
| --- | --- |
| `apimanagement.Product` | `product_id` |
| `apimanagement.ProductGroup` | `product_id`, `group_id` |

This matters because the shared test mock in `Modules/component_unit_test.py` builds its outputs by
converting whatever inputs you passed. The values *are* present in the mock — but a Python
`azure-native` resource object only surfaces properties it **declares**, so reading
`sut.default_product.product_id` simply does not exist to be read.

**Don't** fix this by adding to the hardcoded special-case list inside `component_unit_test.py`.
That is shared test infrastructure used by every component in the repo, and there is already
precedent in this spike for how much friction editing shared code causes.

**Do** use the per-test hook, which is the established repo idiom (15+ test files, template at
`Modules/VirtualMachine/tests/virtual_machine_dcr_component_unit_test.py`):

```python
captured = [{}]
self.add_pulumi_new_resource_mock(
    "azure-native:apimanagement:Product",
    lambda outputs: captured.__setitem__(0, outputs),
)
sut = self._get_sut()
self.assertPulumiOutputCustom(
    sut.default_product.id,
    lambda _: self.assertEqual(captured[0].get("product_id"), "default-product", "..."),
)
```

The `.id` output is the async trigger — by the time it resolves, the callback has already run.

> [!warning] ⚠️ Always prove the capture isn't vacuous
> `{}.get("product_id")` returns `None`. If the callback never fires, an assertion expecting `None`
> passes silently and the test proves nothing. **Deliberately break the expected value once**,
> confirm you get a real mismatch, then revert. This was done for all four Phase 6 tests.

> [!note] 🤔 A real deploy behaves differently
> Azure genuinely *does* return `name: "default-product"` on a real `Product` deploy. The mock
> doesn't, because we never pass `name`. Don't write a test against `.name` on the strength of
> having seen it in deploy output — it will fail under test and the reason won't be obvious.

**Confirmed again in Phase 7** — `apimanagement.Api`, `ApiVersionSet` and `ProductApi` are all the
same shape, and the Phase 7 tests use the same capture idiom. Phase 7 also turned up a second-order
version of this problem — see **G19**.

---

## G19 — A capture-based assertion must hang off the resource it is reading

**Status:** ✅ handled, kept for the record · **Bites during:** any phase's unit tests
**Found:** Phase 7, writing the `ProductApi` test.

This is the trap *inside* G14. Once you know to capture inputs with `add_pulumi_new_resource_mock()`,
the natural next step is to register several captures and assert them all from one convenient
`assertPulumiOutputCustom()` block. **That does not work**, and it fails silently.

The registered mocks fire in the order Pulumi constructs the resources. `assertPulumiOutputCustom`
runs its lambda as soon as the output you gave it resolves — which for an earlier resource is
*before* a later resource has been constructed at all. Its capture dict is still `{}`.

```python
# ❌ Wrong — the ProductApi mock has not fired yet when this lambda runs
self.assertPulumiOutputCustom(
    sut.entities_api.id,
    lambda _: (
        self.assertEqual(api_captured[0].get("path"), "entities", "..."),
        self.assertEqual(product_api_captured[0].get("product_id"), "default-product", "..."),
    ),
)
```

```python
# ✅ Right — one assertion block per resource, keyed on that resource's own output
self.assertPulumiOutputCustom(
    sut.entities_api.id,
    lambda _: self.assertEqual(api_captured[0].get("path"), "entities", "..."),
)
self.assertPulumiOutputCustom(
    sut.default_product_entities_api.id,
    lambda _: self.assertEqual(product_api_captured[0].get("product_id"), "default-product", "..."),
)
```

**Why it's dangerous rather than merely annoying:** combined with the vacuous-capture problem in
G14, an assertion reading an empty dict returns `None`. If you happen to be asserting against a
falsy value, or you write the test expecting `None`, it passes and proves nothing.

> [!tip] The rule
> **One `assertPulumiOutputCustom` block per captured resource, triggered by that same resource's
> `.id`.** Never read resource B's capture from resource A's callback — and still break each expected
> value once to prove the callback fires.

---

## G20 — Single quotes inside a single-quoted policy attribute break the XML

**Status:** ✅ handled, kept for the record · **Bites during:** any policy XML work
**Found:** Phase 7, building the Entities API policy.

The spike writes policy XML with **single-quoted attributes** (a Phase 3 convention — it avoids
escaping the double quotes that C# string literals use constantly). The HMAC signing block is a C#
expression that contains character literals:

```csharp
Convert.ToBase64String(hash).TrimEnd('=').Replace('+', '-').Replace('/', '_')
```

Dropped into `value='...'`, the first `'` in `'='` **terminates the attribute**, and the deploy fails
with a parse error pointing at a column in the middle of what looks like ordinary code. The original
guide fragment used double-quoted attributes with `&quot;` escapes, so the problem doesn't appear
until you convert to the spike's convention.

**Fix:** escape them as `&apos;` — the same escaping family as the `&apos; &apos;` already used in
the Phase 3 service policy.

```python
".TrimEnd(&apos;=&apos;).Replace(&apos;+&apos;, &apos;-&apos;)"
```

> [!tip] Cheap insurance
> Every `build_*_policy_xml()` function has a unit test that runs `ElementTree.fromstring()` on its
> output. That test catches this class of bug in under a second, instead of after a multi-minute
> deploy. **Any new policy builder should get one.**

---

## G21 — `create_availability_web_test()` returns nothing

**Status:** ✅ handled, kept for the record · **Bites during:** any component adding a web test
**Found:** Phase 8, wiring the availability monitor.

`Modules/AppInsights/infra/app_insights_interface.py` builds **two** resources — the `WebTest` and a
`LogSearchAlert` to go with it — and then **has no `return` statement**. Both locals are silenced:

```python
webtest = az.applicationinsights.WebTest(...)   # noqa: F841
ping_test_alert = ...                            # noqa: F841
# ← no return
```

The `# noqa: F841` is doing real damage here: it tells the linter the unused variables are
deliberate, so nothing complains, and the missing return is invisible.

Every instinct says to write `self.webtest = create_availability_web_test(...)`. That compiles, runs,
deploys correctly — and silently binds `None`. The Phase 8 unit test failed with
`AssertionError: unexpectedly None`, which reads like the web test wasn't created when in fact it was
created perfectly.

**Fix:** call it as a statement, and don't keep the result.

```python
# ✅ It registers its own resources. There is nothing to hold on to.
create_availability_web_test(...)
```

> [!warning] Two more things buried in the same helper
> - It **overrides `pulumi_opts.provider`** with its own `appins-webtest-provider`. The in-source
>   warning says *"you can only create a single one of these per component as the
>   appins-webtest-provider will conflict."* **One web test per component, maximum.**
> - The `WebTest` and its alert land in the **hub** resource group (where App Insights lives), not
>   the spoke's. Confirmed on the Phase 8 deploy: both went to `sre-ds-uks-lwr-sh-rg`. Names are
>   client-prefixed so there's no collision, but at 100+ customers every availability test and alert
>   accumulates in one shared hub RG.

---

## G22 — With no handle on a resource, the test must anchor on a *later* one

**Status:** 🟠 mitigated in the spike, needs a real fix · **Bites during:** Phase 8 unit tests, any reorder
**Found:** Phase 8, testing the availability monitor. **Direct consequence of G21.**

G19 gives the rule: *one assertion block per captured resource, keyed on that resource's own output.*
G21 breaks the rule's precondition — when a helper returns nothing, **there is no `.id` to key on**.

The workaround is to anchor on the `.id` of a resource created *after* the one you're asserting, so
the capture is guaranteed populated by the time the callback runs. Phase 8 anchors the `WebTest`
assertions on `sut.diagnostic_setting.id`:

```python
# The WebTest gives us no handle (G21), so we hang off the next resource created after it.
self.assertPulumiOutputCustom(
    sut.diagnostic_setting.id,
    lambda _: self.assertEqual(
        webtest_captured[0].get("request", {}).get("request_url"),
        f"https://{name}.azure-api.net/status-0123456789abcdef",
        "the ping should target APIM's built-in unauthenticated health endpoint",
    ),
)
```

**This creates a hidden coupling.** `__configure_diagnostics()` now *must* create the web test before
calling `logging_interface()`. Swap those two lines — a harmless-looking tidy-up — and the test fails
with an empty capture and a baffling `None != 'https://...'`. Nothing in the component hints that a
test depends on the ordering.

**Mitigations applied:**
- A comment in `__configure_diagnostics()` marks the ordering as load-bearing.
- The assertion was **mutation-tested** — the expected URL was deliberately broken and the test
  failed as intended, proving the callback fires against a populated capture rather than passing
  vacuously (the G14/G19 failure mode).

> [!tip] The real fix, when someone has an afternoon
> Add a `return webtest` to `create_availability_web_test()`. It's a one-line change to a shared
> module, it's backwards-compatible (every existing caller ignores the result), and it retires both
> G21 and G22. Until then, **anchor forward and mutation-test the assertion.**

---

## G15 — Azure holds the APIM name for hours after deletion

**Status:** 🔴 will bite as-is · **Bites during:** any destroy-and-recreate

Raised by Marcus on 2026-08-07, and it's the constraint that shapes how we migrate the existing
customer off the manual setup.

**When you delete an APIM instance, Azure does not immediately release its name.** It holds it for
a period — Marcus's experience was **a few hours**. If you try to create a new instance with the
same name inside that window, the deploy fails on a name collision.

### Why it matters

The agreed migration approach is **destroy and rebuild** — Sam: *"just destroy it and rebuild
it."* Importing the existing resources into Pulumi is possible but *"not a straight import"*, and
Richard called it *"pretty painful and manual."* With one customer across three environments, a
rebuild is the cheaper option.

But a rebuild means deleting the instance and creating it again **with the same name**, which is
exactly the case this trap catches.

### What to do

- **Test it before it goes anywhere near a customer.** We have never done a destroy-and-recreate
  with name reuse. This must be proven on the sandbox first.
- **Plan the window.** Marcus's steer: kill it, leave a gap of a few hours, then deploy. Do it in
  the **afternoon** so the gap doesn't eat a working day.
- **Don't assume it's instant just because the delete returned successfully.** The resource
  disappearing from the portal is not the same as the name being free.

> [!note] Auto-destroy was considered and effectively rejected
> The idea of having the IaC tear down the old instance itself came up. Richard pointed out
> Pulumi won't touch resources it doesn't know about — and the manually-created APIM isn't in
> state — so an auto-destroy step would simply do nothing.
>
> Pulumi's **deployment hooks** (before/after, able to branch on preview / up / destroy) do exist
> and AI Agent uses them, so there's precedent. But they only act on Pulumi-managed resources.
> **Treat as a stretch goal; don't spend time on it.**

---

## G16 — The setup guide is AI-generated, not hand-written

**Status:** 🟠 mitigated by knowing · **Bites during:** any phase

Sam Clifford volunteered this on 2026-08-07, unprompted:

> *"I am 99% sure that they built it from the IaC. So they built the IaC with AI, then they
> pointed AI at the IaC and said 'build me a doc', and then they handed it to us. The only human
> element is modifications me and Marcus have made to it. So you need to assume it's not
> something you should take verbatim. It's a starting point, but I wouldn't use that as the
> decision token for how you phase the work."*

So the [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide) is a **machine's description of VP's old IaC**, lightly edited. It is
not a runbook someone wrote from experience, and it has never been fully executed end to end.

### What it explains

Every inconsistency we've hit so far, retrospectively:

- **Two different OpenAPI spec paths** given in different phases (Phase 7 vs Phase 12)
- **Phases ordered as though they have dependencies that don't exist** — Phase 6 needs nothing
  from Phase 5, for instance
- **Steps Marcus says have never actually been run**
- **Phase 2's default-resource cleanup**, which logs "not found, skipping" on BasicV2 — quite
  possibly describing behaviour of a tier we aren't using

### What to do

- Treat concrete **values** as reasonable defaults — copy them unless there's a reason not to.
- Treat **structure, ordering and dependencies** as suggestions, not requirements.
- When something doesn't match reality, **"the guide is wrong" is now the first hypothesis**,
  not the last.

Working rule 5 has been amended accordingly —
[[APIM IaC - working rules for implementation]].

---

## G17 — Secrets stored in APIM are not obfuscated in the portal

**Status:** 🔴 will bite as-is · **Bites during:** rollout / access review

Spotted by Sam on 2026-08-07 while looking at the spike instance. **A secret held in APIM is
readable in plain text by anyone with portal access to the instance.**

This is not something we've done wrong — it's how the resource behaves. But it changes the
threat model, because "stored in APIM" is not equivalent to "stored in Key Vault" from a
who-can-see-it perspective.

### Why it matters

- The Phase 5 HMAC secret and the 4B/4C client secret both end up visible to anyone who can open
  the APIM blade.
- Today that's a wider group than it should be.

### What to do

- **Consider a read-only developer group** for APIM, so developers can look without seeing
  secrets. Not designed yet.
- Feeds the **management-plane half** of the auth question Sam raised — see
  [[APIM IaC - client secret decision]].
- Phase 5's Key Vault-backed Named Value is still the right pattern; this trap is about *who can
  read the resolved value*, not where it's stored.

---

## G18 — APIM keeps serving while Core is mid-deploy

**Status:** 🔴 will bite as-is · **Bites during:** every customer upgrade

Raised by Sam and Richard on 2026-08-07. During an upgrade there is **a window where APIM and the
Core app it fronts are on different versions**. Sam:

> *"There will be a moment in time where APIM is newer or older than the core app it's talking
> to. So any integrations interacting with APIM might fail during that window when otherwise
> they wouldn't."*

### Why it's worse than the equivalent in the app

Core has a **maintenance mode**, and the VM can be disabled — so a person hitting the app during
a deploy gets a clear "we're down" page and retries later.

**APIM has no equivalent.** It carries on accepting requests and forwarding them to a backend
that may be restarting, half-upgraded, or serving a different API shape. A customer's automated
integration script gets an assortment of 404s and timeouts instead of one unambiguous signal.

### What to do

**Nothing yet — this is unsolved and deliberately out of Phase 7's scope.** Options discussed but
not decided:

- Return a meaningful **`503`** from APIM during a deploy, so integrations have something to back
  off on *(Sam)*
- A **maintenance mode for the API**, mirroring Core's *(Richard)*
- **Versioning headers** and backwards compatibility, as Reporting does — *Sam: probably out of
  scope, needs changes to the Core APIs themselves*
- Settling the **deploy sequence** — Core first or APIM first. **Left unresolved.**

> [!warning] Related, and already real
> Richard mentioned a recent incident where **cache headers** left a customer believing Stage was
> down for a day, after the actual outage had ended. Being fixed separately, but it's the same
> class of problem: the gateway telling callers something the app no longer means.

Full context in [[APIM IaC - OpenAPI source of truth (phase 7)]] §7.6.

---

## G23 — Core holds a *copy* of the signing secret, so APIM must be deployed first

**Found:** 2026-08-10, while correcting Phase 5's secret delivery. **Status:** 🔴 will bite as-is ·
**Bites during:** every deploy of a new environment, and every rotation

This is the concrete half of G18's unresolved *"Core first or APIM first"* question. For the signing
secret at least, **the order is now settled: APIM first, then Core.**

### Why the order matters

The two sides of the signature hold the secret in **different ways**, and only one of them follows
the source:

| Side | How it holds the secret | Picks up a new value |
| --- | --- | --- |
| APIM | A named value **pointing at** the Key Vault entry | Automatically, no redeploy |
| Dayshape (Core) | A **literal copy**, written into `Dayshape.config` on the VM's disk | Only on its next deploy |

Core runs on a VM as the legacy IIS application. It reads plain-text settings from a file, and it
does **not** resolve Key Vault references. So a copy on that side is unavoidable — the value is
taken from the client's `_Core` Azure DevOps variable group when Core deploys, and then frozen until
Core deploys again.

### What goes wrong if the order is reversed

`RequireApimOrigin` is already `true` in the shared `Dayshape.config` for **every** customer, and the
`RequireApimOriginAttribute` filter **fails closed**. So Core with a missing or stale secret does not
degrade gracefully — it rejects **every** request APIM forwards.

Two ways to hit it:

1. **New environment, Core deployed first.** APIM has not run, so the variable group has no
   `ApimHmacSecret`. Core comes up expecting a signature it has no way to verify, and 403s
   everything the moment APIM starts sending traffic.
2. **Rotation.** Regenerating the secret updates the vault, and APIM starts signing with the new
   value immediately. Core carries on checking against the old copy until it is redeployed — so
   there is a **403 window lasting until the next Core deploy**, which could be days.

### What to do

- **Deploy APIM before Core** in any environment build-out, so the variable group is populated before
  Core takes its copy.
- **Treat a rotation as requiring a Core redeploy**, and sequence it accordingly — or do not rotate
  outside a release window.
- Do not assume a failed write is harmless. The Phase 5 hook raises rather than warns for exactly
  this reason: a silently missing secret looks like a successful deployment and then 403s everything.

> [!note] This does not make G18 solved
> G18 is the wider version-skew problem — APIM serving while Core is mid-upgrade. G23 only settles
> the **ordering for the secret**. The 503 / maintenance-mode discussion is still open.

---

## G24 — Every customer deploy reads portal content from a sandbox in another subscription

**Found:** 2026-08-10, Phase 10 planning. **Status:** 🔴

The developer portal's static pages and styling are not authored in this repository. They are
designed by hand in the browser on one nominated APIM instance, and copied from there onto every
customer's instance. That instance is `vprk2-sbx-spk-apim`, in resource group
`vprk2-sbx-spk-apim-rg`, in the Dayshape sandbox subscription
`58d847ff-52c8-4127-b663-1232b8fbbd48`.

This is deliberate, not an accident. The portal is edited through a drag-and-drop visual editor, so
there is no file format to author in — the instance *is* the design tool. Laszlo Szegedi of Voodoo
Park confirmed the intended workflow in `#project-api-voodoo` on 17 June 2026: *"if anyone wants to
make any changes to the styling or static content, they need to do it on the vprk2 apim instance and
then run the content copy scripts"*.

**It is also in active use by Dayshape staff.** On 14 July 2026 Jonathan Lozinski asked for edit
rights on that instance for Sarah Dimmock and Sinead Donnelly, our technical authors, *"so they can
edit the pages before they're copied/deployed to the customer sites"*. So the customer-facing API
documentation is written there, by the tech authoring team, as an ongoing activity. Two things follow:

- Phase 10 is not a one-off bootstrap that copies a fixed design. It is the **publishing pipeline for
  customer-facing documentation**, and the content behind it changes whenever the authors change it.
- The authors are stakeholders in any change to how this works. Moving the content anywhere else
  changes their day-to-day workflow, and they would need somewhere visual to write in regardless.

Access to the editor is worth noting too: portal users are created in Entra on first sign-in, and the
admin editor is reached through the "Developer portal" button in the Azure portal rather than the
public site address.

Automating Phase 10 turns that arrangement into a hard deploy-time dependency, and three consequences
follow that nobody has had to think about while the step was manual:

- **Cross-subscription read.** Whatever identity runs a customer's deploy must be able to read that
  sandbox. Customer deploys otherwise run entirely within the customer's own subscription, so this is
  the only place that assumption breaks.
- **A sandbox becomes a shared production dependency.** If it is deleted, renamed, moved or simply
  unreachable, every customer deploy that includes portal content fails. Nothing about the instance's
  name or resource group signals that it matters, and the team that built it no longer works with us.
- **There is no record of what shipped.** The desired state is whatever is live on that instance at
  the moment of the deploy. Two customers deployed a week apart can receive different portals with no
  way to tell after the fact, and no way to put a customer back to a previous version.

**What to do about it:** the dependency is acceptable, but it has to be visible. Name the sandbox
explicitly in the implementation documentation and in the component's configuration rather than
burying it in code, so anyone tidying up sandboxes can see what depends on it. The name gives no clue
on its own — when Jonathan asked about editor access in July, even Voodoo Park's own engineer went to
the wrong instance and had to be corrected, so expecting anyone at Dayshape to infer it is optimistic.
Consider archiving each captured snapshot to the customer's own storage account at deploy time — that
is far cheaper than putting the content under version control and buys back the ability to answer
"what did this customer actually get?".

---

## G25 — A partial read of the source portal would delete customer pages

**Found:** 2026-08-10, Phase 10 planning. **Status:** 🔴

The natural way to automate Phase 10 is to read the pages from the source instance and declare one
Pulumi resource per page on the target. Pulumi then works out the difference itself, which is both
safer and quieter than the guide's wipe-and-reimport.

That property cuts both ways. Pulumi treats *absent from the program* as *delete from the target*.
So if the read of the source instance ever returns fewer pages than really exist — a throttled
request, a dropped page of results, a transient failure swallowed somewhere — Pulumi will conclude
those pages were deliberately removed and delete them from the customer's portal. The deploy reports
success while the customer's site quietly loses content.

The content item listing is paged, so this is not hypothetical: any bug in following the paging links
produces exactly this shape of failure.

**What to do about it:** never let an empty or suspiciously small read reach the resource
declarations. Fail the deployment instead. A source portal with zero content items is always a fault,
never a legitimate instruction to erase the customer's portal, and it should be treated that way
explicitly rather than left to Pulumi's normal semantics.

---

---

## G26 — Microsoft does not support portal content migration between v2 tiers, and that is exactly what we do

Microsoft's own guidance on automating developer portal deployments carries this note:

> *"Using the script to migrate developer portal content between an API Management instance in a
> classic tier (for example, Standard) and an instance in a v2 tier (for example, Standard v2)
> isn't currently supported. Migration of portal content between instances in the v2 tiers is
> also not supported."*
>
> — [Automate developer portal deployments](https://learn.microsoft.com/en-us/azure/api-management/automate-portal-deployments),
> Microsoft Learn, updated 25 June 2026

That page also carries the banner **"APPLIES TO: Developer | Basic | Standard | Premium"** — the four
*classic* tiers. `BasicV2` is not among them, so the page does not claim to cover our configuration
at all, which reinforces the note rather than softening it.

Both ends of our copy are v2:

| Instance | Tier |
| --- | --- |
| `vprk2-sbx-spk-apim` (the master everything is copied from) | **BasicV2** |
| Every customer instance we deploy | **BasicV2** (`spk-apimgs-infra-apim_sku_name`) |

So the second sentence applies directly. Phase 10 of the setup guide, as written and as currently
performed by hand for live customers, sits on a path Microsoft documents as unsupported.

**It does work.** Manmeet Kaur Bharaj ran the clone against `laldie` on 7 July 2026 and confirmed
success in `#project-api-voodoo`, and APIM is live for customers today. Unsupported is not the same
as broken.

What it does mean:

- If it breaks, Microsoft will not help, and "it used to work" carries no weight.
- It can stop working after any APIM platform update, with no notice and no deprecation window.
- Automating it multiplies the exposure. Today a person runs it occasionally and eyeballs the result.
  Automated, it runs unattended for every customer, so a platform-side change would land everywhere
  before anyone noticed.

**This is not a reason to abandon Phase 10** — the content still has to reach customers somehow. It is
a reason to prefer the route that rests on supported surface. Worth separating two things Microsoft
says, because they are easy to conflate:

| Surface | Microsoft's position |
| --- | --- |
| The content management **REST API** | Supported — *"allows you to import data to, or export from, an API Management service through the content management REST API"* |
| The **migration script** between v2 instances | Not supported |
| **ARM templates** / DevOps Resource Kit for portal content | Not supported |

Sources: [Automate developer portal deployments](https://learn.microsoft.com/en-us/azure/api-management/automate-portal-deployments)
and [Developer portal FAQ](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-faq).

Reading those together: the operations themselves are supported, the packaged migration path is not.

> [!warning] Resolved 11 August 2026 — driving the API ourselves is **not** a viable alternative
> An earlier version of this note tilted towards driving the content management API ourselves rather
> than shelling out to Microsoft's script. Research has since closed that off. Text content (pages,
> layouts, navigation) genuinely is addressable — Pulumi's `azure-native` provider even ships an
> `apimanagement.ContentItem` resource. But **media cannot be reached by any IaC tool**: uploaded
> images and PDFs live in a storage account owned by Microsoft, not by us, and the only way in is a
> one-hour SAS returned from a `POST` action. Verified directly — the container behind `vprk2` sits on
> `apimstj2nsbmrqwjjbe5vpnf`, which does not appear in our subscription at all.
>
> So any hand-rolled approach would reproduce every page and lose every logo. Shelling out to
> `scripts.v3` is not a shortcut we are taking for want of a better option — **it is the only complete
> option that exists.** See [[APIM IaC - portal content, why there is no IaC-native route]].

**Also worth recording:** the same FAQ confirms *"Is the portal's content saved with the backup/restore
functionality in API Management? **No.**"* ([Developer portal FAQ](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-faq))
APIM's own backup does not cover portal content, so the only copy of a customer's portal is whatever
we take ourselves. That makes the snapshot archive in G24 more important, not less.

## G27 — A read-only role cannot copy the portal's images, and the obvious role does not either

**Found:** 2026-08-11, Phase 10 permissions investigation. **Status:** 🔴

The content copy fetches two kinds of thing by two different mechanisms. Pages, text and styling come
back from ordinary `GET .../contentTypes/...` reads. Images and logos do not: the toolkit calls
`POST .../portalconfigs/default/listMediaContentSecrets` to obtain a temporary storage link, and in
Azure's permission model that is an **action**, not a read. Read-only roles do not include it.

Checked against Azure's live role definitions, the permission
`Microsoft.ApiManagement/service/portalConfigs/listMediaContentSecrets/action` is granted by
`API Management Service Contributor` and `Contributor`, and **not** by `Reader`,
`API Management Service Reader Role`, `API Management Service Operator Role`, or
`API Management Developer Portal Content Editor`.

That last one is the trap inside the trap. The role named after the exact job we are doing does not
grant what the job requires, so anyone reasoning from role names — a future us, or SRE reviewing a
request — lands on the wrong answer and it looks right.

The failure it produces is worse than an outright refusal. Text and styling copy **successfully**,
and the run only fails when it reaches the media. The customer is left with a portal carrying every
correct page and no logos, and the error concerns the customer's deploy rather than the source
instance it could not read. Nothing in the message points at a permission on `vprk2`.

**What to do about it:** grant a purpose-built custom role on the `vprk2` resource rather than
`API Management Service Contributor`, because Contributor grants **write** on the content master and
the tool's `cleanup` step exists to wipe whatever is named as a target — a config-generation bug that
named `vprk2` would then be permitted to erase the thing the technical authors write into. The five
operations needed are listed in [[APIM IaC - portal content permissions (phase 10)]]. Note also that
this is not one grant: **four** orchestrator service connections run deploys, and granting only some
of them produces a change that passes all our testing and fails on the first real customer. Verify
per service connection with the read-only probe in that document before trusting it, and add the same
probe to the hook as a pre-flight so the failure is legible.

---

## G28 — The "has anything changed?" fingerprint is not reproducible across machines

**Found:** 2026-08-11, first successful end-to-end portal copy to `grant-sbx-spk-apimgs-apim`.
**Status:** 🔴

The whole point of the change check is to avoid rebuilding a customer's portal when nothing has
changed. It works by hashing the master portal's content into a single fingerprint, storing it on the
customer's APIM, and comparing on the next deploy. That only works if the same content always
produces the same fingerprint.

It does not. The deploy — running inside the Linux dev container — computed and stored
`520f22bd0769…`. The same read, run against the same unchanged master from Windows minutes later,
computes `276787521cbf…`. The Windows value is stable: identical before the copy, after the copy, and
hours apart. So neither machine is being random; the two are simply reading something differently.

**Why it matters more than it first looks.** The obvious cost is time, and on the sandbox a full copy
took about two minutes rather than the ten to fifteen the runbook implies — easy to wave through. The
real cost is what the copy *does*. It is not additive: it wipes the target portal and rebuilds it from
the master. If the fingerprint never matches, that wipe-and-rebuild happens on **every single
deploy**, against the live portal a customer is signed into. A deploy that changes nothing about the
portal still takes the portal down and puts it back. That is an availability risk on a customer-facing
surface, not a build-time inefficiency.

**What has been ruled out**, so the next person does not repeat it:

- *Swapped arguments* — the call site is `compute_fingerprint(source, target, devportal_url)`, correct.
- *Truncation* — the full 64 characters are stored; only the log line is shortened to 12.
- *The copy overwriting sign-in settings* — checked in Azure; the target kept its own IdP client ID
  (`0f9d0e25…`) and did not inherit the master's (`f307b49e…`).
- *Volatile fields in the content* — the only version marker present is a static `"version":"v16"`.
- *Pagination* — every content type returns complete. `page` 23, `document` 29, `layout` 1,
  `blogpost` 0, `blob` 11, `url` 16, `navigation` 0, `block` 1, `popup` 1; each response's `count`
  matches its item count and none carries a `nextLink`. This one mattered most, because a short read
  is exactly the failure mode behind G25.

**The cheap test that has not been run yet.** The dry-run script now prints a hash per part rather
than one combined value — content items, media listing, sign-in IdP, sign-in tenant. Running it
*inside the dev container* takes about 25 seconds and isolates which single input differs between the
two machines. Windows currently reports: content items `e5ec64208728` (162,486 chars), media listing
`e4eb6ee54392` (1,214 chars), sign-in IdP `0f9d0e25-896d-46b9-bf77-dba8554acc93`, sign-in tenant
`201e1d9f-2fb3-4cb1-9306-cb6c0dbefde9`, publish trigger `70058fa4d52bef95`.

The likeliest candidates are the media listing — a storage listing whose XML can legitimately differ
in ordering or in which optional fields are returned — and any text normalisation that differs
between platforms, such as line endings.

**What to do about it:** treat this as a required fix in the Phase 10 ticket, not a nice-to-have. Until
the fingerprint is proven stable across the machines that will actually run it, the safe assumption is
that the copy runs every time, and the ticket should carry that as its worst case. Two things are worth
building in regardless of the root cause: log both fingerprints on a mismatch so the divergence is
visible rather than silent, and hash from a normalised, explicitly-ordered structure rather than from
raw response text, so platform differences cannot leak in.

---

## G29 — The dev deploy command offers to register an unknown environment, and saying yes creates a wrong one

**Found:** 2026-08-11, attempting to deploy to a shared environment. **Status:** 🔴 will bite as-is ·
**Bites during:** anyone's first attempt to deploy outside their own sandbox

There are **two** environment registries. The development deploy command reads the dev one — 73
entries, all personal sandboxes. Every shared and customer environment lives in the other, behind a
credential store most of us cannot read.

Point the dev command at a real environment and it doesn't refuse. It offers:

```
First time seeing client_id 'drch'. Add to GSOT? (y/n)
```

> [!danger] Answering yes creates a **fabricated** record
> It writes an entry pointing at the wrong subscription and a web address that does not exist. The
> deploy then proceeds and appears to work.
>
> The result is a broken deployment in the wrong place, and a bad record left behind for whoever
> comes next.

**The safe answer is always `n`.** There is no correct way to reach a shared environment through the
dev command — the answer is the **deploy pipeline**, not a shortcut.

> [!success] Resolved 2026-08-11 — and the cause was the opposite of what it looked like
> `drch` is not an unknown environment. It is a **properly configured, existing one** —
> `pipelines/Definitions/Internal/Dayshape.Deploy.Internal-Dev-Dreich.yml` declares
> `client_id: drch` against environment `dev-drch-ds-lwr-dv`.
>
> The prompt appears because the **dev registry** genuinely hasn't seen it — real environments live
> in the other one. So "first time seeing" is true of that registry and **false of reality**, and
> saying yes would have created a **duplicate, wrong** record shadowing a correct one.
>
> **The correct route** *(per Richard Donkin)* is the environment's own deploy pipeline — Dreich is
> ADO definition **739** — using its **`iacBranchOverride`** parameter to point at a spike branch.
> See **[[D5 - Getting access to test on a real environment|D5]]**.

Worth noting the prompt is doing something reasonable for its intended use *(registering a genuinely
new sandbox)*. It's being used outside that context, and gives no indication it's about to invent
something that already exists elsewhere.

---

## G30 — Two deployment identities can read the content master by accident, and production cannot at all

**Found:** 2026-08-11, verifying the four service connections against live Azure. **Status:** 🔴 will
bite as-is · **Bites during:** the first customer deploy, after all our testing has passed

G27 established *which* permission the portal copy needs. This is what the four deployment identities
**actually have**:

| Connection | Access to `vprk2` | How |
| --- | --- | --- |
| `…Orchestrator.Internal` | ✅ | `Contributor` **inherited from the whole subscription** |
| `…Orchestrator.Sandbox` | ✅ | `Contributor` + `Owner`, inherited |
| `…Orchestrator.Lower` | ❌ | — |
| `…Orchestrator.Upper` 🔴 **production** | ❌ | — |

**Not one role assignment is scoped to the master itself.** Both working identities work incidentally.

> [!danger] The failure is perfectly disguised
> The two that work are the two we test with. **Dreich and sandbox pass; production fails on the
> first real customer**, at the point where the portal copy runs — which fails the whole deployment.
>
> Nothing in a passing test run hints at it.

Two further consequences:

- The working access is **subscription-wide `Contributor`** — far broader than the five rights needed,
  and it includes the write access that makes G25's "cleanup wipes the master" scenario possible.
- Because it's incidental, it could be **tightened for entirely unrelated reasons** and silently break
  every customer deployment. Nothing anywhere records that APIM depends on it.

**Fix:** an explicit least-privilege assignment on all four, scoped to the `vprk2` resource — which
both closes the gap and removes the reliance on inherited Contributor. → **D2**

---

## G31 — A component the environment has not been told about is skipped silently, and the deploy reports success

**Found:** 2026-08-11. **Corrected:** 2026-08-11 — *the first version of this entry described the
wrong failure; see below.* **Status:** 🔴 will bite · **Bites during:** the first deploy of any
*new* component to any environment

The deployment does **not** work from the list of components in the codebase. It works from a list
held in the **environment registry** (GSOT) — the "infra targets" for that environment. The
orchestrator fetches that list and loops over it.

`--components ApimSpike` is a **filter applied to that list**, not an instruction to deploy.

So when a component exists in the code but has never been registered against the environment:

- it is not in the list, so the loop never reaches it
- the filter excludes everything that *is* in the list, as out of scope
- **nothing deploys, nothing errors, and the pipeline goes green**

> [!danger] The dangerous part is that it looks like success
> No warning, no "component not found" — a tidy successful run that deployed nothing.
>
> Without knowing to check, the natural reading is *"the deploy worked but APIM did not appear"*,
> which sends you hunting through Pulumi and Azure for a problem that is not there.

### ⚠️ Correction — what I first wrote here was wrong

I originally recorded this as *"the deploy stops on a `(y/n)` prompt nobody can answer."* That prompt
is real, but it lives in **`orchestrate-dev`**, the local developer command. **Pipelines run
`orchestrate`, which has no prompts at all.**

I found the prompt, saw it sitting next to component selection, and did not check which command the
pipeline actually calls. Grant pushed back with exactly the right instinct — *if that were true, no
new component could ever be deployed by anyone* — and he was right.

**The conclusion survives; the mechanism does not.** Registration is still required first. It simply
fails silently instead of loudly — and the corrected failure is **harder to diagnose** than the one
I first described.

### How components actually get registered

New environments are seeded with a **fixed starter set** — `Storage, LanguagePack, Certificate,
Networking, Core, Reporting, GenericIntegration, Notification, Solver, Auth`. **APIM is not in it**,
and neither is anything else added later.

There is a CLI command, `add-target-infra`, but it is **hard-blocked to the dev registry**:

```python
if not use_dev_db:
    raise ValueError("Only the dev GSOT can be targetted for the time being")
```

Dreich and every customer environment live in the **real** registry.

> [!warning] Two registries
> `ds-env-config-devdata-sqldb` (**dev**) vs `ds-env-config-sqldb` (**real**).
> `orchestrate-dev` hardcodes **dev**; pipelines default to **real**. A component registered by a
> local run is invisible to the pipeline — which is exactly what happened with sbx.

**Four routes to register a component** on a real environment:

| | Route | Trade-off |
|---|---|---|
| A | SRE inserts the `InfraTargets` row | Fast, no code, no trail |
| B | Relax the dev-only guard on `add-target-infra` | Command already exists for this; guard is presumably deliberate |
| C | Add it to `ENV_STARTER_COMPONENTS` + re-run **ConfigSync** | In code, reviewable, idempotent — but applies to **every** environment |
| D | `orchestrate-dev` locally | Writes to the **dev** registry only — no help |

**Correction:** first written as "no self-service route — needs SRE". Too absolute. **C** is
self-service and stays in code; it is a product decision rather than a test convenience.

**Beyond APIM:** adding a component to a real environment is **not purely a code change**. Build the
registration step into the ticket.

### How to check whether a component is registered

`dbo.EnvComponentRolloutStatus` (`scripts/apis/dsenv/views/env_component_rollout_status.sql`) exposes
this directly, alongside the environment's versioning mode and last deployment result:

```sql
SELECT ClientId, Env, ComponentName, InfraTargetStatus,
       IacVersioningMode, LastDeployedDate, DeploymentSucceeded
FROM dbo.EnvComponentRolloutStatus
WHERE ClientId = 'drch' AND ComponentName = 'ApimSpike';
```

> [!warning] An empty result is not the same as `MISSING`
> The view cross-joins against `SELECT DISTINCT ComponentName FROM dbo.InfraTargets` — the set of
> component names registered **somewhere**. A component registered nowhere at all doesn't appear in
> that set, so you get **no row**, not a `MISSING` row. Read "no rows" as "never registered against
> any environment".

---

## G32 — There is no dry-run through the pipeline

**Found:** 2026-08-11. **Status:** 🟠 worth knowing · **Bites during:** the first run of anything

The orchestrator supports a **preview** operation — show me what would change, change nothing. It's
available when running locally.

**The pipeline doesn't expose it.** The only operation control is a `destroy` box, and the script
accepts only the exact words `destroy` or `force_destroy`; anything else falls back to a real deploy.
Typing `preview` there does **not** give a preview — it gives a deployment.

So the first pipeline run against any environment is a real one. Locally-run previews are the only
rehearsal available.

**Small, contained improvement for the real ticket:** allow `preview` through the same box. It is a
few lines, and it makes every future first-run against a customer environment rehearsable.

---

## G33 — A branch override beats IaC versioning, but only on internal environments

**Found:** 2026-08-12. **Status:** 🟢 useful to know · **Bites during:** any test deploy from a branch

This one is recorded because I initially got it **wrong in the pessimistic direction**, and the
correction is worth keeping.

**The wrong assumption:** that `iacBranchOverride` only applies when the deploy builds from a repo
checkout, so if IaC versioning were switched on for the target environment, the override would be
skipped silently and a published version deployed instead — a green run with none of your code in it.

**What the code actually does** (`iac_version_resolver.py`, `_resolve_override`): the override is
evaluated **before** the versioning mode is read at all. When set on an internal environment it
returns "no version — use a repo checkout", and versioning is bypassed entirely. The code comment
calls this *"the override always wins"* behaviour.

**Consequence:** for a branch test on an internal environment, you never need to know or care what
that environment's `IacVersioningMode` is.

**The real constraint is *where*, not *whether*.** Overrides are refused on genuine client
environments:

| Case | Rule |
| --- | --- |
| Current internal | `env_group` is `sbx` or `int` |
| Legacy internal | `client_group_id` is `sre` or `dev` **and** `env_group` is `lwr` |

Anything else raises `IacOverrideNotPermittedError` — **loudly**, with the client id, env, env group
and client group all in the message. This is a good failure: it says exactly what it refused and why.

Dreich (`dev-drch-ds-lwr-dv`) is legacy internal, so overrides are permitted there.

**Two details that look like traps and aren't:**

- The ADO default is `-`, not empty, because ADO UI parameters can't default to an empty string.
  Both the resolver call and the checkout step explicitly translate `-` to empty. Same trick as
  `components`.
- When an override is set it also supersedes the Core artifact commit checkout, and logs a warning
  saying so. Expected, not a fault.

> [!tip] The wider lesson
> Twice now in this spike I've assumed a *silent* failure where the code actually fails loudly (this
> one), and once I assumed a *loud* failure where it's genuinely silent (G31). The pattern is that
> assuming is the problem, in either direction. Read the resolver, not the parameter description.

---

## G34 — Naming one component does not pull its dependencies in, it silently drops them

**Found:** 2026-08-13, while working out why a Dreich run failed. **Status:** 🔴

`ds_manifest.xml` lists each component's dependencies, and `ApimSpike` declares two — `Storage` and
`Auth`. It is natural to read that as *"ask for ApimSpike and the orchestrator will bring Auth along."*

**It does the opposite.** From `scripts/domain/component_ordering/component_ordering.py` L37-41:

```python
# First, clear out any dependencies that are not pending in our update scope
for manifest in to_process:
    for dependency in manifest.dependencies[:]:
        if dependency not in [manifest.name for manifest in to_process]:
            manifest.remove_dependency(dependency)
```

Anything not already in the requested set is **removed from the dependency list**, not added to the
run. So `components = ApimSpike` deploys ApimSpike alone, with no warning that two declared
dependencies were discarded.

This is a reasonable design — it means "deploy just this one thing" actually works, and it is what
makes targeted redeploys possible. The trap is purely that the manifest *reads* like a request and
behaves like an ordering hint. **Dependencies order a deploy; they do not trigger one.**

> [!danger] ❌ This was **not** the cause of the failed Dreich run — corrected 2026-08-13
> This callout originally claimed the 2026-08-12 Dreich failure was caused by `components =
> ApimSpike` dropping `Auth`. **The full log arrived later and disproves it.** Recording the wrong
> guess here rather than deleting it, because the reasoning looked sound and it is worth knowing why
> it was wrong.
>
> **What the log actually shows:**
> - The run was **`all components`** for `drch`/`dv` (*"Preparing to perform 'up' operation on all
>   components"*), not a scoped one. `Auth` was never excluded by this trap.
> - It failed in APIM's Phase 4 with
>   `No applications found matching filter: "displayName eq 'drch-dv-spk-auth'"` — nothing to do with
>   the vault or the sign-in secret. It died well before that point.
> - **The real cause: at that moment `ApimSpike`'s manifest declared no dependencies at all.** They
>   were added to the manifest **1h48m after the run** (commit `998fc3cbf35`, 12:50; the run was at
>   11:02). With no dependencies, `ApimSpike` sorted into the *first* wave and ran **before `Auth`**.
>
> So the failure was **the absence of the manifest dependencies**, not this trap discarding them.
> Ironically the fix for that failure is the very thing that now makes this trap reachable.
>
> The real cause is written up as **G35**.

**Why this trap still stands.** The mechanism above is real and verified in the code — it simply was
not what bit on 2026-08-12. It becomes reachable **now**, precisely because `ApimSpike` has declared
dependencies: asking for `ApimSpike` alone will silently discard them.

**What to remember.** If a run needs another component's work to have happened, either name both
components explicitly, or make sure that work has already been done in a previous run. Do not rely on
the manifest to arrange it.

Related: **G31** is the same family of problem — something is quietly left out and the deploy still
reports success. **G35** is the failure this was wrongly blamed for.

---

## G35 — Dreich's sign-in record uses an older naming convention, so APIM cannot find it

**Found:** 2026-08-13, from the full log of the failed 2026-08-12 Dreich run. **Status:** 🔴
**Bites during:** any APIM deploy into an environment whose `Auth` component is out of date.

Phase 4 does not hard-code the Core sign-in record — it looks it up **by name**, via
`get_core_dayshape_application()` (shared with the AI Agent). The name is calculated:

```
{clientid}-{environment}-{product}-{component}   ->   drch-dv-spk-auth
```

**On Dreich that record does not exist.** Checked directly in Entra ID on 2026-08-13 — there are
three registrations for `drch`, and the expected name is not among them:

| Name in Entra | What it is |
| --- | --- |
| `drch-dev-Spoke-auth` | The IaC-created one — but under an **older naming convention** |
| `dev-drch-ds-lwr-dv` | The legacy record Dreich's Core **actually** signs users in against |
| `drch-dv-spk-wrkd-app` | Workday integration, unrelated |

The naming scheme changed at some point from `dev`/`Spoke` to `dv`/`spk`. **30 other environments
already carry the new name** (`laldi-dv-spk-auth`, `glaik-dv-spk-auth`, `grant-sbx-spk-auth`, …) —
Dreich simply has not had `Auth` redeployed since the change, so it is stale.

That is also why the earlier proof run **worked on `grant-sbx` but failed on Dreich**: Grant's
sandbox has `grant-sbx-spk-auth` under the current name.

### What this means in practice

- **Deploying `ApimSpike` on its own to Dreich will fail**, with the error above, until `Auth` has
  been run there at least once.
- **Running `Auth` fixes it**, by bringing the name up to date. Verified non-destructive: `display_name`
  on `azuread.ApplicationRegistration` has no `replaceOnChanges`, so it is an **in-place rename** —
  the record keeps its identifier and nothing that trusts it breaks.
- This is **not** APIM-specific. Any component using `get_core_dayshape_application()` — the AI Agent
  included — has the same dependency on `Auth` being current.

> [!warning] A second mismatch worth checking after the run
> Dreich's Core is configured to trust **`dev-drch-ds-lwr-dv`**, *not* the IaC record. The hand-made
> `apim-auth` secret was created on that legacy record too, which suggests the manual guide targeted
> it deliberately.
>
> APIM will put the portal on **`drch-dv-spk-auth`**. Signing in to the portal should work, but the
> **"Try it" call against the real API may be rejected**, because the token would be issued for a
> record Core does not trust — see **G5**. Worth confirming on Dreich rather than assuming.

`get_core_dayshape_application()` already accepts a `legacy_client_id` argument, but **it is unused**
and no caller passes one. That looks like the intended escape hatch for exactly this, left unfinished.

---

## G36 — "Do not delete" is inherited by children, so anything hung off the sign-in record can never be removed

**Found:** 2026-08-13, first sandbox run of the D1 change. **Status:** 🔴

The deploy failed on `Auth` with:

```
error: Preview failed: resource "...ApplicationRegistration$...ApplicationPassword::
grant-sbx-spk-auth-apim-devportal-client-secret" cannot be deleted
```

Nothing in the D1 change touches that resource except to **delete** it — that is the whole point of
the change. So why can it not be deleted?

**The sign-in record is deliberately marked "do not delete"** (`auth_component.py` L86-89):

```python
opts=ResourceOptions(
    # prevents accidental deletion; recreating revokes all externally-granted admin
    # consent, which each client tenant must re-grant themselves
    protect=True,
),
```

That is a sensible safeguard and should stay. The trap is that **the flag is inherited**. The old
secret was declared with `parent=self.application`, and Pulumi's own code
(`pulumi/resource.py` L986-988) does this:

```python
# Infer protection from parent, if one was provided.
if opts.protect is None:
    opts.protect = opts.parent._protect
```

Nobody wrote `protect=True` on the secret. It picked it up from its parent, silently, and was
written into the deployment's memory that way. The safeguard meant for the registration now guards
a throwaway credential as well.

> [!warning] This applies to anything parented to that registration, not just this one secret
> Any future resource declared with `parent=self.application` inherits the same flag and becomes
> equally unremovable. It only bites when you come to delete it — which may be months later, by
> someone who did not write it.

**Clearing it is a one-off manual step per environment**, done *before* the deploy.

Pulumi's state lives in Azure blob storage, so the CLI needs credentials before it will talk to
anything — running `pulumi` cold gives `unable to open bucket azblob://<client>-blob:
azureblob: Options.AccountName is required`. The repo ships a helper for this
(`Products/tools/enable-pulumi.sh`, documented in `Products/tools/ENABLE_PULUMI.md`). It must be
**sourced**, not executed:

```bash
cd /mnt/repo/Products
source tools/enable-pulumi.sh <client> <environment> <config_client>   # e.g. grant sandbox ds

cd Spoke/Auth/infra
pulumi stack select organization/Spoke.Auth/<client>.<env>
pulumi state unprotect --yes \
  'urn:pulumi:<client>.<env>::Spoke.Auth::azuread:index/applicationRegistration:ApplicationRegistration$azuread:index/applicationPassword:ApplicationPassword::<prefix>-apim-devportal-client-secret'
```

> [!warning] Single quotes around the URN, not double
> The URN contains a `$` separating the parent from the child. In double quotes the shell reads
> `$azuread` as a variable, substitutes nothing, and hands Pulumi a URN that matches no resource.

Two other details that are easy to get wrong: the working directory is the component's **`infra`**
folder, not the component folder; and the SAS token the helper mints **expires after 12 hours**, so
re-source it if you come back to this later.

For grant's sandbox that is stack `organization/Spoke.Auth/grant.sandbox` and prefix
`grant-sbx-spk-auth`. The exact URN is quoted verbatim in the failure message, so it can be copied
straight out of the log rather than assembled by hand.

**How many environments are affected?** Only ones where the old code actually ran, which means
`apim_enabled` was switched on. That flag defaults to `false`
(`Spoke/Auth/config/infra_defaults.yml`) and this is unreleased spike code, so in practice it is
**grant-sbx and possibly Dreich — nothing else.**

**If we would rather not touch deployment state by hand**, the alternative is a two-pass release:
put the old secret back with an explicit `protect=False`, deploy once (which does nothing but clear
the flag), then remove the code and deploy again. Fully automated, but two releases and dead code in
between. Worth it for a wide rollout; probably not for two environments.

---

## G37 — A deleted APIM reserves its own name for 48 hours, and nothing in our code clears it

**Found:** 2026-08-13, same sandbox run. **Status:** 🔴

The `ApimSpike` half of the same run failed with:

```
Status=409 Code="ServiceAlreadyExistsInSoftDeletedState"
Message="Api service grant-sbx-spk-apimgs-apim was soft-deleted. In order to create the new
service with the same name, you have to either undelete the service or purge it."
```

Deleting an APIM does not release its name. Azure holds it in a **soft-deleted** state, because the
name is also its public address (`<name>.azure-api.net`). Confirmed live:

| | |
|---|---|
| Deleted | 2026-08-12 13:43 |
| Name released | 2026-08-14 13:42 |
| Window | **48 hours** |

So **tear an environment down and rebuild it the same day and the rebuild fails** — not because
anything is wrong with it, but because it is competing with its own corpse for the name.

Two ways out: purge it (frees the name immediately) or wait out the 48 hours.

**To check whether one is sitting there:**

```bash
az rest --method get --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.ApiManagement/deletedservices?api-version=2022-08-01"
```

**To purge it** — take the `id` straight from that response and `DELETE` it:

```bash
az rest --method delete --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.ApiManagement/locations/<region>/deletedservices/<name>?api-version=2022-08-01"
```

Note the region in the URL is the region the APIM **was deployed in**, not necessarily the one you
are deploying to now. For grant's sandbox that is `westeurope`. A successful purge returns nothing.

> [!warning] Purging is irreversible
> It is the delete that soft-delete exists to protect you from. There is no undo and no recovery of
> the gateway's configuration, certificates or keys. Fine for a sandbox being rebuilt from code;
> think much harder anywhere else.

> [!danger] The purge returns instantly but is not finished — expect `ServiceLocked` next
> Seen immediately after purging on grant-sbx, 2026-08-13. The `DELETE` came back straight away, the
> deleted-services list came back empty, and the very next deploy still failed:
>
> ```
> Status=409 Code="ServiceLocked" Message="The API Service grant-sbx-spk-apimgs-apim is
> transitioning at this time. Please try the request again later."
> ```
>
> Azure reports the purge as done long before it has finished dismantling the gateway behind it, and
> the name stays locked while that happens. **Nothing is wrong and nothing needs fixing — wait a few
> minutes and run again.** The tell is the run duration: it failed in **11 seconds**, so it was
> rejected outright rather than attempting the real 30–45 minute build.
>
> Worth knowing because it looks like the purge did not work. It did.

> [!tip] There is no reliable pre-check — the only real signal is time
> It is tempting to use Azure's name-availability API as a green light:
>
> ```bash
> az rest --method post \
>   --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.ApiManagement/checkNameAvailability?api-version=2022-08-01" \
>   --body '{"name":"<apim-name>","type":"Microsoft.ApiManagement/service"}'
> ```
>
> **It does not work for this.** Proven on grant-sbx 2026-08-13: name reported `nameAvailable: true`,
> the soft-deleted list was empty, and `az apim show` returned `ResourceNotFound` — all three saying
> "gone" — and the very next deploy still failed with `ServiceLocked` after 33 seconds.
>
> The check answers "is this *name* taken?". `ServiceLocked` is a different question: Azure is still
> tearing down the gateway on its scale unit, and refuses new work on that name until it finishes.
> **No public API reports that state.**
>
> **What to do instead: allow 30–45 minutes after a destroy or purge before redeploying.** That is
> how long APIM genuinely takes to dismantle, regardless of how quickly the delete call returned.
> The check above is still worth running — a `false` is a definite stop — but a `true` means only
> "not obviously blocked", not "safe".
>
> **The tell remains the duration.** A `ServiceLocked` failure comes back in well under a minute. A
> real build takes 30–45 minutes. If it failed in seconds, nothing was attempted; wait longer and
> run again.

> [!warning] Budget an hour and a half for a full destroy-and-rebuild
> Teardown (30–45 min, invisible) plus build (30–45 min). Worth knowing before choosing a clean
> rebuild over a plain re-run — a re-run is usually enough, because a failed portal copy is retried
> automatically on the next deploy (the fingerprint is only written on success).

> [!note] We already solve this exact problem elsewhere, just not here
> `Modules/KeyVaultSecret` clears a soft-deleted secret of the same name before creating one — you
> can see it run in the log as `apim-hmac-signing-secret-purge`. There is **no equivalent for the
> APIM service itself**. Whether to add one is a real decision: it makes rebuilds painless, but it
> also means a deploy can permanently destroy the safety net that soft-delete exists to provide.
> This was flagged as an open question in the ticket plan and is still unanswered.

---

## G38 — The one-year expiry we ask for is ignored, and Entra issues two years instead

**Found:** 2026-08-13, checking the credentials the run produced. **Status:** 🟠

Both the old and the new secret ask for a one-year lifetime (`end_date_relative="8760h"`, which is
365 days). Neither got one. Read back from Entra:

| Credential | Created | Expires | Actual life |
|---|---|---|---|
| `apim-devportal-signin` (new, ours) | 13/08/2026 | 13/08/2028 | **2 years** |
| *(unnamed — the old Auth one)* | 12/08/2026 | 12/08/2028 | **2 years** |
| `apim-auth` (hand-made, from the manual guide) | 06/08/2026 | 06/08/2027 | 1 year |

The setting is silently dropped and Entra falls back to its own default of two years. The hand-made
one is a year because a human typed a year.

**This is not a fault and does not need fixing.** A longer life is the safer direction, and renewal
is driven by the 30-day rotation clock, not the expiry date. It matters for two reasons only:

- Anywhere we have written "the secret lasts twelve months" is **wrong** — it lasts two years.
- The residual risk in D1 — *"an environment left undeployed past the expiry breaks"* — is a
  **two-year** window, not one. Considerably less likely to ever happen.

If a fixed date is ever genuinely needed, `end_date` (an absolute RFC3339 timestamp) is honoured
where `end_date_relative` is not.

---

## G39 — APIM eats the version segment, Core insists on it, and nobody puts it back

**Found:** 2026-08-17, first real call through the Dreich gateway. **Status:** 🟢 fixed

Everything about the deploy looked right. APIM built, the portal rendered, Entra sign-in worked. The
first *Try it* call returned:

```
404  "Proxy API version segment 'grades' is not supported. Supported versions: v1."
instance: /proxy/grades
```

Both sides of the wall have a `v1` and they are not the same `v1`:

| | Value | Who owns it |
|---|---|---|
| APIM version set | `v1`, `VersioningScheme.SEGMENT` | the gateway's own addressing |
| Core proxy route | `proxy/v1` (`ProxyRouteConstants.V1`) | the application |

Under the `Segment` scheme APIM strips **both** the API path *and* the version before forwarding —
`/entities/v1/grades` leaves as `/grades`. Appended to a backend of `…/proxy`, Core received
`/proxy/grades`, read `grades` as the version, and correctly refused it. The OpenAPI document could
not have saved us: its paths are `/grades` with no `servers` block.

### What to do

Put the segment back in the backend URL, derived from the same constant the `Api` resource is given
so the two cannot drift when v2 arrives:

```python
backend_url = f"{get_dayshape_url(...).rstrip('/')}/proxy/{apim_spike_helpers.ENTITIES_API_VERSION}"
```

A `<rewrite-uri>` in the policy would also work and is worse: the backend URL would still *read* as
`/proxy`, so the next person sees a URL that looks correct and behaviour that is not.

### The wider point

> [!warning] A green deploy proved nothing about whether the API worked
> Every resource reported success, sign-in worked, and **every API call was broken**. Nothing in the
> pipeline could have told us. This is the argument for **T9** carrying one authenticated call
> through the gateway asserting a 200 — cheap, and the only thing that would have caught this.

---

## G40 — "Core" is three different component names, and checking for one of them 403s the whole API

**Found:** 2026-08-17, on Dreich, immediately after G39 was fixed. **Status:** 🟢 fixed on the spike
branch · **Bites during:** any deploy to an environment that is not hosted the way we assumed

G23 says a missing signing secret "looks like a successful deployment and then 403s everything", and
notes the hook **raises rather than warns** to prevent exactly that. It still happened — because the
hook never got as far as writing anything to fail on.

### What happened

Dreich deployed green. The portal rendered, sign-in worked, routing was correct after G39. Every call
through the gateway came back:

```
403  APIM verification is enabled but not configured.
```

Core had the check switched on with **no secret to check against**. The only trace in a 3,800-line
log was one line, at information level:

```
Skipping ApimHmacSecret: no Core component in this environment, so nothing reads the secret.
```

### The cause

The Dayshape application is not one component. `Globals.py` defines **three** hosting variants, and an
environment registers only the one it uses:

| Component | Short name | Used by |
| --- | --- | --- |
| `Core` | `core` | the containerised hosting |
| `CoreLegacyVM` | `legvm` | **Dreich, and the older client environments** |
| `CoreAppService` | `coreas` | app service hosting |

Our hook looked for the literal string `"Core"`. Dreich registers `CoreLegacyVM`. So the hook
concluded there was no Dayshape application to tell, skipped the secret, and reported success.

`AsbQueues` already treats `CoreLegacyVM` and `Core` as interchangeable — the house pattern existed
and we did not follow it.

### Why the safeguard missed it

G23's protection is on the **write**. This was a **skip**, which is a legitimate path — a portal-only
sandbox genuinely has nothing to tell. The guard for "the write failed" says nothing about "we
decided not to write".

Two changes, not one:

- the check now covers all three names
- the skip logs a **warning** naming the consequence, not information. A skip in the wrong
  environment is now the loudest thing in the log rather than the quietest.

### The bit that is still awkward — and it is not just ordering advice

G23 says *"deploy APIM before Core"*. On Dreich that is **not something an engineer can choose**: the
release pipeline does both in one run, and it does Core first.

| Job | Ran | What it does |
| --- | --- | --- |
| `Deploy Core service` | 14:44:45 – 14:46:44 | Copies configs, substitutes variable-group secrets into `Dayshape.config` |
| `Orchestrate Infra` | 14:49:58 – 15:01:31 | Runs Pulumi, including APIM and the hook that writes `ApimHmacSecret` |

Core took its copy of the settings **three minutes before APIM started**. So on the first release
after enabling APIM for a customer, the variable group is still empty when Core reads it, and Core
comes up with the check on and nothing to check against.

The secret is a `pulumi_random.RandomPassword` held in Pulumi state, so it does not change between
deploys. That limits the damage: **only the first release is short**, and every one after it is
fine. Recovery is a second release, or the much cheaper **Deploy Configs Only (VM)** (ADO pipeline
645).

Verified on 2026-08-17: the Dreich `_Core` group contained no `ApimHmacSecret` at all, which is what
the skipped hook looks like from the outside.

> [!warning] Third green deploy in a row with a completely broken API
> G39 and G40 were both found by hand, both after a fully successful pipeline run. Nothing we have
> would have caught either. The case for one authenticated smoke call asserting a `200` is no longer
> theoretical.

---

## G41 — the environment values are remapped, and the remap is wrong for anything Pulumi did not create

**Found:** 2026-08-18, on Dreich, immediately after G40 was fixed. **Status:** 🟢 fixed on the spike
branch · **Bites during:** any deploy to an environment named before the current conventions

### What happened

With G40 fixed, the hook got as far as looking for the variable group and stopped the deploy:

```
Variable group 'dev-drch-ds-int-dv_Core' was not found.
```

Dreich's group is `dev-drch-ds-**lwr**-dv_Core`. One segment wrong.

### The cause

`componentised_env_config_factory.py` deliberately rewrites the environment values on the way in:

| Raw GSOT value | Becomes | When |
| --- | --- | --- |
| any `env_group` | `int` | environment is in the Dayshape internal subscription |
| `qa` / `st` / `pr` | `iq` / `is` / `ip` | same |
| any `env_group` | `sbx` | sandbox subscription |

That remap is **correct and deliberate** — its comment explains it stops internal `sre` environments
being forced into the customer state stores. It is for naming **Azure resources**.

Azure DevOps variable groups are not Azure resources, and were never named from the remapped values.
We used the remapped ones anyway.

### Both naming conventions are live, and that is fine

| Older | Newer |
| --- | --- |
| `dev-drch-ds-lwr-dv_Core` | `dev-bee1-ds-int-dv_Core` |
| `dev-qaa7-ds-lwr-dv_Core` | `dev-lauth-ds-int-dv_Core` |
| `dev-sre1-ds-lwr-st_Core` | `dev-glaik-ds-int-dv_Core` |

The instinct is that no single calculation can be right for both. It can, and it is the **raw GSOT
value** — because `config_sync_service.py` writes the GSOT record and the group name from the *same
expression*:

| Line | Creates | From |
| --- | --- | --- |
| 157 | the variable group name | `environment.environment_group.short_name` |
| 308 | the GSOT `env_group` field | `environment.environment_group.short_name` |

For anything config sync created, the two **cannot disagree**. For older environments, GSOT still
holds whatever was set at the time, which is also what their group is called. Raw is right for both.

### The rule

> [!tip] Mapped values for what Pulumi builds. Raw GSOT values for everything else.
> The Reporting hook already does this — its `__get_terraform_reporting_*` methods name
> Terraform-era resources and read `env_config.gsot_env_config.*` throughout. `ApimSpike` was the
> only place breaking the pattern.

### A second bug found while fixing the first

`dev-sre1-ds-lwr-**st**_Core` shows the group keeps the raw *environment* too. Our mapped value there
would have been `is`, so internal QA, staging and production would have failed exactly as Dreich did
the moment APIM reached them. Both fields were wrong, not just `env_group`. Nothing had reached those
environments yet, so it had not been seen.

### The one that behaved

Worth noting against G39 and G40: this failure **stopped the deploy**. It did not report success with
a broken API. That is the difference between a check that raises and a check that logs — and it is
the argument for the smoke test, made from the opposite direction.

---

## G42 — the deploy identity can read variable groups but not change them

**Found:** 2026-08-18, on Dreich, immediately after G41 was fixed. **Status:** 🔴 open — needs a
decision, not a code change · **Bites during:** the Phase 5 secret handover, in every environment

### What happened

With G41 fixed, the hook found the right group and got one step further:

```
INFO  Writing ApimHmacSecret to variable group dev-drch-ds-lwr-dv_Core
ERROR Failed to update variable groups with id '1328': 403
      'You do not have permissions to perform this operation on the variable group.
       A variable group Administrator should add you to the Administrator role.'
      IdentityDisplayName: 'Airts Build Service (AirtsOps)'
```

The lookup succeeded. The write did not.

### The cause — and it is deliberate

The Airts project build service is granted **View only** on the Library, and every variable group in
the project inherits it:

| Scope | Permission for `Airts Build Service` |
| --- | --- |
| Library (project level) | `allow = 1` (View) |
| every `_Core` group checked | `effectiveAllow = 1` |

Checked across both naming eras and beyond `_Core`, all identical:

| Group | id | Effective |
| --- | --- | --- |
| `dev-drch-ds-lwr-dv_Core` | 1328 | View |
| `dev-bee1-ds-int-dv_Core` | 2432 | View |
| `dev-qaa1-ds-lwr-dv_Core` | 184 | View |
| `dev-lauth-ds-int-dv_Core` | 2536 | View |
| `dev-day-ds-lwr-st_Core` | 1219 | View |
| `dev-caes-ds-int-dv_IdentityProviders` | 2596 | View |

This is **not a Dreich quirk and not a missing grant on one group**. No pipeline running in the Airts
project can modify any variable group. Updating one requires `Administer` (bit 2) in the Library
namespace.

There is one explicit ACE with full rights (`allow = 23`) belonging to the build service of project
`7db7a126-a7a1-40f9-a292-1402d920f704`, which is **not** one of the four projects that exist today —
a stale grant from a project that has since gone.

### Moving the write into the pipeline does not help

The obvious idea is to do what Svix does — `DevOps.VariableGroups.Update.ps1` against
`$(LegacySpokeRef)_Core`, with `system_accesstoken: $(System.AccessToken)`.

It would fail the same way. The Dreich release (ADO definition 739,
`pipelines/Definitions/Internal/Dayshape.Deploy.Internal-Dev-Dreich.yml`) runs in the **Airts**
project, so its `System.AccessToken` **is** the same View-only identity. Different mechanism, same
wall. Worth recording so nobody spends a day on it.

> [!note] Not yet established
> Whether the Svix step actually succeeds anywhere today. It does not run on Dreich, so this log
> proves nothing either way. If it does succeed, something about its context differs and is worth
> understanding before choosing.

### The convergence worth noticing

G40 found that Core takes its copy of the settings minutes before APIM writes them. G42 finds the
deploy identity is not allowed to write them at all. **Both trace to the same design choice** — the
signing secret is copied into Core's config at deploy time, via Azure DevOps, rather than read where
it already lives. Whichever way T6 goes, that is the thing being decided.

### Nothing to fix in the code

The hook behaved correctly: it raised, named the consequence, and stopped the deploy rather than
reporting success with a broken API. This is a permissions and design decision, not a defect.

---

## G43 — Core reads App Configuration directly, so the variable group is unnecessary

**Found:** 2026-08-18, from Sam Clifford in a call, then verified against the codebase.
**Status:** 🟢 resolved in principle — supersedes the approach behind G40 and G42

### The assumption that caused two failures

`apim_hmac_secret.py` carried this in its docstring:

> Dayshape reads its settings from a config file on disk that is filled in from that group
> during its own deploy, **and it does not resolve vault references**, so a copy is unavoidable
> on that side.

**That is wrong.** It was inherited from the Confluence setup guide, which describes the manual
process, and never checked. Both G40 (Core takes its copy before APIM writes it) and G42 (the
deploy identity is not allowed to write it) are downstream of that one sentence.

### What is actually true

`Dayshape/CodeBase/Utilities/Config/ConfigurationFactory.cs:41`:

```csharp
builder.ConfigureAzureAppConfiguration("spk-core", [MapFromStandardizedKey]);
```

And in `Dayshape.Services.AzureAppConfiguration/IConfigurationBuilderExtensions.cs`:

```csharp
options.ConfigureKeyVault(c => c.SetCredential(azureCredential));
```

So Core reads App Configuration at runtime **and** resolves Key Vault references using its own
managed identity. No copy of the secret is needed anywhere.

### The path a value takes, end to end

Verified each hop rather than assumed:

| Step | Where | Result |
| --- | --- | --- |
| Written as | `AppCfgKeyValueSet` | `spk-core-software-Dayshape-ApimHmacSecret`, label `{clientid}-{env}` |
| Prefix trimmed | `TrimPrefix` | `Dayshape-ApimHmacSecret` |
| Separator swapped | `ApplyConfigurationSectionSeparator` | `Dayshape:ApimHmacSecret` |
| Standardised-key mapping | `MapFromStandardizedKey` | no rule matches — passes through unchanged |
| Read by | `WebConfigHelper.GetSetting("Dayshape", "ApimHmacSecret")` | builds key `Dayshape:ApimHmacSecret` ✅ |

`WebConfigHelper` is backed by `IConfiguration`, and `DependencyConfig.cs:82` registers that as
`ConfigurationFactory().Build()` — the same builder that includes App Config. The chain closes.

**App Config is added *after* the file sections in `Build()`, and later sources win**, so the
App Config value overrides the `REPLACE-WITH-HMAC-SECRET` placeholder in `Dayshape.config`
without needing the file to change at all.

### Three components already do exactly this

None of them writes to a variable group:

| Component | Key it writes to Core |
| --- | --- |
| LanguagePack | `spk-core-software-LanguagePackStore-AzureBlobStorageContainerName` (+ account name) |
| Solver | `spk-core-software-Dayshape-SolverLocation` |
| CognitiveSearch | `spk-core-software-Search-AzureAISearchEndpoint` |

Solver is the closest parallel — it writes a private IP that does not exist until its VM is
built, which is the same "value only known at deploy time" shape as our secret.

### For a secret, use a Key Vault reference — and note the two mechanisms

`AppCfgKeyValueSet` writes plain values only; it has no `content_type`, so it cannot express a
Key Vault reference. Two routes exist:

| Route | Used by | Handles secrets |
| --- | --- | --- |
| Pulumi `AppCfgKeyValueSet` | LanguagePack, Solver, CognitiveSearch | ❌ plain values only |
| Hook + `config_interface.set_config(secret_id=…)` | GenericSync | ✅ writes a `SecretReferenceConfigurationSetting` |

GenericSync's `generic_sync_appconfig_envvariables.py` is the precedent for the secret case —
it maps `braidApiConfig__username`/`password` to Key Vault secret names and writes references.

**We already have what is needed:** `apim_hmac_secret.py` sets
`self.secret_uri = vault_info.get_secret_id(KEY_VAULT_SECRET_NAME)` and already stores the
secret in the spoke vault. The write target changes; nothing about secret generation does.

### What this fixes

| Problem | Why it goes away |
| --- | --- |
| **G42** — 403, no permission to write the variable group | Nothing is written to Azure DevOps |
| **G40** — Core takes its copy before APIM writes it | Nothing is copied; Core reads the live value |
| Rotation requiring a Core redeploy | Reference is unversioned, so a rotated secret is picked up on next start |
| Secret existing in plain text in a variable group | It only ever lives in Key Vault |

### The one real constraint

Settings are read at **application startup**, not continuously — there is no `ConfigureRefresh`
for ordinary settings (only feature flags, and their interval is set to 365 days), and
`DependencyConfig.cs:81` registers the configuration as a singleton *"to keep settings
consistent for the lifetime of the app"*.

So the requirement changes from *"APIM must deploy before Core's deploy step reads the variable
group"* to the much weaker *"the value must be present before Core next starts"*. If APIM is
deployed to an environment where Core is already running, **Core needs a restart** to pick the
secret up. Worth stating in the ticket rather than discovering later.

### Prerequisites to confirm per environment

App Config is skipped silently if these are missing — `GetAppConfigEndpoint()` returns null and
the extension does `return builder`:

- `AzureAppConfigEndpoint`
- `ClientId`
- `EnvironmentShortId`

Core sets all three on both hosting models (`core_vm.py:216` for the legacy VM,
`core_app_service.py:52` for the app service), so this should hold — but a missing endpoint
fails **silently**, which is the same trap as G39 and G40. Assert it rather than assume it.

---

## G44 — a Key Vault reference is only half the job; the reader needs vault access, and on Dreich only APIM has it

G43 established that Core resolves Key Vault references itself, so we can hand it a **pointer** to
the signing secret rather than a copy. That is the right shape, but it introduces a dependency that
the variable-group route never had: **whoever reads the setting must be allowed to open the vault.**

App Configuration hands back the reference and stops there. The application follows it using its own
identity. If that identity lacks `Key Vault Secrets User`, the setting resolves to **nothing** — and,
in keeping with the rest of this list, does so quietly.

### Access is by group membership, not role assignment

This is the part that catches people. Our IaC does not create a role assignment on the vault. It adds
the principal to an Entra group named after the vault:

```
{vault_name}-app-key_vault-secrets_user
```

The group holds the role; membership grants it. `get_kv_access`
(`Modules/KeyVault/key_vault_security.py:80-88`) looks the group up by name and creates an
`entra.GroupMember`. So "does X have access?" is answered by listing that group's members, **not** by
listing role assignments on the vault — which will only ever show the group itself.

### What is actually true on Dreich

| Party | Needs vault read because | In the group? |
|---|---|---|
| APIM's managed identity | it fills its named value from the vault entry | **Yes** |
| The Dayshape VM (`dev-drch-ds-uks-lwr-dv-1-vm`) | it resolves the App Configuration reference at startup | **No** |

The VM is in `ds-int-pfm-config-readonly`, so it can read App Configuration and *will* receive the
setting. It simply cannot follow the reference to the vault. That is the exact gap the spike closes by
adding `get_kv_access` to `CoreLegacyVM`.

### ⚠️ Correction — "the group is empty" was wrong, and the way it was wrong matters

An earlier version of this entry stated the group had **zero members** and that APIM was therefore
missing from it. That was false. Both `az ad group member list` and Graph `transitiveMembers` return
an **empty list** for this group — no error, no warning — while asking the same question from the
other direction gives the opposite answer:

```
az ad group member check --group drchdvspkstrkv-app-key_vault-secrets_user \
    --member-id <apim-principal-id>          -> { "value": true }

POST /servicePrincipals/<apim-principal-id>/checkMemberGroups
                                             -> [ "<secrets_user group id>" ]
```

A principal cannot belong to a group with no members, so one of the two reads is being filtered. The
member *listing* is restricted for our accounts; the membership *check* is not.

The uncomfortable part: this entry already warned that "an empty result is indistinguishable from a
permissions problem on the caller" — and then drew the conclusion anyway, from two reads that shared
the same blind spot. Repeating a query is not corroboration if both attempts ask the same question of
the same restricted view.

**Use `az ad group member check` or `checkMemberGroups`, never a member listing, to answer "does X
have access?" in this directory.** Listing tells you nothing either way.

### Not every Core hosting model grants itself access

There are three, and they do not agree:

| Variant | Grants `Key Vault Secrets User`? |
|---|---|
| `CoreVM` (modern) | **Yes** — `core_vm.py:287`, via `_add_kvsync_to_vm` |
| `CoreLegacyVM` (**what Dreich runs**) | **No** — adopts a pre-existing VM via `VirtualMachine.get(...)`, so nothing in its own code grants anything |
| `CoreAppService` | **No** — language-pack read, App Config read and ACR pull only (`core_app_service.py:124-145`) |

Reading `CoreVM` and concluding "Core has vault access" is wrong for two of the three. The spike adds
the same `get_kv_access` call to the other two so all three behave alike.

### The lesson

A Key Vault reference moves the problem rather than removing it. The secret is no longer copied —
good — but the reader now needs a permission it may not have, granted through a mechanism that is not
where you would look for it, in a component that may not be yours. Check the **group membership**, on
the **specific hosting variant** in use, before assuming the reference will resolve.

---
---

## G45 — there are two ways to write an App Configuration setting, and only one of them is owned by Pulumi

Having established (G43, G44) that the right way to reach Core is a Key Vault reference in App
Configuration, there is a second decision hiding behind it: **how** that setting gets written. The
repo offers two routes and they look interchangeable. They are not.

| | Imperative — `app_cfg_interface.config_interface` | Stateful — `AppCfgKeyValueSet` |
|---|---|---|
| What it is | A direct call to the Azure SDK | A `pulumi.ComponentResource` |
| Runs from | A deployment hook, after Pulumi has finished | The Pulumi program itself |
| In Pulumi state | **No** | **Yes** |
| Removed on destroy | Only if you hand-write the cleanup | **Automatically** |
| Drift detected | Never | **Yes** |
| Can be silently skipped | **Yes** — see G40 | No |
| Auth needed | Management-plane `ListKeys` on the store | ARM write on `configurationStores/keyValues` |

The trap is that **LanguagePack uses both**, so whichever one you happen to read first looks like the
house pattern. Its *hook* uses the imperative route; its *component* uses `AppCfgKeyValueSet` to write
`spk-core-software-LanguagePackStore-*` — keys in **Core's** namespace, from a spoke component, via a
provider scoped to the platform subscription. That second one is the pattern to copy.

### Why it matters more than it looks

Choosing the hook meant hand-writing a `_clear_hmac_secret` for the destroy case, and hand-writing an
`_environment_has_core` check to decide whether to write at all. Both are code that exists purely to
reimplement what Pulumi already does. The `_environment_has_core` check is where G40 came from: it
looked only for `Core`, missed the `CoreLegacyVM` hosting variant Dreich actually runs, skipped the
setting without complaint, and left every request through the gateway failing with 403. Deleting the
branch deletes the bug class.

It also breaks a stated repo rule — *"Pulumi components should not interact with Az APIs directly"*
(`copilot-iac-development.instructions.md`).

### The two routes produce an identical setting

Worth knowing before switching, because it makes the change low-risk. The Azure SDK's
`SecretReferenceConfigurationSetting` writes exactly:

```
content_type: application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8
value:        {"uri": "https://<vault>.vault.azure.net/secrets/<name>"}
```

and `pulumi_azure_native.appconfiguration.KeyValue` accepts both fields. So this is a change of
mechanism, not of outcome — the bytes Core reads are the same either way. `AppCfgKeyValueSet` did not
previously expose `content_type`; the spike added an optional `secret_id` argument to it that emits
both fields.

### The lesson

"Write a setting into App Configuration" is not one decision, it is two. Ask *who owns this
afterwards* before writing the code. If the answer is not Pulumi, you have signed up to hand-write
creation, deletion and drift handling — and the skip conditions that go with them.


## Related

- [[APIM IaC - portal content, why there is no IaC-native route]] — why the script is the only complete option

- [[APIM Findings Summary]]
- [[APIM IaC - portal content permissions (phase 10)]] — full detail behind G27, and the grant needed
  to make Phase 10 work in production
- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — source of G15–G18
- [[APIM IaC - outstanding questions]] — open questions needing someone else's answer
- [[APIM IaC - ticket breakdown]] — T3 carries the fixes for G1, G6 and G7
- [[APIM IaC - HMAC signing secret decision (phase 5)]] — G7's failure mode (a secret that expires with no
  owner) is avoided in Phase 5 by having nothing that expires
- APIM spike - clean redeploy runbook — failure modes seen during redeploys
- Slack `#project-api-voodoo`, 16–17 June 2026 — Jonathan Lozinski asks how portal content is
  maintained across many instances; Laszlo Szegedi's reply is the source of G24

---

## G46 — a test assertion inside a Pulumi Output can pass by never running, and you cannot catch it failing

`assertPulumiOutputCustom` — the helper every component test uses to assert on a value that only
exists once Pulumi resolves it — is, in full:

```python
return output.apply(assertion_function)
```

It does not await. The assertion is handed to Pulumi and runs later, during `wait_for_rpcs()` at the
end of the test. Two consequences, both of which bit while building the APIM alert rules.

### 1. You cannot wrap one in `assertRaises`

This looks like a reasonable guard — *prove the assertion machinery actually bites* — and it does not
work:

```python
with self.assertRaises(AssertionError):
    self.assertPulumiOutputCustom(output, lambda q: self.assertIn("not in the query", q))
```

The `AssertionError` **is** raised, but after the `with` block has already exited, so `assertRaises`
never sees it. The test fails with the raw error rather than passing. Anything relying on catching a
failure from inside an Output callback needs restructuring to be synchronous instead.

The good news buried in that failure: assertions inside Output callbacks genuinely do run and
genuinely do fail the test. They are not silently skipped.

### 2. Mock captures are only populated if you anchor correctly

The common pattern is to register a resource mock that stashes the resource's inputs into a list, then
assert on that list inside an Output callback:

```python
captured = [{}]
self.add_pulumi_new_resource_mock("azure-native:monitor:MetricAlert", lambda o: captured.__setitem__(0, o))
sut = self._get_sut()
self.assertPulumiOutputCustom(some_output, lambda _: self.assertEqual(captured[0].get("name"), ...))
```

Whether `captured[0]` is populated depends entirely on **which Output you anchored on**. If
`some_output` resolves before the resource you are capturing has been registered, you assert against
an empty dict and get `AssertionError: None != 'Capacity'` — which reads like the resource was never
created, when in fact it was created a moment too late. The existing web-test assertion in
`apim_spike_component_unit_test.py` carries a comment about this for exactly that reason: it anchors
on the web test's *own* id.

### The way round both

Where a wrapper class keeps its configuration on `self`, assert on that directly. `MetricAlert` stores
`name`, `resource_group`, `scopes`, `criteria`, `window_size`, `evaluation_frequency` and `severity`,
so:

```python
capacity = sut.alerts.capacity_alert
self.assertEqual(capacity.criteria.all_of[0].metric_name, "Capacity")
self.assertEqual(capacity.window_size, "PT10M")
```

Same values, no mock, no Output, no ordering dependency. Note `LogSearchAlert` does **not** keep the
created resource or its criteria on `self` in the same way — for the log rules the query text was
exposed deliberately on the component's own return value (`ApimAlerts.queries`) so tests had something
stable to read.

**Rule of thumb:** reach for `assertPulumiOutputCustom` only when the value genuinely cannot exist
until Pulumi resolves it. If the same information is available synchronously, assert on that instead.

### Related

- **G22** — the same class of problem from the other direction: no resource handle at all, so the
  assertion has to anchor on the next resource created, making construction order load-bearing.
- `Products/Spoke/ApimSpike/tests/apim_spike_component_unit_test.py`
- `Products/Modules/Alerts/metric_alert.py`, `Products/Modules/Alerts/log_search_alert.py`

---

## G47 — a time filter inside alert KQL is a second look-back window, and only one of the two is reachable from config

Every one of the six APIM log-alert queries was written with a time filter of its own:

```kql
ApiManagementGatewayLogs
| where TimeGenerated >= ago({window_minutes}m)
```

while the alert rule that runs it was built with a separately hardcoded `window_size="PT5M"`.

Both are the same concept — *how far back does this rule look* — but they are set in two different
places, and only the query one was wired to config. Raise `window_minutes` to 15 and you get a query
reading 15 minutes of data inside a rule that only evaluates 5. Lower it and the rule silently
ignores most of its own window. **The rule looks tuned and is not.**

### Why this is easy to write and hard to spot

Written on its own, `| where TimeGenerated >= ago(5m)` looks like good defensive KQL — it is what
you would type in Log Analytics to test the query by hand. The duplication only exists once the
query is attached to a scheduled rule, and nothing in Azure warns about it. The rule still deploys,
still evaluates, and still returns rows.

### What the estate does instead

**No sibling component's KQL filters on time at all.** Checked directly:

- All three `Spoke/CognitiveSearch/infra/kql/*.kql` files substitute only `{search_name}`.
- `Modules/ContainerApp` queries receive only `{name}`.

The rule's `window_size` alone governs the range of data the query sees. That is what lets
CognitiveSearch run the *same query shape* at `PT5M` for one rule and `PT15M` for another, changing
nothing but the rule.

> [!tip] The fix is a deletion
> Remove the `ago(...)` line. There is then exactly one window setting, it is the one the platform
> already exposes as `alert-<rule>-window_size`, and the drift cannot occur.

### Guard

`test_queries_carry_no_time_filter_of_their_own` in
`Products/Spoke/ApimSpike/tests/apim_spike_component_unit_test.py` fails if `ago(` reappears in any
alert query.

### Related

- Found while bringing the alert config onto the estate `alert-<rule>-<field>` convention —
  [[APIM IaC - diagnostics and monitoring (phase 8)]] §14.
- The wider lesson: **when a setting exists in two places, check which one config actually reaches.**
  Same shape as **G45**, where two ways of writing an App Configuration setting looked equivalent and
  only one was owned by Pulumi.

---

## G48 — APIM should be deployed to West Europe, but the code defaults to the hub's region

**Added:** 2026-10-09. **Status:** 🟠

The team's intended region for APIM is **West Europe**. The code does not hard-code that. It reads the
`apim-location` setting (set once when the APIM is first created) and, if nothing is set, falls back to
the **hub's** region. After creation the location is ignored (`ignore_changes=["location"]`), so an
APIM cannot be moved by changing the setting later.

**What to do:** for every new environment, set `apim-location` to `westeurope` explicitly rather than
relying on the hub default, and check the result in the portal after the first deploy. If the APIM comes
up in another region, the only fix is to destroy and rebuild it (see G37 for the name-hold and purge
steps).

Why it matters: the sandbox was first built in West Europe because UK South refused the capacity, and
the data-residency question for the gateway's region is still open (see D8 and DEVO-30006).
