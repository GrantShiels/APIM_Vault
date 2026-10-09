---
tags: [apim, iac, devo-28368, authentication, entra, reference]
updated: 2026-08-11
status: living-doc
related-ticket: DEVO-28368
---

> [!warning] Audited 2026-10-07
> Implemented in DEVO-28373 (scope is `entities_access_as_user`). Token checks: DEVO-31025. Long-term login: DEVO-30734.

# APIM authentication — how it works, what's proven, what isn't

> [!abstract] Why this document exists
> Authentication is the single most confusing part of this work and, until now, there was nowhere to
> send someone who asked *"how does customer sign-in actually work, and what's left?"*
>
> The answer was spread across a decision document, six entries in a 57-page trap register, and a
> set of meeting notes. **This is that answer, assembled in one place.** It's a summary of what we
> already know, not new research.

---

## The single most important thing to understand

> [!important] There are **two** authentications here, and they are completely separate
> They get conflated constantly — including by us, early on. Almost every confusion about this work
> traces back to mixing them up.
>
> | | **Signing into the documentation site** | **Calling the API** |
> | --- | --- | --- |
> | **Who does it** | A person, in a browser | A machine, in a script |
> | **What it protects** | The documentation website | The actual API |
> | **How** | Microsoft Entra sign-in | A subscription key + a signature |
> | **If it breaks** | Customers can't read the docs. **The API keeps working.** | Customers can't use the API at all. |
> | **Status** | ⚠️ Works, with real caveats | ✅ Proven |

**A useful rule of thumb:** if the sign-in side broke completely tomorrow, no customer integration
would stop working. That's not an argument for ignoring it — but it's why the two shouldn't share a
risk assessment.

---

## Part 1 — Calling the API *(the part that works)*

A customer's system proves who it is with two things:

1. **A subscription key** — like a username, identifying which customer is calling
2. **A signature** — proof the request genuinely came from them and wasn't altered in transit

The gateway checks both before letting anything through, and the signature check happens **before**
the request reaches Dayshape. If either is wrong, the request is refused at the gateway.

**Status: built and proven.** Verified against real Azure.

> [!caution] One thing has never been tested
> The signature relies on a shared value that the gateway creates and Dayshape holds a copy of.
> **That hand-over has never actually run**, because it needs a real Dayshape deployment and we've
> only ever had a sandbox.
>
> The code exists and is reviewed. It has not been observed working. → **D5**

---

## Part 2 — Signing into the documentation site *(the part with caveats)*

Customers' staff sign in with their Microsoft account to read the API documentation.

### How it works

The site hands sign-in to Microsoft Entra using **Dayshape's existing sign-in registration** — the
same one the main product uses. It has to borrow it; it can't have its own **(G9)**.

> [!info] Why it can't have its own — the root of most problems here
> Dayshape's main product checks a specific kind of sign-in token, and that token is **always
> stamped with the identity of whoever requested it.** That stamp is fixed by the protocol; it
> cannot be changed.
>
> So if the documentation site had its own identity, the token would be stamped with *its* identity,
> and Dayshape would reject it as coming from a stranger.
>
> **Everything awkward about the sign-in setup follows from this one constraint** — including the
> whole secret problem in **D1**.

### And one more inherited constraint

The sign-in method the setup guide specifies is a **deprecated one (G5)**. We're using it because
it's the only one Dayshape currently accepts — the modern alternative returns a different kind of
token that the product rejects.

It works. But it is a **known dead end**, not a finished design. The stated ambition — recorded in
`#project-api`, December 2025 — is an API authentication approach that will stand for around five
years, and this isn't it.

> **Do not read "it works" as "it is finished."** This is a scheduled migration that hasn't been
> scheduled.

---

## Part 3 — The five things that are wrong or unproven

| | Issue | Impact | Status |
| --- | --- | --- | --- |
| **G7** | ~~The sign-in secret **expires after 12 months** and nobody owns replacing it~~ | **Fixed.** The deployment now replaces the secret every 30 days by itself | ✅ **resolved 2026-08-13** → **D1** |
| **G1** | The site asks Microsoft for the user's **email address**, but permission for that was never granted | May cause a consent prompt or an outright sign-in failure. **Never tested with a real customer account.** | 🔴 |
| **G2** | Sign-in **silently requires the user to be explicitly permitted** | Anyone not on the list gets a generic error that doesn't say so. Easy to lose an afternoon to. **Whether the current group is right for external customers is an open design point.** | 🔴 |
| **G6** | Deploying the sign-in component **can wipe the documentation site's return addresses** | Portal sign-in breaks until the next correct deployment. Guarded by a setting **maintained by hand, not in version control.** | 🟠 mitigated |
| **G17** | Secrets stored in the gateway are **readable in plain text** by anyone with access to it | Not a mistake on our part — it's how the resource behaves. But "stored in the gateway" is **not** equivalent to "stored securely". | 🔴 → **D1** |

### The two worth your attention

**G7 was a guaranteed future incident — it is now closed.** It was not a risk but a certainty, on a
date nobody had recorded. As of 2026-08-13 the deployment creates the secret itself and replaces it
every 30 days, so both halves of **D1** are answered: how it is *created* **and** who *replaces* it.

> The one thing left, stated plainly: renewal happens during an APIM deployment. An environment left
> undeployed for more than twelve months still drifts. Customers get 5–6 deploys a year, so this only
> really touches long-dormant internal environments.

**G1 has never been tested with a real customer.** Everything we know about sign-in comes from
Dayshape staff signing in with Dayshape accounts. An external customer's first sign-in attempt is
genuinely unproven, and G1 and G2 both make it plausible that it fails. → **D5**

---

## Part 4 — What this means for the decisions

| Decision | Comes from |
| --- | --- |
| **[[D1 - The developer portal sign-in secret\|D1 Part 1]]** | G9 forces borrowing Dayshape's registration → creates the secret problem. G17 means wherever it's stored, it's visible. ✅ *resolved* |
| **[[D1 - The developer portal sign-in secret\|D1 Part 2]]** | G7, directly. ✅ *resolved — it renews itself* |
| **[[D5 - Getting access to test on a real environment\|D5]]** | G1 and G2 can only be settled by a real sign-in. |

> [!tip] For the conversation with Richard and Marcus
> Sam asked that authentication be discussed **as a whole** rather than piecemeal. This document is
> the whole. **D1 no longer needs deciding** — both halves are answered and built — but it does want a
> read-through, because it puts a credential on Dayshape's own sign-in record. So the two things to get
> out of that conversation are:
>
> 1. **Is everyone comfortable with the deployment adding and rotating a credential on Dayshape's
>    sign-in record?** (D1 — a review, not a decision)
> 2. **Is the deprecated sign-in method acceptable for launch, and what replaces it?** (G5 — currently
>    tracked nowhere, because it's a direction rather than a blocker)

---

## The plain version for a Product Owner

Customers get a documentation website where their developers log in with their Microsoft account and
read how to use our API. That is separate from the API itself, which is protected differently and
works.

The login side works, but it borrows a credential from the main Dayshape product because of a
Microsoft limitation we can't work around. That creates two problems: **getting that credential onto
customer deployments safely**, and **replacing it before it expires in twelve months.** If nobody
replaces it, customer logins stop working roughly a year after launch, with nothing to warn us.

We also haven't yet had a real customer log in — only Dayshape staff. Two known issues make it
plausible that a first external login fails, and we can't test that until we can deploy somewhere
real.

---

## Related

- [[APIM IaC - client secret decision]] — the working analysis behind D1 *(superseded — D1 is the answer)*
- [[APIM IaC - known traps and gotchas]] — G1, G2, G5, G6, G7, G9, G17
- [[Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07]] — where Sam asked for auth to be taken as a whole
- [[APIM IaC - outstanding questions]] — Q5, Q7, Q8
