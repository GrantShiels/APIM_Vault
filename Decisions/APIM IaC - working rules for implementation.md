---
tags: [apim, iac, devo-28368, decision, ways-of-working]
updated: 2026-08-07
status: active
related-ticket: DEVO-28368
---

> [!warning] Audited 2026-10-07 — partly stale
> The rewrite of the tickets it predicts has happened and the component is `Apim` on `iac-main`, not
> `Spoke/ApimSpike`. The coding rules below are still useful. See [[_START HERE - APIM handover]].

# Working rules for implementing APIM in IaC

> Set out by Grant on 2026-08-06, before starting Phase 6.
> **These apply to every phase from here on**, and to whoever picks the work up afterwards.
> They exist because Phase 5 got built twice — see the worked example at the bottom.

---

## Why these rules exist

This spike has two jobs, and it is easy to forget the second one.

The obvious job is **finding out whether APIM can be deployed from code**. The less obvious job is
that this is also **how the team is learning Pulumi**. Every phase we build is a worked example
somebody will copy later. If the examples are rough, the production code will be rough, and the
learning will be of the wrong habits.

So "it's only a spike" is not a reason to cut a corner. It's the opposite — the code should look as
close to what we would genuinely ship as we can manage.

---

## The five rules for the code

### 1. Look for existing code before writing any

Before building anything, check whether another component or module in `Products/` already does it.
Not something *similar in spirit* — something we can actually import and use.

The repo is bigger than it looks. Naming, config, secrets, app registrations, App Insights, Log
Analytics, alerting and Key Vault access all have modules we already own. The
[[APIM IaC - ticket breakdown|existing-code audit]]
maps a lot of this out already.

**In practice:** grep `Products/` for the thing you are about to build. Read what you find. Only
write new code once you are satisfied nothing covers it.

### 2. Copy the existing pattern, don't invent a better one

Rule 1 is about *reusing code*. This one is about **reusing shape** — the cases where no module
exists, but several components clearly solve the same *kind* of problem the same way.

If three components handle a situation one way, we handle it that way too. Even if a different
approach would be technically neater. Consistency is worth more than cleverness here, because the
next person to read this code will have read those three components first.

**Some house patterns worth knowing:**

| Situation | House pattern | Where to look |
|---|---|---|
| "Azure needs a moment before this works" | `pulumiverse_time.Sleep` | `Modules/AgwCertificate`, `ContainerAppsEnv`, `SqlDb` |
| Adopting something Azure created for us | `ResourceOptions(import_=...)` | `Modules/ServiceBus` |
| Something azure-native genuinely can't express | dynamic provider | `DynamicProviders/AGW/Certificate` |
| Writing a secret into the spoke vault | `Modules/KeyVaultSecret` | `Spoke/Certificate` |
| Handing a value to another component | post-deployment hook | `Spoke/ApimSpike/hooks` |

### 3. Write it like production code, not spike code

No half measures on the basis that it gets thrown away. Specifically:

- **Unit tests alongside the code**, as every other component has.
- **Real modules and real components** — no shortcuts that "work for now".
- **Comments explain the non-obvious**, and describe the code as it stands. A comment that excuses a
  breach of the rules is not a fix — fix the breach.
- **`ruff check` clean**, tests green, before it's called done.

The code being on a spike branch changes where it lives. It does not change how it's written.

### 4. Follow the repo's own IaC instructions

Two files in `.github/instructions/`, both scoped to `Products/**`, are the authority. They are not
optional and they are not advisory.

**`copilot-iac-development.instructions.md`** — the technical rules. The two that bite most often:

> *"Cloud CLIs: Azure CLI (`az`) for **read access only**; perform no write operations through CLI"*

> *"Pulumi components should **not** interact with Az APIs directly. Use azure-native resources."*

**`copilot-iac-llm-behaviour.instructions.md`** — the behavioural rules. Four sections, but these are
the lines that matter most for this work:

> *"If you write 200 lines and it could be 50, rewrite it."*

> *"Match existing style, even if you'd do it differently."*

> *"Every changed line should trace directly to the user's request."*

> *"No abstractions for single-use code. No 'flexibility' or 'configurability' that wasn't requested."*

Read both before starting a phase, not after finishing one.

### 5. Treat the setup guide as a starting point, not a specification

**Amended 2026-08-07.** This rule used to say "reproduce it as written". That was based on a wrong
assumption about where the guide came from.

Sam Clifford, on the call:

> *"I am 99% sure that they built it from the IaC. So they built the IaC with AI, then they
> pointed AI at the IaC and said 'build me a doc', and then they handed it to us. The only human
> element is modifications me and Marcus have made to it. So you need to assume it's not something
> you should take verbatim. It's a starting point, but I wouldn't use that as the decision token
> for how you phase the work."*

So the [APIM Setup Guide](https://airtsdev.atlassian.net/wiki/spaces/DEV/pages/4999643160/APIM+Setup+Guide) is **AI-generated from VP's old IaC**, lightly edited by Sam and
Marcus. It is not a hand-written runbook, and it is not authoritative.

**That explains every contradiction we've hit so far:** two different OpenAPI spec paths given in
different phases; phases ordered as though they have dependencies that don't exist; and steps
Marcus says have never actually been run.

**What still holds:**

- **Concrete values stay as the guide has them** unless there's a reason to change them. A policy,
  a name, a threshold — copy it. We're still not redesigning APIM, and arbitrary divergence just
  makes the two harder to compare.
- **Raise deviations, don't sneak them in.** If we conclude the guide is wrong, say so explicitly
  in the plan and the PR. That part of the original rule was right and stays.

**What changes:**

- **The guide does not settle a design question on its own.** "The guide says so" is context, not
  evidence. Where the code, the platform's actual behaviour, or the existing IaC patterns disagree
  with it, **they win**.
- **Phase ordering is ours to choose.** Sam explicitly said not to use the guide as the decision
  token for phasing. We've followed its order so far because it's been workable, not because it's
  correct.
- **Expect it to be wrong occasionally.** When something doesn't match reality — a setting that
  isn't there, a default that doesn't exist — the guide being wrong is now the *first* hypothesis,
  not the last.

Grant's original steer still applies to values: *"the reason isn't necessary for our work. We just
want something that works and looks like the guide."* It just no longer applies to structure.

---

## How we write comments

Four conventions, applied consistently across the spike:

1. **Describe the present state only.** No "previously this used…", "replaces the old…", "originally
   we…". A comment is a description of the code as it stands, not a changelog. Git has the history.
2. **Never cite a file and line number** (`app_cfg_interface.py:214`). Line numbers rot within days.
   Name the class, method or file instead.
3. **Callables get brackets, things don't.** `get_kv_access()` but `ApimHmacSecret`, `vault_access`.
4. **Explain the non-obvious in plain English.** Assume the reader knows Python but not Azure. The
   comment should say *why*, since the code already says *what*.

And the one that matters most:

> [!warning] A comment must never excuse a rule breach
> If code breaks one of the rules above and the instinct is to write a comment explaining why that's
> acceptable — **fix the code instead**. The comment is a signal the approach is wrong.

---

## Ways of working

**Grant deploys and tests; I make the changes.** No `pulumi up`, no `pulumi destroy`, no Azure calls
that change anything. Reading Azure is fine; writing is not. This is partly the repo's own rule
(§4 above) and partly that Grant is learning by running it himself.

**Nothing gets committed or pushed without Grant asking.** He reviews the working tree first and
tests end to end manually. When a commit is asked for: `DEVO-28368:` prefix, the Copilot
`Co-authored-by` trailer, `yarn lint-staged` and `dotnet husky run` beforehand, and **never**
`--no-verify`.

**Richard Donkin and the IaC reviewer are taken as gospel.** They own SRE and IaC. Where their
guidance conflicts with what seems sensible, theirs wins, and it gets written down rather than
argued.

**Grant's own notes outrank any source I've read myself.** *(Added 2026-08-07.)* Where Grant's
notes in the Obsidian vault disagree with a meeting transcript, a Slack thread, the setup guide or
anything else, **the notes are right and the other source is wrong**. He was in the room; a
transcript is a machine's guess at what was said, and it mangles names, tooling and numbers
routinely.

So: use the transcript to *fill gaps* Grant's notes don't cover, never to *correct* them. If
something in another source appears to contradict the notes, that's a question to raise — not a
discrepancy to resolve unilaterally. And where a detail exists only in a transcript and looks
uncertain, **flag it as unconfirmed rather than asserting it**.

**Phase 2's code is not to be edited.** Grant is separately working out whether it's needed at all
(it's a confirmed no-op on BasicV2). Leave `interface/apim_arm.py`, `apim_defaults.py` and the
`local.Command` in the component alone.

**"Ticket" means the local one.** When Grant says "update the ticket", he means the entry in
[[APIM IaC - ticket breakdown]], not Jira. Both the Jira tickets (DEVO-28369 … 28373) *and* the
local plan will be substantially rewritten once implementation is finished — so neither is fixed, and
neither should be treated as untouchable. Keep the local plan current as we go; it's the version we
actually work from day to day.

---

## Explaining the work

Everything gets explained **in plain, low-jargon English by default** — not just when asked. Grant
relays this work to a Product Owner and to a team who, like him, hadn't worked with APIM before this
spike started.

That means: spell out what a thing does before naming it, prefer a sentence over a term of art, and
never assume the Azure vocabulary is shared. If a technical term is genuinely needed, define it once
in passing.

This applies to written notes, decision records, ticket updates and answers in conversation alike.

---

## The worked example these came from

Phase 5 needed to handle a gap: Azure doesn't apply an access grant everywhere the instant it's made,
so APIM's first attempt to read a Key Vault secret can be refused.

**What was built first:** a custom Pulumi dynamic provider that asked APIM whether its last read had
worked, told it to retry, and polled until success or timeout. Roughly **400 lines across 5 files,
with 24 tests**. It was correct, and it worked.

**What it was replaced with:** **8 lines** of `pulumiverse_time.Sleep`. `Modules/AgwCertificate`
already waits 180 seconds after granting *the same role* on *the same vault*. `ContainerAppsEnv` and
`SqlDb` wait the same way after their own grants. Six files in the repo use this. Nobody checked.

Both solutions work. The 8-line one is the right answer, because it is the one the next person will
recognise.

> [!important] The test to apply before building anything
> **"Has someone in this repo already had this problem?"**
> If you're about to write a custom provider, a helper class, or anything over about 50 lines — stop
> and grep first. The answer is usually yes.

---

## Related

- [[APIM IaC - ticket breakdown]] — the authoritative plan; each ticket notes what to reuse
- [[APIM IaC - known traps and gotchas]] — G13 records the Phase 5 race and its fix
- [[APIM IaC - HMAC signing secret decision (phase 5)]] — the decision record these rules came out of
