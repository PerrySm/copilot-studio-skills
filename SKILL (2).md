---
name: copilot-studio-agent-anatomy
description: Use when reading or writing the internals of a Copilot Studio agent (new harness) — bots.configuration JSON, botcomponents types, tool YAML (WorkflowTool, ConnectorTool, McpTool), agent triggers (ExternalTriggerConfiguration + ExecuteCopilot flow), connection reference naming, or when a component saved via API does not show up in the Studio UI.
---

# Copilot Studio agent anatomy (new harness)

## Overview

Every shape below was extracted 1:1 from working agents in env
`Default-<tenantId>` — copy them exactly. Components with
invented shapes save without error but are **silently invisible** in the Studio UI.

## Storage map

| Studio concept | Dataverse location |
|---|---|
| Agent | `bots` row: `name`, `schemaname`, `configuration`, `iconbase64`, `publishedon` |
| Instructions / model / channels / web search | inside `bots.configuration` JSON |
| Tool or Topic V2 | `botcomponents` componenttype **9**, lookup `parentbotid`, YAML in `data` |
| Agent trigger | `botcomponents` componenttype **17** + a hidden helper flow (below) |
| Knowledge source | componenttype 16 |
| Copilot settings | componenttype 18 |
| Tool connections | `connectionreferences` named `<botschema>.cr.<connector>[.<connectionid>]` |

componenttype values: 0 Topic, 1 Skill, 2 BotVariable, 3 BotEntity, 4 Dialog,
5 Trigger, 6 NLU, 7 NLG, 8 DialogSchema, 9 Topic(V2)/tools, 10 Translations(V2),
11 Entity(V2), 12 Variable(V2), 13 Skill(V2), 14 FileAttachment, 15 CustomGPT,
16 KnowledgeSource, 17 ExternalTrigger, 18 CopilotSettings, 19 TestCase, 20 CustomIndicator.

## bots.configuration (JSON string)

Key paths (real example, agent on model Sonnet5):
```json
{
  "channels": [{"id": "MsTeams", "channelId": "MsTeams"}, {"id": "Microsoft365Copilot", "channelId": "Microsoft365Copilot"}],
  "$kind": "BotConfiguration",
  "recognizer": {"$kind": "CLICopilotRecognizer"},
  "agentSettings": {
    "$kind": "AgentSettings",
    "model": {"$kind": "ModelConfig", "series": "Sonnet5"},
    "instructions": {"$kind": "Instructions",
      "segments": [{"$kind": "StaticSegment", "value": "<full instruction text>"}]},
    "web": {"$kind": "WebSettings", "enableWebSearch": false}
  }
}
```
Instructions = `agentSettings.instructions.segments[].value`. Always rewrite the whole
JSON string on PATCH.

## Tools (botcomponents type 9)

schemaname: `<botschema>.tool.<Name>_<3charSuffix>`. Human description goes in the
record's `description` column. YAML in `data`:

**WorkflowTool** (a Skills-trigger flow as tool):
```yaml
kind: WorkflowTool
workflowId: <Dataverse workflowid GUID>
toolInputs:
  - name: text                # trigger schema property name
    displayName: NumeFisier
    description: <what the model should pass here>
toolOutputs:
  - name: status
    displayName: status
connectionProperties:
  $kind: ConnectionProperties
  diagnostics:
  mode: Invoker               # or Maker
```

**ConnectorTool** (single connector action):
```yaml
kind: ConnectorTool
authMode: Maker               # or Invoker
connectionReference: <botschema>.cr.shared_jira.<connectionid>
connectorId: /providers/Microsoft.PowerApps/apis/shared_jira
operationId: ListIssues
# optional pre-bound inputs:
toolInputs:
  - name: emailMessage.To
    value:
      kind: ContextVariableReference
      name: SendanemailV2.emailMessage_To
```

**McpTool** (MCP server):
```yaml
kind: McpTool
authMode: Invoker
connectionReference: <botschema>.cr.<connector>.<connectionid>
connectorId: /providers/Microsoft.PowerApps/apis/shared_rovo-20mcp-20server-...
operationId: InvokeServer
```

Validation note (2026-08-20): a WorkflowTool component added through the new-UI picker is
structurally IDENTICAL to the recipe above (UI default: `mode: Invoker`; descriptions are
copied from the flow's description/schema titles). One open question: an API-created tool
component did not render in an already-open new-UI agent page while a UI-created twin did —
if your API-created tool doesn't appear, hard-refresh the page before assuming a bad shape,
and verify the record via API. The new-harness agent page has NO Triggers section (only the
classic designer shows one) — a type-17 component is invisible there, but the trigger
mechanism (helper flow → ExecuteCopilot) fires regardless of UI visibility.

**HARD LIMIT discovered 2026-08-20 (classic harness):** creating the type-17 component
via API makes `PvaPublish` FAIL with `InvalidReferenceError: CloudFlow with id '<x>' not
found` — even when flowId is the correct Dataverse workflowid (also fails with resourceid;
sharing the flow to the bot's `<botid>_1` team and modernflowtype changes don't fix it).
The publish validator resolves CloudFlows through a PVA-internal registry that only the
Studio trigger wizard populates. RECIPE THAT WORKS: create the trigger in the Studio
wizard (it generates the helper flow + component), THEN customize the generated flow via
`power-automate deploy` (add conditions, rewrite the ExecuteCopilot message). Note the
wizard also shares the helper flow with the bot's team (mask 852023) — replicate that if
you ever hand-wire flows to agents. The API-created type-17 recipe below is verified only
on paper for classic; on the NEW harness there is no publish-time wizard either way.

## Agent triggers = 2 pieces

Real example: "When a new email arrives (V3)" on agent `contoso_exampleAgent`.

**Piece 1 — a normal solution-aware flow** (kept `modernflowtype=0`, so hidden from the
Workflows list) with the event trigger plus exactly one action:
```json
"Sends_a_prompt_to_the_specified_copilot_for_processing": {
  "inputs": {
    "host": {"apiId": "/providers/Microsoft.PowerApps/apis/shared_microsoftcopilotstudio",
             "connectionName": "shared_microsoftcopilotstudio", "operationId": "ExecuteCopilot"},
    "parameters": {
      "Copilot": "<bot schemaname>",
      "body/message": "Use content from @{triggerBody()}"
    }
  }, "runAfter": {}, "type": "OpenApiConnection"
}
```
Example Copilot Studio connection reference:
`<your_copilotstudio_connref>`.

Wire the connector in the flow's `connectionReferences` exactly like any other connector
(real example from the working trigger flow):
```json
"shared_microsoftcopilotstudio": {
  "api": {"name": "shared_microsoftcopilotstudio"},
  "connection": {"connectionReferenceLogicalName": "<your_copilotstudio_connref>"},
  "runtimeSource": "embedded"
}
```
CLI/API-created flows default to `modernflowtype=0`, so nothing to do to keep the helper
flow hidden (see `copilot-studio-sharing-visibility`).

**Piece 2 — botcomponent type 17** on the agent, schemaname
`<botschema>.ExternalTriggerComponent.<TruncatedName>.<guid>`, `data`:
```yaml
kind: ExternalTriggerConfiguration
externalTriggerSource:
  kind: WorkflowExternalTrigger
  flowId: <Dataverse workflowid of piece 1>

extensionData:
  flowName: <ProcessSimple flow id of piece 1>
  flowUrl: /providers/Microsoft.ProcessSimple/environments/<envId>/flows/<flowName>
  triggerConnectionType: Office 365 Outlook   # UI label only
```

Field sources for piece 2:
- `flowId` = the flow's Dataverse `workflowid`.
- `flowName` = the flow's **`resourceid`** column on the same `workflows` row (verified:
  equals the ProcessSimple flow id used in `flowUrl` and run URLs). `resourceid` is EMPTY
  while the flow is a never-activated draft — activate the flow first, then read it.
- `<envId>` = the literal environment name incl. prefix, e.g.
  `Default-<tenantId>`.
- `triggerConnectionType` = free-text UI label; use the connector display name
  (e.g. `SharePoint`).

Naming conventions (observed, not documented by Microsoft): tool `<Name>` = display name
with spaces/diacritics stripped (e.g. "Adu transcript sedinta v2" →
`Adutranscriptsedintav2`), suffix = 3 random alphanumerics for uniqueness (`_1KX`).
Type-17 `<TruncatedName>` = display name similarly stripped and truncated (observed ~19
chars: `Whenanewemailarrive`), `<guid>` = a fresh GUID (the platform's own observed value
is even truncated to 35 chars — uniqueness is all that matters). Copy blank lines and the
empty `diagnostics:` key exactly as shown in the examples.

For proactive delivery: the recipient must have opened the agent in Teams at least once
and the agent must be published to the Teams channel.

## Common mistakes

- Inventing YAML fields or schemaname patterns — component saves but never appears in UI.
- Putting the bot GUID in `ExecuteCopilot`'s `Copilot` parameter — it takes the **schemaname**.
- Editing tool YAML while the Studio designer is open — designer Save overwrites it.
- Forgetting `PvaPublish` after edits — runtime keeps serving the old published version.
