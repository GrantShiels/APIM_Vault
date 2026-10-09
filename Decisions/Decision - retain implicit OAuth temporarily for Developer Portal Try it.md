---
tags: [apim, iac, decision, auth, oauth, security, devo-28373, devo-30734]
date: 2026-09-30
status: interim
decided-by: "Grant Shiels — continue the existing production-compatible approach for now"
replaces: nothing
replacement-ticket: DEVO-30734
related-pr: 8288
---

# Decision — retain implicit OAuth temporarily for Developer Portal Try it

> [!abstract] The decision in one box
> Keep the Developer Portal **Try it** console on the implicit OAuth flow for the current APIM IaC
> delivery. This produces the ID token that the existing Dayshape API authentication path accepts
> and matches the approach already used for Kannua and the APIM Setup Guide.
>
> This is a **temporary compatibility decision**, not the target authentication design. Implicit is
> deprecated, and an ID token proves sign-in rather than granting access to an API.
> [[https://airtsdev.atlassian.net/browse/DEVO-30734|DEVO-30734]] is the replacement work: it must
> introduce a supported flow, remove the standard implicit-flow path, and prove the new access token
> works through both APIM and Core.

---

## Why this decision was revisited

During [[https://airtsdev.atlassian.net/browse/DEVO-28373|DEVO-28373]] / 
[[https://github.com/dayshape/dayshape/pull/8288|PR #8288]], the APIM OAuth authorization server was
configured to match the spike and Setup Guide:

- grant type: `implicit`
- scopes: `openid profile email api://<core-client-id>/entities_access_as_user`

Testing on `grant-sbx` established that the managed Developer Portal sees `openid` and asks Entra
for an **ID token**. It then sends that token as the API bearer. The token had:

- the Developer Portal application's client ID as `aud`
- no `scp` claim
- a `nonce`
- the expected `tid`, `oid`, and token version

That cannot satisfy the access-token contract originally written into DEVO-28373 and DEVO-31025.
The first reaction was to switch Try it to authorization code and request only the Entities API
scope.

Further Slack research found why the existing setup uses implicit and why Kannua has the same
configuration.

---

## The historical reason

This was a deliberate compatibility workaround, although that context was lost when the settings
were copied into the APIM Setup Guide.

In December 2025, the API project established:

1. APIM normally obtains and sends an **access token**.
2. The existing Dayshape API authentication path expected an **ID token**.
3. Voodoo Park found that configuring the Developer Portal with implicit OAuth and OpenID scopes
   caused it to send the ID token that Dayshape accepted.
4. This unblocked delivery, but everyone involved described it as temporary and deprecated.

Laszlo's summary was explicit:

> Finally I was able to configure the APIM to use the `id_token` and it was accepted by the
> reporting API. So this can work as a temporary solution. But I had to use the deprecated implicit
> authentication flow so it cannot be a long-term solution. In my opinion the recommended direction
> would be to modify the reporting API to use the access token.

Keir explained that implicit was being used to "trick" APIM into sending the ID token because that
was what Dayshape expected. The proper fix was either a custom authentication component or changing
Dayshape to accept access tokens.

The Setup Guide later recorded the working settings but not this rationale. The APIM IaC spike then
faithfully copied the guide.

---

## What is being kept for now

For the current APIM IaC delivery:

- Keep the APIM OAuth authorization server on the implicit grant.
- Keep the OpenID scopes that cause the managed portal to produce the existing-compatible ID token.
- Keep the implicit callback and token-issuance settings needed by that flow.
- Continue treating portal sign-in and Try it as separate configurations, even though they currently
  use the same temporary token shape.
- Do not weaken future documentation by describing the ID token as an API access token.

This keeps the IaC aligned with:

- the currently working Kannua configuration
- the manual APIM Setup Guide
- the spike proven on Dreich
- the authentication behaviour existing Dayshape APIs currently accept

---

## What this decision does **not** mean

It does not mean:

- implicit OAuth is considered secure or future-proof
- an ID token is the correct authorization credential for a web API
- the new `entities_access_as_user` delegated scope is proven through Try it
- DEVO-31025 can validate Core audience plus `scp=entities_access_as_user` while the portal still
  sends the current ID token
- the temporary flow should be copied into new products once the supported replacement exists

The distinction matters:

| Token | What it establishes | Current use |
| --- | --- | --- |
| **ID token** | The user signed in to a particular client application | Temporary Try it compatibility |
| **Access token** | The caller has delegated permission to call a particular API | Target design under DEVO-30734 |

---

## Effect on the current tickets

### DEVO-28373 / PR #8288

The implementation can keep implicit so the present Dayshape API path remains usable, but the
ticket and PR must not claim that Try it obtains a scoped Core access token.

The evidence should instead state:

- portal sign-in works
- Try it obtains and forwards the existing-compatible ID token
- this is an intentional temporary workaround
- the access-token acceptance criteria are deferred to DEVO-30734

This is a scope/acceptance-criteria change and should be visible on the ticket and PR rather than
hidden in code comments.

### DEVO-31025

DEVO-31025 currently expects:

- `aud` for Core
- `scp=entities_access_as_user`
- runtime JWT validation before using `tid` and `oid`

Those checks would reject the token emitted by the temporary Try it flow. Therefore DEVO-31025
must not be implemented against its current contract until one of these is true:

1. DEVO-30734 has delivered the supported Core access-token flow, or
2. DEVO-31025 is explicitly split so it does not block the current temporary portal flow.

It must not be weakened to treat a portal ID token as API authorization merely to preserve the
temporary setup.

### DEVO-30734

[[https://airtsdev.atlassian.net/browse/DEVO-30734|DEVO-30734]] directly replaces this decision.
Its linked requirement, JPD-1157, requires:

- a supported OAuth flow for Developer Portal/API usage
- removal of implicit from the standard path
- removal of guest-user onboarding as the default path
- support for the agreed multi-SSO approach
- gateway authentication plus Core entity/action authorization
- repeatable setup and support documentation

This is not an unrelated future improvement. It is the planned completion of the authentication
work that the temporary APIM flow deliberately leaves unfinished.

---

## Exit criteria for this temporary decision

Implicit can be removed when DEVO-30734 demonstrates all of the following end to end:

1. An external developer can sign in using the agreed supported identity route.
2. Developer Portal Try it obtains an access token intended for the agreed API.
3. The token has the agreed audience, tenant, identity, and delegated scope claims.
4. APIM validates the token and returns `401` for missing or invalid authentication.
5. Core accepts the token, resolves the caller, and enforces entity/action permissions.
6. An authenticated caller without permission receives `403`.
7. Existing direct API consumers and the Kannua path have an explicit migration or compatibility
   plan.
8. The APIM Setup Guide and IaC documentation have been updated to remove the implicit workaround
   from the standard setup.

Until these are proven, changing the IaC to authorization code would risk producing a technically
correct access token that the existing Dayshape API path cannot use.

---

## Risks accepted for the interim

| Risk | Current position |
| --- | --- |
| Implicit OAuth is deprecated and returns tokens through the browser redirect | Accepted temporarily; DEVO-30734 is the exit |
| Try it uses an ID token as a bearer token | Accepted for compatibility, not described as proper API authorization |
| The token has the portal client as its audience and no API `scp` | Runtime validation must not pretend otherwise |
| Customer developers currently rely on Dayshape Entra guest access | Existing interim approach; DEVO-30734 replaces it |
| The Setup Guide presents the workaround without its historical warning | Documentation follow-up required |
| Future engineers may mistake "working" for "finished" | Mitigated by this decision record and explicit ticket links |

---

## Validation required before the current PR merges

Even while retaining the temporary approach, PR #8288 should still prove:

- the Developer Portal sign-in works
- Try it obtains a token and sends it to the API
- a real Entities API request succeeds in an environment with Core
- the observed token shape is documented by claim name only
- no full token is placed in GitHub, Jira, Slack, logs, or screenshots
- the PR description names DEVO-30734 as the supported-flow replacement

A green infrastructure deployment is not enough. Previous APIM spike runs were green while API
requests were unusable.

---

## Ownership and follow-up

| Work | Owner |
| --- | --- |
| Current temporary IaC implementation | DEVO-28373 / Ecosystem |
| Supported auth/SSO replacement | DEVO-30734 / William Robertson, with Application Platform |
| Runtime token validation and trusted rate-limit identity | DEVO-31025, sequenced after the supported token contract |
| Security/architecture approval | Required by DEVO-30734 |
| Setup Guide correction | Assign during DEVO-30734 or rollout/handover |

---

## Sources

- [[https://airtsdev.atlassian.net/browse/DEVO-28373|DEVO-28373]]
- [[https://github.com/dayshape/dayshape/pull/8288|PR #8288]]
- [[https://airtsdev.atlassian.net/browse/DEVO-30734|DEVO-30734]]
- [[https://airtsdev.atlassian.net/browse/JPD-1157|JPD-1157]]
- [[https://airtsdev.atlassian.net/browse/DEVO-31025|DEVO-31025]]
- Slack `#project-api-voodoo`, thread started 2 December 2025 — APIM access-token versus
  Dayshape ID-token compatibility discussion
- Slack `#app-platform`, 15 December 2025 — confirmation that implicit was used to obtain the ID
  token Dayshape expected
- [[APIM IaC - authentication (phase 4)]]
- [[Decision - customer developer accounts in our Entra directory]]

