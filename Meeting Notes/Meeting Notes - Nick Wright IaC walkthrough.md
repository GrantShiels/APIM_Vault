---
tags: [apim, meeting-notes, iac, nick-wright, pulumi, testing, devo-28368]
date: 2026-07-14
status: raw-to-tidy
---

# Meeting Notes — Nick Wright (Platform Engineer, IaC walkthrough)

> Notes from the call with **Nick Wright**, a **platform engineer
> who set up much of the company's IaC.** Highly relevant — he's hit most of the problems we're about to.
> Context: [[Target IaC process - componentised framework]] · [[Decision - do not build on existing APIM IaC]].

## Headline

- Nick is a **primary author of the IaC framework** — most senior source we've spoken to so far.
- Gave us a **practical way of working**: build the APIM component **with Copilot in VS Code**,
  going through the **APIM doc section by section**, and using the **Azure portal as the reference**
  for what "good" looks like before translating it to Pulumi.

## Ways of working (Nick's recommendations) ✅

- **Build with Copilot, section by section.** Work through the APIM setup doc **one piece at a time**,
  asking Copilot to implement each section of the component rather than trying to do it all at once.
- **"Portal first, then Pulumi" workflow.** Work out how to do a task in the **Azure portal** first —
  open the relevant tab, see how it's configured — then ask **Copilot "I'd like that done via Pulumi"**.
  Use the portal as the source of truth for the desired end state.
- **Once setup is done, sit with Copilot to configure.** Go through the **configuration options in the
  APIM doc** and ask Copilot **how each one can be configured** in the component. Treat the doc's config
  list as the checklist to work through.
- **Learn from Email & Services components.** ⭐ **Look at the `Email` and `Services` components** —
  **Nick built these and hit a lot of the same issues we'll have.** They're our best worked examples /
  reference for patterns and gotchas. *(In repo: `Products/Spoke/Email`, `Products/Spoke/Services`.)*

## Hard rules / gotchas ⚠️

- **Delete resources via Pulumi, NOT the Azure portal.** Deleting through the portal **stresses Pulumi
  out** (its state gets out of sync with reality). Always tear down using **Pulumi / the orchestrator
  `Destroy` operation** so state stays consistent.
- **Never press Stop while the orchestrator is running.** **Let it run and fail on its own** if it's
  going to fail. **Interrupting it mid-run causes issues** (leaves things half-created / state
  inconsistent). Patience over Ctrl-C.
- **Always clean up after yourself.** Tear down anything you stand up so we **don't rack up unnecessary
  Azure cost.** Cost discipline is on us for sandbox work.

## My sandbox resource groups (now provisioned) 🟢

Subscription: **`58d847ff-52c8-4127-b663-1232b8fbbd48`** (the sandbox spoke subscription).

| Purpose | Resource group | Link |
| --- | --- | --- |
| **APIM** | `grant-sbx-spk-apimgs-rg` | [Azure portal](https://portal.azure.com/#@dayshape.com/resource/subscriptions/58d847ff-52c8-4127-b663-1232b8fbbd48/resourceGroups/grant-sbx-spk-apimgs-rg/overview) |
| **Testing / Storage** | `grant-sbx-spk-str-rg` | [Azure portal](https://portal.azure.com/#@dayshape.com/resource/subscriptions/58d847ff-52c8-4127-b663-1232b8fbbd48/resourceGroups/grant-sbx-spk-str-rg/overview) |

> These give us a **real place to test end-to-end deploys** — directly answers the open "how do we
> test this?" question raised in [[Meeting Notes - Erdem IaC walkthrough]].

## What this confirms / changes

- **Reinforces [[Decision - do not build on existing APIM IaC]]** — Nick's steer is to *build* the
  component (with Copilot, portal-first), not resurrect the broken code. Consistent with Ivo's steer.
- **Answers the testing linchpin** — we now have named sandbox resource groups under the sandbox
  subscription, so we can run the orchestrator against our own space and clean up after.
- **Gives us worked references** — `Email` and `Services` components are the go-to examples, not APIM.

## Actions / follow-ups

- [ ] Review the **`Email`** and **`Services`** components (`Products/Spoke/`) for patterns + gotchas.
- [ ] Do a **portal-first walkthrough** of a single APIM config task, then translate to Pulumi with Copilot.
- [ ] Work through the **APIM doc config options** with Copilot, one section at a time.
- [ ] Always finish a test session with a **Pulumi `Destroy`** — never delete via the portal.
- [ ] Confirm cost/ownership expectations for `grant-sbx-*` groups with Andy L (still open).

## Related


- [[Decision - do not build on existing APIM IaC]]
- [[Meeting Notes - Erdem IaC walkthrough]]
- [[Target IaC process - componentised framework]]
- [[Meeting prep - Andy L and Erdem]]
