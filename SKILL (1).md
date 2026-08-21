---
name: copilot-studio-sharing-visibility
description: Use when a flow does not appear in Copilot Studio's Workflows list or "Add a tool" picker, when sharing a new-harness Copilot Studio agent with a colleague (no Share button exists in the new UI), when a publish fails with "does not have ReadAccess ... connectionreference / Referință conexiune", or when deciding what access a colleague needs to see or use a flow/agent.
---

# Copilot Studio visibility & sharing (new harness)

## Overview

Two independent gates control what a user sees in the new Copilot Studio:
(1) the `modernflowtype` field on flows, (2) Dataverse record access (role depth +
explicit shares). Both are fully controllable via API. All rules below were validated
live on 2026-08-20 (env a real production environment, confirmed visually in Studio).

## Flow visibility: `workflow.modernflowtype`

| Value | Label | Effect |
|---|---|---|
| 0 | PowerAutomateFlow | invisible in Studio (default for CLI/API-created flows!) |
| 1 | CopilotStudioFlow | appears in Studio's **Workflows** list |
| 2 | M365CopilotAgentFlow | M365 Copilot variant |

- **Workflows list** = readable flows with `modernflowtype=1` (drafts included, any trigger).
- **"Add a tool" picker** = `modernflowtype=1` AND activated (`statecode=1`) AND trigger
  `Request` / `kind: Skills`.
- Agent-trigger helper flows stay at 0 on purpose (they surface under Triggers instead).
- Solution membership is **irrelevant** to visibility (proven by counter-examples both ways).

Make a CLI-created flow natively visible:
```powershell
PATCH $base/workflows(<workflowid>)  body: {"modernflowtype":1}
```
No shell-flow workaround needed anymore. WARNING: the change is ONE-WAY — patching back
to 0 fails with `InvalidModernFlowOperation: This modern flow's type cannot currently be
changed`. Set 1 only on flows meant to be Studio-visible (tool flows), never on trigger
helper flows.

## Access model per table (this org)

| Table | Read from role | Consequence |
|---|---|---|
| `workflow` | Business-unit level | colleagues' flows visible without any share |
| `bot` | User level only | colleagues' agents invisible unless shared |
| `connectionreference` | None on others' records | designer **publish validates ReadAccess** on every connref in the flow → "does not have ReadAccess … Referință conexiune". Fix: owner opens the record (Access Checker URL from the error) → Share → Read; or GrantAccess. Direct Dataverse PATCH bypasses designer validation. |

Flow co-ownership does NOT propagate a share onto the flow's connection references
(PoaAccessRights stays None) — the morning-of-2026-08-20 publish failure proves it.

Where connref ReadAccess is enforced (all field-tested 2026-08-20):
- designer **publish** of an existing flow → 403 on unreadable connrefs;
- **creating a NEW workflow record** (`POST workflows`, incl. `power-automate create`)
  → 403 on unreadable connrefs, even though PATCH-ing the same reference into an
  EXISTING flow's clientdata is not validated;
- separately, **activation** runs the connector's dynamic validation (e.g. SharePoint
  `GetTable`) through the referenced connection → `Forbidden` if the connection's
  *account* lacks access to the site/resource, regardless of connref record access.
  Tip: the SharePoint `table` parameter accepts the list display name, not just its GUID.

## Sharing a new-harness agent (no UI button — API only)

Grant to the colleague, in this order (validated: agent appeared and opened in
colleague's Studio):

1. the `bot` record — mask `ReadAccess,WriteAccess,AppendAccess,AppendToAccess`
2. **every** `botcomponents` row of that bot — same mask
   (cascade on `botcomponent_parent_bot` is **NoCascade**: shares do NOT propagate)
3. every `connectionreferences` row named `<botschema>.cr.*` — `ReadAccess`

```powershell
# one GrantAccess call per record; repeat for each component/connref
# ($base + auth headers: see copilot-studio-api-client)
POST $base/GrantAccess
{"Target":{"@odata.type":"Microsoft.Dynamics.CRM.bot","botid":"<botid>"},
 "PrincipalAccess":{"Principal":{"@odata.type":"Microsoft.Dynamics.CRM.systemuser",
 "systemuserid":"<userid>"},"AccessMask":"ReadAccess,WriteAccess,AppendAccess,AppendToAccess"}}
```
Target shapes for the other tables:
`{"@odata.type":"Microsoft.Dynamics.CRM.botcomponent","botcomponentid":"<id>"}` and
`{"@odata.type":"Microsoft.Dynamics.CRM.connectionreference","connectionreferenceid":"<id>"}`.

Verify / audit / revoke:
```powershell
GET $base/principalobjectaccessset?$filter=objectid eq <recordid>   # who has what
POST $base/RevokeAccess  (Target + Revokee)                          # undo
```

Anything added to the agent later is NOT covered — re-run step 2 for new components and
step 3 for any new `<botschema>.cr.*` connection references they bring.

## Changing a flow's primary owner (Assign)

Reassigning `ownerid` on a workflow fails with
`does not have the prvChangeOwnerIdOfWorkflow privilege` unless the CALLER's security
role has the Assign privilege on Workflow — record-level `AssignAccess` from GrantAccess
is NOT enough (role privilege is checked first). Makers in this org lack it; only an
admin can reassign. Workaround that avoids the need entirely: the target owner creates
an empty flow themselves (they own it natively), then deploy the real definition over it
with `power-automate deploy` — deploying to an EXISTING flow also skips connref
ReadAccess validation, so the definition may reference the new owner's connection
references directly. Primary ownership rarely matters anyway: co-owners can edit/use,
and runtime uses the embedded connection references, not the owner.

## How the platform itself shares (old-harness pattern)

Old-harness Share creates a Dataverse team named `<botid-without-dashes>_1` (type 0)
holding the members, shared on the bot with mask 786487. These teams also show up as
cryptic co-owners (`e4c2d08039..._1`) on flows used as agent tools. To mimic platform
behavior exactly, add the user to that team instead of direct GrantAccess.

Access-mask decoder: 1 Read · 2 Write · 4 Append · 16 AppendTo · 65536 Delete ·
262144 Share · 524288 Assign. 852023 = full co-owner; 23 = read/write/append; 1 = read-only.

## Common mistakes

- Concluding "wrong flow id" when a colleague's record errors — it's table-level access
  (bot/connref are user-level; workflow is BU-level).
- Sharing only the bot and expecting components to follow (NoCascade).
- Expecting flow co-ownership to fix connectionreference ReadAccess at publish — it does not.
- Chasing solution membership to explain Studio visibility — it never matters; check
  `modernflowtype` and record access instead.
