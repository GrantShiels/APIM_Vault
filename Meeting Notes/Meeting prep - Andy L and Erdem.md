---
tags: [apim, meeting-prep, iac, devo-28368]
updated: 2026-07-09
---

# Meeting prep — Andy L & Erdem

> Prep for the two conversations Iain set up. Context:
> [[Target IaC process - componentised framework]].

## Andy L — Infra/SRE (Infra Platform) lead

Focus: ownership boundaries + how APIM fits the IaC release machinery his team owns.

- Your team owns `iac-main`, RC tagging, promotion PRs and the deploy pipelines — **where does
  Ecosystem's responsibility end and yours begin** for the APIM component?
- The APIM component uses `DayshapeComponent` and convention-compliant `spk-apim-infra-*` keys, but
  `__main__.py` is a **manual dev harness** with hardcoded pentest values. **What's needed to run
  it through the IaC orchestrator** instead (clientid/env + App Config)?
- Config source of truth is **App Config (ACS)** — is there **tooling/precedent for seeding a
  component's keys per customer/env**, and how do **Key Vault secret references** get provisioned?
- The `ds_manifest.xml` has **no dependencies** — what **dependencies** should APIM declare
  (resource group, core app, Entra, App Insights, Key Vault) for correct ordering?
- **Who wrote this component, and why was it never adopted into the pipeline?** (Confirms
  *complete-and-wire* vs *rebuild*.)
- What **support / pairing** can your team offer while we ramp up on Pulumi/the framework?
- **Monitoring/alerts (DEVO-28372)** — is there a standard **alert/action-group pattern** for
  Spoke components we adopt?
- **Rollback** — how does versioned rollback work in practice for a Spoke component?
- What **access** (Azure, App Config, ADO pipelines, `iac-main`) will Ecosystem need?

## Erdem — similar project

Focus: learn from someone who's already built/adapted a component in this framework.

- What did **building/deploying your component** actually involve end-to-end?
- How **steep was the Pulumi/Python learning curve** coming from app-dev? What tripped you up?
- Which **docs/tools were most useful**, and which were missing or out of date?
- Any **reusable patterns/helpers** we can copy for APIM?
- How did you handle **component-specific config + secrets**?
- How long did it realistically take from **zero → deploying to a non-prod env**?
- Anything you'd **do differently** knowing what you know now?

## Related

- [[Target IaC process - componentised framework]]
- [[Meeting Notes - Iain Macnab outcome]]
