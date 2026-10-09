---
tags: [apim, iac, decision, auth, security, devo-28368]
date: 2026-08-19
status: decided
decided-by: "Gabriela (Product) — carry on with the current approach"
source: "Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07; verified against branch code 2026-08-19"
---

# Decision — customer developer accounts live in Dayshape's Entra directory

> [!abstract] The decision in one box
> Customer developers who need the API documentation site are given an account in **Dayshape's own
> Entra directory**. The documentation site accepts sign-ins from our directory **only**, and the
> built-in "username and password" option is deliberately switched off.
>
> Gabriela originally flagged this as something we didn't want to do. She has since confirmed we
> **carry on with this approach** for the IaC work. It is also what the code on the branch already
> does — so this decision records and confirms existing behaviour rather than changing it.

---

## What was actually decided

Two things, and it's worth keeping them apart:

| | |
| --- | --- |
| **Settled** | Whether we're *willing* to put customer developers in our directory. Yes — Gabriela has confirmed we carry on. |
| **Not settled** | Whether it actually *works* for a real customer. Never tested. Two known gaps below. |

Gabriela's confirmation removes the product objection. It does not remove the mechanical problems,
and those are the ones that will bite during delivery.

---

## Why we're constrained — the chain

This isn't a preference. It's the end of a chain where each link removes an option:

1. **The documentation site can't have its own identity.**
   Dayshape's product only accepts a sign-in token stamped with its *own* identity. That stamp is
   fixed by the sign-in protocol — it's always the identity of whoever requested the token. If the
   documentation site had its own registration, the token would carry the documentation site's
   stamp and the product would reject it as a stranger.

2. **The obvious workaround is already used up.**
   The product has exactly **two** slots for plugging in outside sign-in providers. Both are
   already taken by real customer sign-in providers. There's no third slot without a change to the
   product itself.

3. **So the documentation site has to borrow Dayshape's main sign-in registration.**
   That's what the code does.

4. **That registration only admits explicitly approved people.**
   It's configured so that nobody can sign in unless they've been added to an approved group.
   Today that's the *All Dayshape Users* group plus the deployment account.

5. **Therefore, to let a customer developer in, someone has to put them in a group in our directory.**

Every awkward thing about documentation site sign-in traces back to step 1.

---

## What the code actually does

Verified against the branch on 2026-08-19, not from memory.

**The sign-in provider is pinned to our directory** — `apim_spike_component.py` (Phase 4C):

```python
client_id=core_app.client_id,                              # borrows the product's registration
signin_tenant=global_variables.dayshape_tenant_id,
allowed_tenants=[global_variables.dayshape_tenant_id],     # our directory, and only ours
```

`allowed_tenants` is the important one. It is a hardcoded single-element list, not driven by
config. A customer signing in with their own company account would be refused.

**The alternative way in is deliberately removed** — Phase 4D deletes the built-in "username and
password" identity from the documentation site. Without that, anyone could self-register with any
email address and password they liked. With it removed, a directory account is the *only* route in.

Together those two facts mean the decision isn't really open in the current build — it's already
made in code.

---

## Options considered

| Option | What it would mean | Why not (for now) |
| --- | --- | --- |
| **Customers use their own company accounts** | Customer developers sign in with their existing work identity. No accounts in our directory. | Blocked twice over — the sign-in provider is pinned to our directory, *and* the product would need to accept outside identities. Needs an infrastructure **and** a product change. |
| **A separate product built for external users** (Entra External ID) | Proper separation between staff and customers. | Blocked by the same product constraint at step 1. Clean in principle, not reachable today. |
| **Keep the username/password option** | No directory accounts needed at all. | Anyone could self-register unverified with any email address. Rejected on security grounds. |
| **No documentation site sign-in at all** | Publish the docs some other way — public page, PDF, in-product. | Still genuinely viable. The API works with or without documentation site sign-in, so this loses less than it sounds. Worth keeping on the table. |
| ✅ **Accounts in our directory** | What we do today. | Chosen — it's the only option that works without a product change. |

---

## What this means in practice

- **External people sit in the same directory as Dayshape staff.** That's the thing Gabriela
  didn't like, and the objection is reasonable even though we're proceeding.
- **Somebody has to add each customer developer to an approved group.** Manual, ongoing, and it
  has **no owner assigned**. This needs a name against it before the first customer.
- **The currently approved group is `All Dayshape Users`** — which is almost certainly the wrong
  group for external people. A separate group for API consumers is likely needed, and that's a
  change to a shared, security-sensitive registration, so it needs sign-off rather than a quiet fix.
- **Offboarding matters as much as onboarding.** When a customer developer leaves their company,
  nothing in our process currently notices. Worth deciding who owns removal.

---

## What's still unproven ⚠️

Both of these are real, both are in the code today, and **neither has ever been exercised by a
genuine customer account** — everything we know comes from Dayshape staff using Dayshape accounts.

### 1. The documentation site asks for something it was never given permission to ask for

The site requests three things about the user when they sign in:

```python
default_scope="openid profile email"     # apim_spike_component.py, Phase 4B
```

But the sign-in registration was only ever granted — and only ever consented for — the first two:

```python
scope_ids=[openid, profile]              # auth_component.py, API access
claim_values=["openid", "profile"]       # auth_component.py, admin consent
```

`email` is missing from both. The likely result is a consent prompt the user can't get past, or an
outright sign-in failure. An earlier Slack thread listed `email` **and** `User.Read` as
prerequisites; neither is present.

The fix is small — add the missing permissions — but the registration is **shared with the main
product**, so this is a sign-off change, not a unilateral one.

### 2. Only explicitly assigned people can sign in at all

The registration requires assignment before anyone can sign in:

```python
app_role_assignment_required=True        # auth_component.py
```

When it refuses someone, the error is a generic "not assigned to a role" message that reads like a
broken configuration rather than a missing group membership. Expect this to be misdiagnosed the
first time it happens.

---

## What reduces the risk

Two things make this less alarming than it first reads:

- **Customer developers would be on stage, not production.** Production is expected to stay locked
  down, with access limited to the automated integration itself.
  Sam: *"Why would they need production data? That'd be really scary."*
- **If documentation site sign-in broke completely, no customer integration would stop working.**
  Signing into the docs and calling the API are two entirely separate things — the API is protected
  by a subscription key and request signing, not by this. Worst case, customers can't read the
  documentation. That's a bad day, not an outage.

---

## Related concerns raised in the same conversation

Not part of this decision, but they came up together and shouldn't get lost:

- **Secrets stored in the gateway are readable in plain text.** Sam spotted that you can open a
  stored secret in APIM and read the value. That's how the resource behaves, not a mistake of ours,
  but "stored in APIM" is not "stored securely". Sam suggested looking at a read-only security
  group that can't see secret values.
- **The sign-in method we're using is deprecated.** It works, and it's the only one the product
  accepts, but it's a known dead end. The stated ambition was an approach good for about five
  years — this isn't it.
- **Nobody has documented how the documentation site sign-in actually behaves.** Flagged in the
  same meeting and still true.

---

## Who needs to be involved

From the 7 August meeting, the agreed plan was to write the options up and run them past:

| Who | Why |
| --- | --- |
| Sam C, Richard, Marcus | As engineers — Sam asked specifically to see the Entra material |
| **Infra Platform (Ivan)** | Sam: *"Might be worth getting Infra-Platform written in as well — if Ivan is going to see it, he can coordinate."* |
| Gabriela | As product — has now confirmed the approach |
| Security | Because the change touches a registration shared with the main product |

---

## If we ever want to change it

Moving to customer-owned accounts is **not a config tweak**. It needs:

1. The sign-in provider unpinned from our directory (infrastructure change), **and**
2. The product changed to accept identities other than its own (product change).

Worth knowing now, because it's the kind of thing that gets assumed to be a small setting later.

---

## Related notes

- [[APIM IaC - authentication (phase 4)]] — the full picture of the two separate authentications
- [[D5 - Getting access to test on a real environment]] — what the untested gaps above are blocked on
- [[D1 - The developer portal sign-in secret]] — a worked example of the same constraint cascading
- [[APIM IaC - outstanding questions]] — ownership and access questions
- [[APIM IaC - known traps and gotchas]] — the detailed write-ups behind the two unproven items
- [[APIM IaC - ticket breakdown]] — the documentation site sign-in ticket
- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — where this was raised
