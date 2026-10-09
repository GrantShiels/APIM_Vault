---
tags: [apim, iac, test-setup, data-points, reference, devo-28368]
updated: 2026-07-16
status: living-doc
source: "APIM Setup Guide (Prerequisites + Variables) — pageId 4999643160"
env: dev-dreich
---

# APIM test setup — my data points

> **Living reference** of the concrete values I'm using while
> working through the [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide)
> for my **Dreich (dev)** test. Mirrors the guide's **Prerequisites** + **Variables** tables.
> **Update as I go.** ✏️

> [!warning] Do NOT paste the actual client secret here
> `<idp-client-secret>` is a **real secret** and this vault is not a secrets store. Record only
> **where it lives** and **when it expires** — keep the value itself in a password manager / the
> secure place the process uses. See the gotcha note below.

## Environment

| Field | Value |
| --- | --- |
| Target environment | **Dreich** (internal dev) |
| `<env-name>` | `dev` |
| `<client-name>` | `drch` *(confirm — from resource naming `dev-drch-spk-apim`)* |
| APIM service | `dev-drch-spk-apim` |
| APIM resource group | `dev-drch-spk-apim-rg` |
| APIM subscription ID | `32320e53-28cb-46a6-ba9e-0a35a1c8987c` |
| APIM link | [Azure portal](https://portal.azure.com/#@dayshape.com/resource/subscriptions/32320e53-28cb-46a6-ba9e-0a35a1c8987c/resourceGroups/dev-drch-spk-apim-rg/providers/Microsoft.ApiManagement/service/dev-drch-spk-apim/overview) |
| `<apim-devportal-url>` | `https://dev-drch-spk-apim.developer.azure-api.net` *(confirm on APIM Overview blade — custom domain?)* |

> [!note] IaC PoC (Option B) — separate from this manual walkthrough
> The **IaC spike** (`Products/Spoke/ApimSpike`) creates its **own new, separately-named spike APIM**
> in the **Dreich subscription** (`32320e53-…`), reusing this **same Core app registration**
> (client id `7e65f249-…`, tenant `201e1d9f-…`) for identity. It must **not** touch/adopt the manual
> `dev-drch-spk-apim` above (adoption = separate Q1). Phase 4 in IaC adds *that new service's* portal
> redirect URIs to the Core app reg **additively** (existing URIs, incl. the manual `dev-drch` ones
> below, must be preserved). 4A needs the Core app reg **object id** —
> `az ad app show --id 7e65f249-82dc-4224-8ff4-15e8c0be2346 --query id -o tsv`.
> ~~Client secret is read from env var `APIM_IDP_CLIENT_SECRET`, never committed.~~
> **Out of date as of 2026-08-13** — the deploy creates its own sign-in secret
> (`apim-devportal-signin`) and rotates it. No env var, nothing to set by hand.
> See [[D1 - The developer portal sign-in secret]].

## Prerequisites (from the guide)

| Prerequisite | My value / status | Notes |
| --- | --- | --- |
| **(Spoke) Subscription** | `32320e53-28cb-46a6-ba9e-0a35a1c8987c` | Must match the one hosting Dayshape Core. ⏳ confirm Core is here too. |
| **Resource Group** | `dev-drch-spk-apim-rg` | Guide pattern: `<client-name>-<env-name>-spk-apim-rg`. |
| **Entra ID App Registration** | ✅ Client ID `7e65f249-82dc-4224-8ff4-15e8c0be2346`, Tenant ID `201e1d9f-2fb3-4cb1-9306-cb6c0dbefde9` | **Reuse the same app reg as Dayshape Core** — did NOT create a new one. |
| **Client secret in Entra** | ⏳ create `apim-auth`, 12-month expiry | **Value shown ONCE** at creation — copy immediately to secure store. See gotcha. |
| **Application Insights** | ⏳ TODO | Instance used for APIM logging. |
| **OpenAPI spec (Entities API)** | `Infrastructure/openapi.v1.json` (in `dayshape/dayshape`) | Regenerates each Core release; imports the API operations. |
| **Log Analytics workspace** | ⏳ TODO | Pattern `<ClientGroupId>-ds-<LocationId>-<Env>-sh-law` (existing one). |

## Variables (from the guide)

| Variable                | My value                                            | Where it comes from                                                                        |
| ----------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `<env-name>`            | `dev`                                               | —                                                                                          |
| `<client-name>`         | `drch` *(confirm)*                                  | —                                                                                          |
| `<rate-limit-number>`   | ⏳ TODO                                              | How many requests allowed per `<rate-limit-seconds>`.                                      |
| `<rate-limit-seconds>`  | ⏳ TODO                                              | Time window in seconds for the rate limit.                                                 |
| `<idp-tenant-id>`       | `201e1d9f-2fb3-4cb1-9306-cb6c0dbefde9`              | Core app reg **Overview → Directory (tenant) ID**.                                         |
| `<idp-client-id>`       | `7e65f249-82dc-4224-8ff4-15e8c0be2346`              | Core app reg **Overview → Application (client) ID**.                                       |
| `<idp-client-secret>`   | 🔒 in secure store (NOT here)                       | Core app reg **Certificates & secrets → new `apim-auth` secret → Value**. See image below. |
| `<hmac-signing-secret>` | ⏳ TODO *(optional)*                                 | Must match the secret used by Dayshape Core.                                               |
| `<apim-devportal-url>`  | `https://dev-drch-spk-apim.developer.azure-api.net` | APIM Developer Portal URL (confirm on Overview).                                           |

### Where to get the client secret Value

On the Core app registration → **Manage → Certificates & secrets → Client secrets** tab, the
`apim-auth` secret's **Value** (with the copy icon) is what you need — **not** the *Secret ID*.

*(screenshot missing from vault)*

> [!warning] This screenshot shows a (partial) real secret Value
> The **Value** column here is a live credential. Don't share this note/image outside your secure
> context, and treat the Dreich `apim-auth` secret as **exposed** — consider **rotating it**
> (delete + recreate) once setup is done, since it's been captured in a screenshot.

## Redirect URIs added in 4A (for reference)

Added to the **Core Dayshape app registration** (Manage → Authentication):

**Single-page application:**
- `https://dev-drch-spk-apim.developer.azure-api.net/`
- `https://dev-drch-spk-apim.developer.azure-api.net/signin`

**Web:**
- `https://dev-drch-spk-apim.developer.azure-api.net/signin-oauth/implicit/callback`
- `https://dev-drch-spk-apim.developer.azure-api.net/signin-oauth/code/callback/entra-id`

Status: ⏳ TODO / ✅ done — *(update)*

## Gotchas I hit (guide rough edges)

- **No separate "APIM app registration"** — 4A means the **existing Core Dayshape app registration**.
  There's no app to create; that's the "missing step" that confused me. `<idp-client-id>` /
  `<idp-tenant-id>` both come from **that** app reg's Overview.
- **App registration may sit in a different subscription** from APIM — guide says *"switch to the
  subscription where the Entra ID sits"*.
- **`<idp-client-secret>` is created, not found** — add a **new** client secret (`apim-auth`,
  12-month) on the Core app reg. The **Value** is shown **only once** at creation and can never be
  retrieved again — copy it immediately. The **Secret ID** column is NOT the secret.
- **Shared app reg = care needed** — only add my `apim-auth` secret; don't touch existing secrets.
  The 12-month expiry is a future **rotation** task (day-2 / SRE handover concern).

## Related


- [[Meeting Notes - Nick Wright IaC walkthrough]]
- [[Team skills vs APIM delivery - risk assessment]]
