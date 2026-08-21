---
name: copilot-studio-api-client
description: Use when working with Microsoft Copilot Studio agents (new harness / "github harness") from the command line — listing agents, reading or editing instructions, tools, triggers, publishing, or when the Studio UI lacks an option (sharing, bulk edits). Covers Dataverse bots/botcomponents authoring, PVA actions (PvaPublish, PvaCreateBotComponents), and az-token auth on this machine.
---

# Copilot Studio API Client (new harness)

## Overview

The new Copilot Studio stores ALL agent authoring state in Dataverse. There is no
separate authoring REST API needed: `bots` + `botcomponents` + `connectionreferences`
+ bound PVA actions cover the full lifecycle. Everything below was validated live on
env `Default-<tenantId>` (org `<yourorg>.crm4.dynamics.com`).

**REQUIRED BACKGROUND:** component shapes and YAML formats live in
`copilot-studio-agent-anatomy`. Flow visibility rules and agent sharing live in
`copilot-studio-sharing-visibility`. Flows themselves are managed with the
`power-automate-api-client` skill (same venv, same auth).

## Auth recipe (this machine)

`az` is the pip-installed `az.bat` inside the project venv — it must be on PATH:

```powershell
$env:PATH = "<path-to-venv>\Scripts;$env:PATH"
$tok = (az account get-access-token --resource "https://<yourorg>.crm4.dynamics.com" --output json | ConvertFrom-Json).accessToken
$h = @{ Authorization = "Bearer $tok"; "Content-Type" = "application/json" }
$base = "https://<yourorg>.crm4.dynamics.com/api/data/v9.2"
```

Tokens are short-lived, never persisted. Re-acquire per session. Call pattern for
everything below: `Invoke-RestMethod -Headers $h -Uri "$base/..."` (add `-Method Patch/Post`
and `-Body $json` for writes).

## Quick reference

| Task | Call |
|---|---|
| List agents | `GET $base/bots?$select=name,schemaname,statecode,publishedon` |
| Read agent config (instructions, model, channels) | `GET $base/bots(<botid>)?$select=configuration` — JSON, see anatomy skill |
| Edit instructions | `PATCH $base/bots(<botid>)` body `{"configuration":"<full JSON string>"}` — instructions text lives at `agentSettings.instructions.segments[].value`; parse with ConvertFrom-Json, modify, re-serialize the WHOLE object with `ConvertTo-Json -Depth 20 -Compress` (escaping handled for you) |
| List an agent's components | `GET $base/botcomponents?$filter=_parentbotid_value eq <botid>` |
| Read/edit a tool | `GET/PATCH botcomponents(<id>)` — YAML in `data`, description in `description` |
| Create component | `POST $base/botcomponents` with `parentbotid@odata.bind` = `/bots(<botid>)` |
| Batch create components | `POST $base/bots(<botid>)/Microsoft.Dynamics.CRM.PvaCreateBotComponents` param `BotComponents` |
| Publish agent | `POST $base/bots(<botid>)/Microsoft.Dynamics.CRM.PvaPublish` (returns job) then `POST bots(<botid>)/...PvaPublishStatus` body `{"PublishBotJob":"<job string from PvaPublish>"}` — both bound on bot |
| Test conversation endpoint | `POST bots(<botid>)/...PvaGetDirectLineEndpoint`; unbound `PvaStartConversation` (Version, Request) |
| Delete agent | `PvaDeleteBot` (bound on bot) — destructive, confirm first |
| Who has access to a record | `GET $base/principalobjectaccessset?$filter=objectid eq <recordid>` |
| Grant access | `POST $base/GrantAccess` (see sharing-visibility skill for the full agent-share recipe) |

## Rules

- **Never edit while the Studio visual designer is open** — a Save in the designer
  regenerates state and silently overwrites API edits (proven failure mode). There is no
  programmatic check for this: ask the user to close the designer tab before writing.
- Copy component shapes 1:1 from a working example (anatomy skill). Wrong shapes are
  NOT rejected — they save fine and are silently invisible in the Studio UI.
- `bots.configuration` is one big JSON string: always GET → parse → modify → PATCH the
  whole string. Never hand-splice substrings.
- Dataverse writes (PATCH/POST/GrantAccess) trip the session permission gate — expect
  approval prompts; keep write commands small and single-purpose (compound scripts with
  function definitions get blocked more often than simple inline calls).
- Read access differs per table: `workflow` = business-unit-wide from role;
  `bot` and `connectionreference` = own records + explicit shares only. If a colleague's
  record 404s/permission-errors, it's access, not a wrong id.
- After changing components or configuration, the agent serves the OLD version until
  `PvaPublish` (or a manual publish in Studio).
- `PvaPublish` is field-tested (2026-08-20): POST with body `{}` succeeds; the response has
  empty `PublishedBotContentId` and null `PublishBotJobResponse` even on success. Publishing
  is ASYNCHRONOUS — `publishedon` updates ~15-30s after the call; poll it to confirm. `PvaStartConversation`
  remains untested.
- `botcomponents.schemaname` max length is 100 — long ExternalTriggerComponent names must be
  truncated to exactly 100 chars (the platform does the same; uniqueness is all that matters).

## Finding an agent's id

```powershell
GET $base/bots?$select=name,botid,schemaname&$filter=contains(name,'<partial>')
```
`schemaname` (e.g. `ai_genereazaminutasedintei_X-WO6U`) is what flows pass to the
`ExecuteCopilot` action as the `Copilot` parameter — not the GUID.

## Common mistakes

- Querying `bots(<id>)/bot_botcomponent` — that N:N returns nothing; use the
  `_parentbotid_value` filter on `botcomponents` instead.
- Expecting share on a bot to cascade to its components: `botcomponent_parent_bot`
  cascade is **NoCascade** — grant each component explicitly.
- Treating a missing agent in `GET bots` as "deleted": bots are user-level read;
  a colleague's agent simply isn't visible to you without a share.
- Using the bot GUID where the schemaname is expected (ExecuteCopilot, component
  schemaname prefixes).
