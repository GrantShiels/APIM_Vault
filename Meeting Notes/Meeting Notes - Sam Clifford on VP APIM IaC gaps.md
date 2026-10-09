---
tags: [apim, iac, context, slack, devo-28368]
updated: 2026-07-20
status: context-note
related-ticket: DEVO-28368
source: "Slack thread reply, 2026-07-20"
---

# Sam Clifford — gaps in the VP (Voodoo Park) APIM IaC

> Context shared by **Sam Clifford** (the engineer with the most
> APIM knowledge — he did the initial pairing/conversations with Voodoo Park) on his return from
> leave. Posted in the spike Slack thread on **2026-07-20**.
> [Slack message](https://dayshape.slack.com/archives/C04PX2AL8SD/p1784540005393999?thread_ts=1783680097.364809&cid=C04PX2AL8SD)

## Why this matters

Sam is the closest thing we have to a subject-matter expert on the **existing (VP) APIM IaC** — the
code we've already agreed to treat as throwaway. His view on **where that code fell short** tells us
what our from-scratch build must get right. Two of his four points are things we hadn't fully
captured yet.

## His four gaps (his priority order)

1. **⚠️ APIM provisioning will cause pipeline timeouts — his #1 practical blocker.**
   Azure can take **~1 hour** to provision an APIM instance. Our DevOps IaC pipelines authenticate
   with **access tokens that expire after 1 hour, with no mechanism to refresh them**. So a single
   provisioning run can outlive its own credentials → **the pipeline fails partway through**.
   - **Candidate fix (from when Sam last looked):** move away from **Entra federated credentials** and
     use **PATs** (personal access tokens) instead. He flagged it's worth exploring other
     alternatives too.
   - This is a **genuinely new risk** for our register — see Q3 in [[APIM IaC - outstanding questions]].

2. **OpenAPI spec selection was baked into the IaC — the piece he'd prioritise.**
   The question he'd tackle first: **how do we deploy the OpenAPI spec (the *software*) onto the
   provisioned IaC?** IaC can be deployed on a **different lifecycle** to the software, so we must make
   sure the **spec version aligns with the Core version** that's actually deployed. Independently
   confirms our **Q2** and the **T7** "spec version as config" direction.

3. **Introduction of new patterns.** VP's IaC brought in **new config patterns** that differ from how
   we handle config everywhere else in the IaC.

4. **Not following any of our IaC patterns** more generally.

## His conclusion

> "To Ivo's point, we should just create this from scratch. Happy to jump on a call to discuss in
> more detail."

Reinforces [[Decision - do not build on existing APIM IaC]] — now backed by **both** Ivo and Sam
(the two people closest to the VP code). Points 3 & 4 are *why* we're not salvaging it.

## Actions for us

- [ ] Take Sam up on the **call** — he's the best person to pressure-test the from-scratch plan,
      especially the provisioning-timeout fix.
- [ ] Add the **provisioning-timeout / token-expiry** risk to the register (done — Q3) and the ticket
      plan risk callouts.
- [ ] Fold the **spec-vs-Core-version lifecycle** point into T7 / Q2 thinking.

## Related

- [[APIM IaC - outstanding questions]] — Q2 (spec update) and Q3 (provisioning timeout).
- [[APIM IaC - ticket breakdown]] — T1 (provisioning), T7 (spec as config).
- [[Decision - do not build on existing APIM IaC]] — build from scratch (Ivo + Sam).

