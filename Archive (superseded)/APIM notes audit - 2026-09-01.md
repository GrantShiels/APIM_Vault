---
title: APIM notes audit
created: 2026-09-01
status: snapshot — not a living document
tags: [apim, iac, audit, devo-28368]
---

# APIM notes audit — 1 September 2026

> [!warning] This is a dated snapshot, not a living document
> Judged against the 8 IaC tickets as they stood on **1 September 2026**, the PoC branch
> `spike/DEVO-28368-apim-investigation`, and Confluence FDP page 5570265126 (version 10, last
> edited 24 August). If the tickets change, this goes stale like everything else here.
>
> **Nothing has been edited, moved or deleted.** This is a report only.

---

## Summary

52 files audited. The vault is in **much better shape than expected** — the tickets were written
from these notes, so most content matches by construction.

| Verdict | Count | Meaning |
| --- | --- | --- |
| **CURRENT** | 14 | Accurate and still useful |
| **HISTORICAL** | 22 | True when written, a record not a plan — keep, don't act on |
| **CARRY** | 9 | Holds something still live that has no home in a ticket |
| **SUPERSEDED** | 4 | Content now lives in a ticket |
| **CONTRADICTS** | 2 | States something false about the current plan |
| **DEAD** | 1 | Delete |

### The three things most worth fixing

**1. DEVO-28373 is wrong, and the notes are right.** The ticket says to *"create the app
registration"* for the developer portal. Three separate notes record that APIM **cannot have its
own app registration** — it is forced to borrow Core's, and this was confirmed from Core's source
code, not assumed. A developer following the ticket will build the wrong thing and hit a confusing
failure. **Fix the ticket.** Detail below.

**2. One note points at the wrong tickets.** `Existing APIM component - state assessment.md`
carries a mapping table built against a superseded 5-ticket set. It sends readers to DEVO-28372
for "monitoring/alerts" when that ticket is request signing. Highest chance of actively wasting
someone's time.

**3. The vault says "Ticket 1–8", Jira says DEVO-xxxxx, and nothing connects them.** The mapping
is clean and 1:1 (below). Adding the DEVO numbers is a find-and-replace and makes the whole vault
navigable.

### Two predictions I got wrong, recorded honestly

- I expected the `(phase N)` filenames to be stale vocabulary from the Confluence guide. **They are
  not.** Every ticket cites guide phases explicitly (*"Setup guide coverage: Phase 5"*). The
  numbering is shared, not superseded. No action needed.
- I expected `Unaswered questions (post Spike).md` to be an empty stub. It contains one real
  question that is now **answerable** — see CARRY list.

---

## Where the tickets and the FDP disagree

Both are authoritative. Two authoritative sources contradicting each other matters more than a
stale note.

> [!danger] The FDP is itself out of date
> FDP version 10 was last edited **24 August**. Every ticket was rewritten **26–31 August**. There
> are three tiers of staleness here, not two.

### 1. Performance — a straight contradiction

| Source | Says |
| --- | --- |
| **FDP §7** | *"There should be no impact to performance as this is it's own piece of IaC removed from the core app"* — load testing marked "See above", i.e. not planned |
| **DEVO-30006** | *"There is no performance evidence. No load test has been run, and no target has been agreed for how much slower a call through the gateway is than a call straight to Dayshape."* |

These cannot both be true. Putting a gateway in front of the API adds latency by definition. The
FDP answer reads as though it were written about *the Core app being unaffected*, which is a
different question from the one asked. **The FDP is the wrong one.**

### 2. The biggest risk in the epic is missing from the risk register

DEVO-30004 opens with *"the largest remaining risk in the epic"* — the master documentation site
living unowned in a personal sandbox. The FDP risk table (§5) does not mention it. Nor does it
mention the unknown production cost, or the "deploy goes green, every API call broken" failure
mode that DEVO-30006 is built around. Three of its five rows are blank.

### 3. The decision log stops on 13 August

Three decisions recorded, all still correct: build fresh rather than VP's component (29/07),
keep Entra-based portal sign-in (13/08), generate OpenAPI in the release pipeline (11/08).

The problem is **absence**, not contradiction. Decisions since made and visible in the tickets but
never logged:

- Basic v2 as the tier for every customer deployment
- Region is configuration with hub fallback, **not** pinned to West Europe
- Alerts live in the platform management subscription
- HMAC secret reaches Core via an App Configuration pointer, not a variable group
- The clone script gets exactly one home
- The deploy-window response shape (two distinct "temporarily unavailable" answers)

### 4. Testing sections unfilled

FDP §9–11 are almost entirely `TBC`. Meanwhile DEVO-30005 and DEVO-30006 carry hard evidence
requirements — every alert proven to fire and clear, smoke test proven to fail when broken. Not a
contradiction, a gap, but a visible one if anyone reviews the FDP.

---

## Contradictions

Only two, and one of them is the ticket's fault.

### C1 — DEVO-28373 says to create an app registration. It can't be done.

**Severity: high.** This will cost a developer real time.

**The ticket says:**
> *"Create the app registration and the sign-in secret the documentation site needs."*
> PR AC: *"App registration and sign-in secret are created by the component."*

**The notes say** (`outstanding questions.md` Q7, Q5; `known traps and gotchas.md` G9; `D1`):

> *Q7 — Should APIM have its own app registration? ✅ **answered — no, and it can't.** It borrows
> Core's, and that is **forced**, not a preference.*

The evidence is specific and was taken from Core's source rather than assumed:

- `AuthenticationExtensions.cs` validates the token audience against `{ ClientId, api://{ClientId} }`
  — Core's own identity and nothing else. A token from a separate app registration is refused.
- `IdpConfigurationProvider` has exactly **two** hard-coded sign-in-provider slots, both already
  used by real customer providers. There is no third slot without changing Core.

What actually happens is that APIM **adds a secret to Core's existing app registration** and
replaces it every 30 days. Redirect URIs deliberately stay with the Auth component, because that
resource genuinely is authoritative — which is why the ticket's *other* half is correct.

**Why this is dangerous rather than merely untidy:** creating a separate app registration
*succeeds* in Azure. It deploys clean. The failure only appears at sign-in, as a token rejection
with nothing useful on screen. The notes record that this path was explored at length before being
ruled out — so the ticket as written invites someone to repeat work already done and discarded.

> **Recommended fix to DEVO-28373:** replace *"Create the app registration"* with *"Add a sign-in
> secret to Core's existing app registration"*, and cite G9 for why a separate one is impossible.

### C1b — the same ticket is over-confident about renewal

**Severity: low, but same ticket.**

DEVO-28373 says: *"The sign-in secret renews itself, so there is no expiry to manage."*

D1 / Q5 are more precise: renewal happens **only when APIM is deployed**. The note states the
residual risk plainly — *"an environment left alone for over twelve months still drifts."*

The ticket's version would let someone close this off as solved. It is solved *for actively
deployed environments*. Worth one clause in the ticket.

### C2 — `Existing APIM component - state assessment.md` points at the wrong tickets

**Severity: medium.** Sends readers to the wrong place.

Its closing table maps work to tickets using a superseded 5-ticket set:

| Note says | Ticket actually is |
| --- | --- |
| DEVO-28369 — orchestration | Build the production APIM component |
| DEVO-28370 — config source of truth | Publish the API definition |
| DEVO-28371 — residual manual steps | API, routing, deploy-window responses |
| DEVO-28372 — monitoring/alerts | **Request signing (HMAC)** |
| DEVO-28373 — SRE handover | **Developer portal sign-in** |

The body of the note is fine — it is a resource inventory of the old component and already carries
a clear superseded banner at the top. Only the table is wrong.

> [!tip] This note contains something Evelynn needs right now
> It records that the old component **did** clean up Azure's default Echo API and Starter/Unlimited
> products (guide Phase 2). DEVO-28369 asks her to *"check first whether those defaults are even
> created on the tier we are using"*. This note is evidence they exist on at least one tier — a
> useful starting point rather than a blank investigation.

---

## Still live, and with no home in a ticket

The CARRY list — things that are true, unresolved, and would be lost if the vault were archived.

| # | What | Where it lives now | Suggested home |
| --- | --- | --- | --- |
| 1 | **Secrets stored in APIM are not obfuscated** — readable in the portal by anyone with access. A read-only developer group may be needed. | G17, Q5 | **DEVO-28373 or a security ticket.** Not mentioned in any ticket. Security-relevant. |
| 2 | **Does anything in the estate expect APIM to have a fixed outbound IP?** Basic v2 has none. | Q4 follow-up | DEVO-30006 |
| 3 | **Basic v2's structural limits** — no private networking (APIM reaches Core over the public internet), no fixed IP, no backup/restore, **no migration path between v2 and classic in either direction**, ceiling of 10 scale units, no multi-region, no availability zones | Q4 | **Architecture review** — none of this appears in any ticket, and "changing tier later means rebuilding every customer's instance" is exactly the kind of thing a review should see |
| 4 | **The guide asks for an email address we do not grant permission for** | Q8 | DEVO-28373 |
| 5 | **Production service connections cannot read the content master at all.** `…Orchestrator.Upper` and `…Orchestrator.Lower` have no access; Dreich and Sandbox have it only by inheritance. **A Dreich test will pass and production will fail.** | D5, G30 | **DEVO-30004** — the ticket says "what permission is granted" but does not carry this evidenced finding, which is sharper than the ticket's wording |
| 6 | **Data residency** — whether customer API traffic may transit another country. Making region configurable (DEVO-28369) gives us the *mechanism*; it does not answer whether a given region is *permitted*, and does not help where Azure refuses the region. | D8, Q6 | DEVO-30006 or a compliance owner |
| 7 | **Is the per-customer monthly cost at 100+ instances understood and budgeted?** | Q4 follow-up | DEVO-30006 (which raises cost but not at this scale) |
| 8 | **Are the `docs/runbook` docs used by anyone — can they be removed?** | `Unaswered questions (post Spike).md` | **Answerable now: no.** DEVO-28370 cites `docs/runbook/002-APIM-Nswag-OpenApi-generation.md` as the authoritative record of the generation step. Do not remove that one. |
| 9 | **The setup guide is AI-generated, not hand-written**, and has confirmed omissions and invalid policy XML as printed | G16, G8, G3, G4 | Worth stating on the Confluence guide itself — every ticket cites it as *"Setup guide coverage"* |

---

## The mapping the whole vault is missing

Every note says "Ticket 1" through "Ticket 8". Jira says DEVO. Nothing connects them. The mapping
is exact:

| Vault | Jira | Subject |
| --- | --- | --- |
| Ticket 1 | **DEVO-28369** | Build the production APIM component |
| Ticket 2 | **DEVO-28370** | Publish the API definition from the Core build |
| Ticket 3 | **DEVO-28371** | API, routing, deploy-window responses |
| Ticket 4 | **DEVO-28372** | Request signing between gateway and Dayshape |
| Ticket 5 | **DEVO-28373** | Developer portal sign-in |
| Ticket 6 | **DEVO-30004** | Permanent home for the customer documentation site |
| Ticket 7 | **DEVO-30005** | Monitoring and alerting |
| Ticket 8 | **DEVO-30006** | Rollout, handover, per-customer enablement |

Confirmed against the section headings of `APIM IaC - ticket breakdown.md` (24 August) and the
summary of each Jira issue. Every one matches.

---

## Full file table

### Root

| File | Verdict | Why | Owner |
| --- | --- | --- | --- |
| `APIM IaC - known traps and gotchas.md` (124 KB) | **CURRENT** | 47 entries. The most valuable file here — the tickets were written from it. See gotchas note below. | All |
| `APIM IaC - how to run a Dreich test deploy.md` (22 KB) | **CARRY** | Procedure is accurate and still the route. Says `ApimSpike`; component becomes `APIM`. | DEVO-30006 |
| `APIM IaC - deploying to a new environment (prerequisites).md` (12 KB) | **CARRY** | Same — accurate, spike component name throughout (10 occurrences). | DEVO-30006 |
| `APIM IaC - authentication (phase 4).md` (9 KB) | **CURRENT** | Background for the sign-in flow. Part of the evidence base for C1. | DEVO-28373 |
| `APIM IaC - what made this hard (retrospective).md` (11 KB) | **HISTORICAL** | Already published to Confluence. Record, not plan. | — |
| `Team skills vs APIM delivery - risk assessment.md` (9 KB) | **CARRY** | Written pre-spike, so its risk framing is dated — but the FDP risk table is nearly empty and this is the only real risk assessment in existence. | FDP §5 |

### Decisions

| File | Verdict | Why | Owner |
| --- | --- | --- | --- |
| `APIM IaC - outstanding questions.md` (37 KB) | **CURRENT** | The Q-register. Q1–Q12, most answered in place. Two nits: the Q9 row still says "🔴 open" when the same file records it as answered further down; and it says "10-phase guide" — the guide has 12. | All |
| `APIM IaC - working rules for implementation.md` (12 KB) | **CURRENT** | Working practice, not plan. Doesn't go stale. | — |
| `APIM IaC - rollout and testing strategy.md` (11 KB) | **CARRY** | Feeds the rollout ticket; predates it. | DEVO-30006 |
| `APIM IaC - client secret decision.md` (19 KB) | **SUPERSEDED** | Absorbed into D1 on 13 August. | D1 |
| `Decision - customer developer accounts in our Entra directory.md` (11 KB) | **CURRENT** | Matches FDP decision log 13/08 and DEVO-28373. | DEVO-28373 |
| `Decision - do not build on existing APIM IaC.md` (3 KB) | **HISTORICAL** | Decided, recorded in FDP 29/07, reflected in DEVO-28369. Carries a now-pointless TODO to *"confirm Ivo's reasons when Slack access is back"* — the decision has since been made and logged; drop it. | DEVO-28369 |
| `Existing APIM component - state assessment.md` (5 KB) | 🔴 **CONTRADICTS** | Wrong ticket mapping table — see **C2**. Body is a useful inventory of the component DEVO-28369 will delete. | DEVO-28369 |

### Decisions / Needs sign-off

| File | Verdict | Why | Owner |
| --- | --- | --- | --- |
| `_Decisions needing sign-off - start here.md` (7 KB) | **CURRENT** | Index still accurate. Status column is a fortnight behind (D5 "in motion", D8 "⬜"). Uses Ticket 1–8 numbering. | — |
| `D1 - The developer portal sign-in secret.md` (39 KB) | **CURRENT** | Both parts resolved in-note. **This is the evidence base for C1** — it is right and the ticket is wrong. | DEVO-28373 |
| `D2 - The master documentation site.md` (26 KB) | **CURRENT** | DEVO-30004 lifted this almost verbatim. All three parts still ⬜. Genuinely still needs the conversation. | DEVO-30004 |
| `D5 - Getting access to test on a real environment.md` (8 KB) | **CURRENT** | Self-resolved in-note (ADO pipeline 739 + `iacBranchOverride`). Retains two things worth keeping: the production-access finding (**CARRY 5**) and the GSOT trap. | DEVO-30004 |
| `D7 - Accepting an unsupported content migration.md` (5 KB) | **CURRENT** | Still genuinely undecided. DEVO-30004 requires exactly this acceptance. Says "26.5"; target is now **26.6.0**. | DEVO-30004 |
| `D8 - Is West Europe acceptable for customer API traffic.md` (5 KB) | **SUPERSEDED (in part)** | Its Option C — *"region becomes a per-customer setting"* — is what DEVO-28369 now specifies. **But the data-residency question underneath is not answered by that**, and the note itself flags that Option C *"doesn't help where Azure won't allow the region"*. See **CARRY 6**. | DEVO-28369 |
| `D10 - What customers see during a deploy window.md` (5 KB) | **SUPERSEDED** | Decided. DEVO-28371: *"The agreed shape is two clearly different 'temporarily unavailable' answers"* — that is this note's Option A, refined. Note still reads "needs-decision" and asks "which option for 26.5?". | DEVO-28371 |

### Implementation Ideas

Every one maps cleanly to a ticket. The `(phase N)` naming is **correct** — the tickets cite the
same guide phases.

| File | Verdict | Owner |
| --- | --- | --- |
| `APIM IaC - OpenAPI source of truth (phase 7).md` (43 KB) | **CURRENT** | DEVO-28370 |
| `APIM IaC - developer portal content (phase 10).md` (26 KB) | **CURRENT** | DEVO-30004 |
| `APIM IaC - HMAC signing secret decision (phase 5).md` (24 KB) | **CURRENT** | DEVO-28372 |
| `APIM IaC - diagnostics and monitoring (phase 8).md` (22 KB) | **CURRENT** | DEVO-30005 |
| `APIM IaC - portal content permissions (phase 10).md` (14 KB) | **CURRENT** | DEVO-30004 |
| `APIM IaC - portal content, why there is no IaC-native route.md` (9 KB) | **CURRENT** | DEVO-30004 — this is the "we checked and it can't be avoided" evidence |
| `APIM IaC - maintenance responses (design sketch).md` (6 KB) | **SUPERSEDED** | DEVO-28371 — design now agreed in the ticket |
| `APIM IaC - portal content capture dry run.md` (5 KB) | **HISTORICAL** | DEVO-30004 |
| `APIM IaC - ticket breakdown.md` (27 KB, 24 Aug) | **HISTORICAL** | Superseded by Jira, but it is the Rosetta Stone for the Ticket 1–8 mapping. Keep until the DEVO numbers are added everywhere. |
| `APIM IaC - ticket breakdown.md.bak` (25 KB) | 🗑️ **DEAD** | Stray backup of an older version. Delete. |

### Inital Findings — all HISTORICAL

`APIM Findings Summary.md` · `What is APIM.md` · `What APIM means for Ecosystem.md` ·
`Why Ecosystem owns APIM.md` · `APIM and SRE - understanding.md` ·
`Target IaC process - componentised framework.md`

Six orientation notes from 11 August, 2–4 KB each. None reads as a current plan — checked. Still
the best onboarding material that exists for someone new to this work. Keep as a set.

### Meeting Notes — all HISTORICAL

Ten notes, 11–18 August. A record of what was said on a day; they don't go stale the way a plan
does. Light pass only, as agreed. **None reads as a current plan.** Two are load-bearing as
evidence and are cited by the decision notes:

- `Meeting Notes - Richard, Marcus, Sam C and Andy L 2026-08-07.md` (27 KB) — where Q1, Q2, Q5 and
  Q12 were answered
- `Meeting Notes - Sam C and Marcus 2026-08-18.md` (12 KB)

### Questions for Meetings

| File | Verdict | Why |
| --- | --- | --- |
| `Questions for Richard Donkin - 4A and dynamic providers.md` (7 KB) | **CARRY** | DEVO-28373 requires a dynamic provider and points at `DynamicProviders/`. Worth checking whether these questions were answered before Evelynn or you hit the same ground. |
| `Unaswered questions (post Spike).md` (0.1 KB) | **CARRY** | One line, and it is answerable — see **CARRY 8**. Not the empty stub it looks like. |

### Data

| File | Verdict | Why |
| --- | --- | --- |
| `APIM test setup - my data points.md` (8 KB) | **HISTORICAL** | Spike environment values. |
| `Top secret.md` (0.3 KB) | 🗑️ **DEAD** | ✅ **No live secret** — the value is already masked as `******`, and the file carries its own note that nothing reads the variable since 13 August. Safe to delete. Separately: this was the **IDP** secret. The **HMAC** secret on Dreich is a different one and still wants rotating. |

### attachments

`Screenshot_5.png`, `client-secret-value-location.png` — **CURRENT**, referenced by notes.

---

## The gotchas register — a note of its own

`known traps and gotchas.md` is 124 KB and 47 entries, and it is the single most valuable artefact
of the spike. The question asked was *"which entries are now fixed by a ticket, and which are still
live traps."*

**The great majority are now carried in a ticket.** Spot-checked, each of these appears near-verbatim
in the "Watch out for" section of its ticket:

| Gotcha | Now in |
| --- | --- |
| G39 — APIM eats the version segment, Core insists on it | DEVO-28371 |
| G40 — "Core" is three different component names | DEVO-28372 |
| G44 — vault access is by group membership, and listing tells you nothing | DEVO-28372 |
| G47 — a time filter inside alert KQL is a second look-back window | DEVO-30005 |
| G14 — APIM resources return far fewer outputs than you pass in | DEVO-28369 |
| G15 / G37 — Azure holds the APIM name for hours after deletion | DEVO-28369 |
| G10 — child resources need an explicit name | DEVO-28369 |
| G6 — running Auth without the flag strips the portal's redirect URIs | DEVO-28373 |
| G2 — sign-in silently requires app-role assignment | DEVO-28373 |
| G5 — implicit flow is a deprecated dependency | DEVO-28373 |
| G24 / G26 — the sandbox master, and the unsupported copy | DEVO-30004 |
| G18 / G23 — APIM keeps serving mid-deploy; APIM deploys first | DEVO-28371 |
| G31 — a component the environment doesn't know about is skipped, and the deploy reports success | DEVO-30006 |

**Live traps with no ticket home** — mostly IaC-general rather than APIM-specific, which is
probably why they didn't make it into tickets. Worth keeping the register alive for these alone:

G17 (secrets visible in the APIM portal) · G28 (change-detection fingerprint not reproducible
across machines) · G29 (the dev deploy command offers to register an unknown environment, and
saying yes creates a broken one) · G32 (there is no dry-run through the pipeline) · G33 (a branch
override beats IaC versioning, but only on internal environments) · G34 (naming one component
silently drops its dependencies) · G36 ("do not delete" is inherited by children, so anything hung
off the sign-in record can never be removed) · G41 (environment values are remapped, and the remap
is wrong for anything Pulumi did not create) · G46 (a test assertion inside a Pulumi Output can
pass by never running)

> [!note] G29 is the one that already bit us
> *"The dev deploy command, when given an environment it doesn't recognise, offers to add it. The
> safe answer is always no."* This is the same failure family as the
> `No environment found for client_id='nyx'` error hit recently. It has no ticket home and is
> exactly the kind of thing a new starter walks into.

**Recommendation:** do not fold this file into the tickets. Keep it as the register, and add a
column or tag marking which entries are now ticket-carried. Its value is that it is *one place*.

---

## Recommendations

Grouped so they can be done in batches. Nothing here has been actioned.

### Do first — these are wrong and will cost time

1. **Fix DEVO-28373.** Replace *"Create the app registration"* with *"Add a sign-in secret to
   Core's existing app registration"*, cite G9, and soften *"renews itself, so there is no expiry
   to manage"* to note that renewal only happens on deploy. *(C1, C1b)*
2. **Fix the ticket table** at the bottom of `Existing APIM component - state assessment.md`, or
   delete the table and keep the inventory. *(C2)*
3. **Delete two files:** `APIM IaC - ticket breakdown.md.bak` and `Data\Top secret.md`.

### Do next — cheap, high value

4. **Add DEVO numbers alongside "Ticket N" throughout the vault**, using the mapping above. Largely
   find-and-replace. This is the single change that makes the vault usable by anyone else.
5. **Rename `ApimSpike` → `APIM`** in the two operational how-tos, and mark them as the seed of the
   DEVO-30006 runbook rather than spike notes.
6. **Close out the resolved decision notes** — D10 is decided in DEVO-28371 and D8's mechanism is
   decided in DEVO-28369. Fill in their Decision boxes with a pointer rather than leaving them
   reading "to be completed". Note that D8's *data-residency* half stays open.
7. **Fix "26.5" → "26.6.0"** in D7 and D10. The FDP gives the target as 26.6.0 / 6 November.

### Take to someone else

8. **The FDP needs a pass, and it is more out of date than the vault.** Specifically: the
   performance answer in §7 contradicts DEVO-30006 and should be corrected; the risk table is
   missing the epic's biggest risk; the decision log stops on 13 August and is missing at least six
   decisions. *(You own this page.)*
9. **Raise CARRY 1 (secrets readable in the APIM portal) with security or fold it into DEVO-28373.**
   It is the only unhomed item with a security dimension.
10. **Put CARRY 3 (Basic v2's structural limits) in front of the architecture review.** No private
    networking, no fixed IP, no backup/restore, and **no migration path between v2 and classic in
    either direction** — that last one means a tier change later would mean rebuilding every
    customer's instance. None of it is in a ticket.
11. **Get CARRY 5 into DEVO-30004** — production service connections currently cannot read the
    content master at all, and a Dreich test passing proves nothing about production. The ticket
    covers this only as "what permission is granted"; the note has the evidence.

### Leave alone

12. **Meeting Notes, Inital Findings, and the retrospective.** 22 files, all HISTORICAL, none
    misleading. They are the record of how this was worked out and cost nothing to keep.
13. **`known traps and gotchas.md`.** Do not break it up. Tag entries that are now ticket-carried,
    but keep it whole.
