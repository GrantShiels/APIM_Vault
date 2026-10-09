---
ticket: DEVO-28368
related: Phase 10
status: superseded by the code — the dry run has been run
date: 2026-08-10
---

# Portal content capture — dry run

> [!warning] Superseded by the code
> This described a dry run as a plan. It has since been run twice, and the tooling lives in the repo as `apim_portal_content_dryrun.py`.
>
> **Where this document and the code disagree, the code is right.** Kept for the reasoning behind
> the decisions, not as a description of how things behave today.

A single read-only action that collapses most of the open questions in Phase 10. It reads the
master portal content from `vprk2` and writes it to a folder on your machine. **It does not
change vprk2, and it does not touch any customer instance.**

## Why this is worth doing before anything else

Six things we are currently guessing at become facts:

| Question | What the capture tells us |
| --- | --- |
| **How much media is there?** | Whether the Pulumi route is viable at all — see 4.6 in the design doc |
| How many pages are we copying? | Whether a page-by-page approach is sensible or unwieldy |
| What does a snapshot look like on disk? | The format we would need to store as a versioned bundle |
| How big is it? | Whether it belongs in blob storage or could sit in source control |
| Is the content simple built-in widgets? | Confirms the v2-tier limitation works in our favour |
| Can we even reach vprk2 with normal credentials? | Whether the sandbox dependency is a practical problem |

Until we have this, every design choice in Phase 10 is an opinion.

## Safety

`capture.js` calls a single `export()` operation. It reads content and writes it to a local
folder. There is no code path in it that modifies the source service. The destructive scripts
are `cleanup.js` and `generate.js` — **neither is used here.**

## Prerequisites

- `git`, `node` and `npm` on your machine
- Azure CLI, signed in: `az login`
- Read access to the `vprk2` service (see *Possible snags* below)

## Steps

**1 — Point the CLI at the right subscription**

```powershell
az account set --subscription 58d847ff-52c8-4127-b663-1232b8fbbd48
az account show --query "{name:name, id:id}" -o table
```

**2 — Clone Microsoft's developer portal repository**

Somewhere outside the Dayshape repo — this is throwaway.

```powershell
cd $env:TEMP
git clone https://github.com/Azure/api-management-developer-portal.git
cd api-management-developer-portal
git checkout 2.34.0
```

The tag matters. `master` moves, and the version pinned here is the one Voodoo Park's tool uses.

**3 — Install dependencies**

From the repository root, not from `scripts.v3`:

```powershell
npm install
```

**4 — Run the capture**

```powershell
cd scripts.v3
node ./capture `
  --subscriptionId "58d847ff-52c8-4127-b663-1232b8fbbd48" `
  --resourceGroupName "vprk2-sbx-spk-apim-rg" `
  --serviceName "vprk2-sbx-spk-apim" `
  --folder "../dist/snapshot" `
  --timestamp
```

It should finish with `The content was captured in the ... folder.` followed by `DONE`.

**5 — Look at what came back**

```powershell
cd ..\dist
$snap = Get-ChildItem -Directory | Sort-Object Name -Descending | Select-Object -First 1
Get-ChildItem -Recurse $snap.FullName | Measure-Object -Property Length -Sum |
  Select-Object @{n='Files';e={$_.Count}}, @{n='TotalMB';e={[math]::Round($_.Sum/1MB,2)}}
Get-ChildItem $snap.FullName
Get-ChildItem -Recurse $snap.FullName | Group-Object Extension |
  Sort-Object Count -Descending | Select-Object Count, Name
```

## What to record

Paste the output of step 5 back into the session and I will fold it into the design doc. The
four numbers that matter:

1. **Total size** — decides blob storage versus source control
2. **Number of media files** (images: `.png`, `.jpg`, `.svg`, `.gif`) — the Option B blocker
3. **Number of content items** — how many pages we would be managing
4. **Folder shape** — whether media and content are cleanly separated

## Possible snags

**"Insufficient privileges" or a 403.** The portal admin operation needs
`Microsoft.ApiManagement/service/users/token/action` at scope `.../users/1`. Standard
Contributor on the subscription may not include it. Microsoft documents a custom role for this
in the [developer portal FAQ](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-faq).
If this happens, it is itself a useful finding — it means the automated version needs a
specific role assignment on vprk2, which nobody has accounted for yet.

**`npm install` warnings.** Expect noise and audit warnings. Only a hard failure matters.

**Wrong tenant.** If `az account show` shows a different tenant, `az login --tenant <id>` first.

## Afterwards

Delete the clone — it is a scratch copy:

```powershell
Remove-Item -Recurse -Force $env:TEMP\api-management-developer-portal
```

Keep the snapshot folder if it is small; it is the first backup of the master content that has
ever existed outside the instance itself.

## Related

- [[APIM IaC - developer portal content (phase 10)]] — 4.6 is the question this answers
- [[APIM IaC - known traps and gotchas]] — G24, G25, G26
