---
tags:
  - apim
  - iac
  - research
  - developer-portal
ticket: DEVO-28368
status: settled
date: 2026-08-11
updated: 2026-08-11
---

# APIM IaC — portal content, why there is no IaC-native route

> [!abstract] The whole thing in four sentences
> We asked whether the developer portal could be copied between APIM instances declaratively — as
> Pulumi resources — instead of shelling out to Microsoft's Node.js scripts from our deployment.
> The answer is no, and the reason is media: uploaded images and PDFs live in a storage account
> Microsoft owns, reachable only through a one-hour key handed out by an API call, so no
> infrastructure tool can describe them. Text content genuinely is addressable, and Pulumi even ships
> a resource for it, but a half-copy that reproduces every page and loses every logo is worse than no
> copy at all. Shelling out to `scripts.v3` is not a compromise we settled for — it is the only
> complete route that exists.

---

## Part 1 — The question

Phase 10 copies Dayshape's portal design from the `vprk2` master onto each customer's APIM instance.
Our plan runs Microsoft's PowerShell/Node.js migration script from a Pulumi deployment hook.

That has a smell to it: a `git clone` of a Microsoft repository and an `npm ci` in the middle of a
customer deployment. Before building it, we checked whether Pulumi could do this properly — portal
content as declared resources, like everything else we deploy.

---

## Part 2 — The short answer

**No.** Verified against the Pulumi provider we have installed, against live `vprk2`, and against
Microsoft's current documentation.

| Candidate | What it really manages | Solves this? |
| --- | --- | --- |
| `azure-native.apimanagement.ContentItem` | Pages, layouts, styles, navigation — the JSON documents | ⚠️ **Text only** |
| `azure-native.apimanagement.ContentType` | The *schema* for content, not content itself | ❌ |
| `azure-native.apimanagement.Documentation` | Docs attached to an individual API | ❌ |
| `PortalRevision` (ARM) | Publishing a draft — no Pulumi resource generated for it | ❌ Publish only |
| Terraform `azurerm_api_management` `developer_portal` block | The custom **hostname**, nothing else | ❌ |
| Terraform / Bicep / ARM generally | Instance configuration | ❌ |
| Microsoft `scripts.v3` | Text **and** media, plus publish | ✅ **Only complete option** |

> [!danger] The distinction that trips people up
> Portal **settings** — sign-in, sign-up, delegation, CORS, custom domain — *are* well covered by IaC,
> and we already manage them that way. Portal **content** — pages, layout, images — is not. Blog
> posts and even module READMEs conflate the two constantly. Anything claiming "developer portal in
> Terraform" is almost always talking about settings.

---

## Part 3 — Why the gap exists

Portal content is stored in two completely different places.

**Text** lives in APIM's own content store and *is* a proper ARM resource
(`…/contentTypes/{type}/contentItems/{id}`). That is why Pulumi has a `ContentItem` resource for it,
with an untyped `properties` passthrough. Declaring pages as Pulumi resources is genuinely possible.

**Media** does not. Images and PDFs sit in an Azure Storage container, and the only way to reach it is:

```
POST …/portalconfigs/default/listMediaContentSecrets
  → returns a SAS URL, valid about an hour
```

Three things make that impossible to model declaratively:

1. **The storage account is not ours.** Verified directly — the container behind `vprk2` lives on
   `apimstj2nsbmrqwjjbe5vpnf`, which returns *"Storage account not found"* when queried against our
   subscription. It belongs to the APIM service itself.
2. **The credential is ephemeral.** A one-hour SAS fetched at runtime is not something a resource
   graph can hold.
3. **It is a `POST` action, not a readable property.** Which is also why a read-only role cannot copy
   media — see [[APIM IaC - known traps and gotchas]] G27.

So a hand-rolled Pulumi implementation would reproduce every page and lose every logo. For a
customer-facing portal that is arguably a worse outcome than not copying at all — the pages would
look broken rather than absent.

There is a second obstacle worth knowing: the content store has to be **initialised** before anything
can be written to it, and that initialisation is triggered by a human opening the portal in admin mode.
There is no documented API for it. A public Terraform module hit exactly this and gave up, documenting
that it deliberately does not publish the portal for this reason.

---

## Part 4 — What Microsoft currently says

The guidance page was updated **25 June 2026** and still points at one thing only: `scripts.v3/migrate.js`.
No Bicep, no Terraform, no ARM alternative is offered. Nothing has superseded it.

It also repeats the warning already recorded as G26:

> *"Migration of portal content between instances in the v2 tiers is also not supported."*

We run BasicV2 at both ends. This changes nothing about our plan — it confirms what we already knew —
but it now has a current citation rather than one from October 2025.

> [!question] A documentation bug worth not falling for
> The same page says self-hosted portals with custom storage *"need to use the **original**
> `scripts.v3/migrate.js` script"*, then immediately says *"the original script doesn't work for
> managed portals… use the script from the `/scripts.v3` folder instead."*
>
> Both sentences link to the same folder, so the page appears to contradict itself. It reads as though
> there is a newer script we should be using and are not. There isn't — this is a stale link from
> before the older `scripts/` folder was retired. **We are on the correct path.** Anyone reading that
> page cold will lose half an hour to it.

One line there is genuinely useful: the script *"optionally and for managed portals only: automatically
publishes the portal."* Ours is a managed portal, so the script's own `publish: true` handles
publishing — which independently supports removing the separate Pulumi publish resource.

---

## Part 5 — Community options

None worth adopting. GitHub searches for Pulumi or Terraform approaches to portal *content* returned
nothing maintained. The one real-world Terraform module found covers settings only and explicitly
documents that it cannot automate the content side.

---

## Part 6 — Worth watching

- **If Microsoft ever exposes portal media as an addressable resource**, this whole decision reopens.
  No sign of it on any roadmap.
- **v2-tier migration support** — the likeliest thing to change, since it works in practice and looks
  like a tooling/doc gap rather than a technical barrier.
- **The Fluent UI portal refresh** (instances created after June 2025) — unclear whether it alters the
  content storage model. Worth watching `scripts.v3/utils.js` for changes.

---

## Part 7 — Where this leaves us

The plan does not change. What changes is our confidence in it: shelling out to `scripts.v3` was the
pragmatic choice, and it is now the *evidenced* one. If anyone asks why a customer deployment clones a
Microsoft repository, the answer is that the alternative loses every image.

Two things follow for the build:

- **The script publishes for managed portals**, reinforcing the decision to remove the separate Pulumi
  publish resource and rely on a single publish.
- **`ContentItem` being real is still useful** — not for copying, but it confirms the read side of our
  change-detection fingerprint sits on supported, stable surface.

---

## For the Product Owner — the plain version

We checked whether the customer-facing API portal could be set up by our normal automated tooling,
rather than by running a script Microsoft publishes.

It can't, and the reason is simple: the portal's text is stored somewhere our tools can reach, but its
**images are not**. They sit in storage Microsoft owns, and the only way to get at them is a temporary
pass that lasts about an hour. Our tools can't work with something that expires.

If we built it ourselves anyway, every customer's portal would have all the right words and none of the
Dayshape branding — which looks broken rather than simply unfinished.

So we will keep using Microsoft's script, which handles both. That was already the plan; this work
confirms it is the only complete option rather than merely the easiest one.

---

## Related

- [[APIM IaC - developer portal content (phase 10)]] — the design this supports
- [[APIM IaC - portal content permissions (phase 10)]] — the access it needs, and why read-only fails
- [[APIM IaC - known traps and gotchas]] — G24, G25, G26, G27
- [[APIM Findings Summary]]
