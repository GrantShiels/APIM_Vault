---
tags: [apim, iac, meeting-notes, devo-28368]
updated: 2026-08-05
date: 2026-08-05
attendees: [Grant Shiels, Richard Donkin, Marcus]
related-ticket: DEVO-28368
---

# Meeting Notes — Richard & Marcus, 2026-08-05

> Four things came out of this that affect how the component gets
> built. Richard also ran the IaC code review skill against the branch — findings are tracked
> separately (see the bottom of this note).

---

## 1. Config model — where values belong

Agreed split for anything configurable in the component:

| Where | What goes there |
| --- | --- |
| **App Config** | Values that are **dynamic** — change per environment, per deploy, or over time |
| **GSOT** | Values that **do not change** and are **shared across multiple components** |
| **Variables in code** | Everything else — the plain constants we have been defining as we go |

**What this means for us.** ApimSpike currently has **no `config/` directory at all** — everything is
either component config or hardcoded in the helpers. The IaC review flagged the missing directory
independently. Before this leaves spike status, every value needs placing deliberately against the
three buckets above rather than defaulting to "hardcode it".

Worth doing as one pass rather than piecemeal, because the SKU, location, CORS origins, rate limits
and API version all need the same judgement applied.

**Tracked as:** `mtg-config-model`

---

## 2. Always check the official docs before building

Standing instruction, and one we have already been bitten by twice.

- **APIM** — https://learn.microsoft.com/en-us/azure/api-management/
- **Pulumi azure-native registry** — https://www.pulumi.com/registry/packages/azure-native/

**Why this matters concretely.** Two of the problems we have hit this week were both discoverable in
the docs before writing a line:

- `portalsettings` vs `portalconfigs` — the v2 tiers only run the current developer portal, which is
  documented. We found it by deploying and failing (G11).
- `enable_client_certificate` — the ARM reference states verbatim that it is *"only meant to be used
  for Consumption SKU Service"*. We set it on BasicV2 and it silently did nothing.

Six phases remain unbuilt. Every one of them should start with the docs, not the guide alone — the
guide was written against a classic tier and cannot be assumed correct for BasicV2.

**Tracked as:** `mtg-docs-check`

---

## 3. Client secrets — use Key Vault, and get product involved

Richard's note, close to verbatim:

> *For client secrets, we use Key Vault. We will need to talk to product about it. When we generate a
> client secret and then add it to the APIM — how can we generate a secret on setup? I should make a
> document around setting it up. Look at the service principal component. We should phrase the
> question very clearly and get a write-up of what the issue is and what the pros and cons are. Look
> at the standard pattern for this.*

**Done** — written up as [[APIM IaC - client secret decision]].

**Headline finding from doing that write-up:** the question is not "who types the secret in". Dayshape
already **generates** secrets in automation — two working patterns exist in the repo:

- `Modules/ServicePrincipal` — generates with `azuread.ApplicationPassword`, **stores in Key Vault**
- `Spoke/AiAgent` — generates and **consumes it in the same run**, never stores it

The real question is **who owns the credential**, because APIM has no app registration of its own —
it borrows Core's, which Auth owns. Four options are set out in the document; **Option 1 (Auth
generates it, stores in Key Vault) looks strongest** because it keeps a single owner and is not
blocked behind the unresolved token question.

**Tracked as:** `mtg-secret-doc` · **Register entry:** Q5 in [[APIM IaC - outstanding questions]]

---

## 4. IaC code review skill

Richard ran the [IaC code review skill](https://github.com/dayshape/dayshape/blob/iac-main/.github/skills/iac-code-review/SKILL.md)
against the branch at `a415b19`. It is now available locally as `.github/skills/iac-code-review`
following the develop merge.

**Outcome: 5 Major, 4 Minor, no blockers.** Full output kept in the session files as
`review-04.08.md`.

Two notes Richard added by hand that are worth remembering for next time:

- Tell the skill that **infra deployed by earlier code can be destroyed**, since this is a spike —
  otherwise it generates irrelevant migration/compatibility comments.
- The **`interface/` script-plus-`local.Command` approach is not liked**. Richard: *"I'd hope there's
  a better way of doing this directly in Pulumi, even if we import the default resources then delete
  them… Generally we try hard to avoid direct use of Azure API."* This aligns with the dynamic
  provider rework already on the backlog.
- The Copilot instruction that steered us away from Azure CLI **needs fixing** — it was only ever
  meant to apply to interactive AI use, not to generated code.

**Findings tracked as:** `rv-client-cert`, `rv-shared-helpers`, `rv-cors`, `rv-networking`,
`rv-patch-mock`, `rv-readme`, `rv-api-version`, `rv-misc`

---

## Related

- [[APIM IaC - client secret decision]]
- [[APIM IaC - outstanding questions]]
- [[APIM IaC - known traps and gotchas]]
- [[APIM IaC - ticket breakdown]]
