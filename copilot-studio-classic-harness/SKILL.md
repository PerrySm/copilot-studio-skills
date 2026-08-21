---
name: copilot-studio-classic-harness
description: Use when working with CLASSIC (old harness) Copilot Studio agents via API — topics/AdaptiveDialog YAML, .gpt.default instructions, .action. flow tools (TaskDialog/InvokeFlowTaskAction), knowledge files, or when migrating an agent between the new "github harness" and the classic experience, or when deciding which harness an agent is on (billing/credits, Triggers UI, Share UI differences).
---

# Copilot Studio classic harness (old experience)

## Overview

Classic agents use the same Dataverse storage (`bots` + `botcomponents`) as new-harness
agents but with DIFFERENT component shapes and a different `configuration` layout. All
shapes below extracted from live working agents (env a real production environment, 2026-08-20).

**REQUIRED BACKGROUND:** auth + call patterns in `copilot-studio-api-client`; new-harness
shapes AND the 2-piece external-trigger recipe in `copilot-studio-agent-anatomy`;
visibility/sharing in `copilot-studio-sharing-visibility`.

Conventions used below: `<botschema>` = the bot record's `schemaname` column. Component
edits are `PATCH botcomponents(<id>)` with body `{"data":"<full YAML string>"}` (serialize
with ConvertTo-Json so escaping is handled).

## Telling the harnesses apart (from `bots.configuration`)

| Field | Classic (old) | New (github harness) |
|---|---|---|
| `recognizer.$kind` | `GenerativeAIRecognizer` | `CLICopilotRecognizer` |
| Instructions | botcomponent type 15 `.gpt.default` | `configuration.agentSettings.instructions` |
| Model | `.gpt.default` → `aISettings.model.modelNameHint` (e.g. `GPT5Auto`) | `agentSettings.model.series` (e.g. `Sonnet5`) |
| Config extras | `gPTSettings.defaultSchemaName`, `settings.GenerativeActionsEnabled`, `isAgentConnectable` | `channels[]`, `agentSettings.web` |
| Triggers UI | YES (classic designer) | NO section — API only |
| Share UI | YES (creates team `<botid>_1`) | NO — API only (GrantAccess recipe) |
| Billing | classic licensing | Copilot credits after grace period |

## Component map (classic)

| Component | type | schemaname pattern | data YAML kind |
|---|---|---|---|
| Topic | 9 | `<botschema>.topic.<Name>` | `AdaptiveDialog` |
| Flow tool ("action") | 9 | `<botschema>.action.<Name>` | `TaskDialog` |
| Instructions/GPT | 15 | `<botschema>.gpt.default` | `GptComponentMetadata` |
| Knowledge source | 16 | `<botschema>.topic.<file>_<suffix>` | (file metadata) |
| File attachment | 14 | `<botschema>.file.<name>_<suffix>` | |
| Global variable | 12 | `<botschema>.GlobalVariableComponent.<Name>` | |
| Test case | 19 | `mspva_<guid>` | |
| External trigger | 17 | same 2-piece recipe as new harness (see agent-anatomy — it was extracted FROM a classic agent) | `ExternalTriggerConfiguration` |

## Instructions component (`.gpt.default`, type 15)

```yaml
kind: GptComponentMetadata
instructions: "<full instruction text, one quoted YAML scalar>"
gptCapabilities:
  webBrowsing: true

aISettings:
  model:
    kind: PreviewModels
    modelNameHint: GPT5Auto

declarativeSkillsMetadata:
```
Edit instructions = PATCH this component's `data`. The bot's `configuration` points to it
via `gPTSettings.defaultSchemaName`.

## Flow tool ("action", type 9, `.action.<Name>`)

```yaml
kind: TaskDialog
modelDisplayName: Verificare documente neavizate      # optional — name shown to the model
modelDescription: <when/why the model should call it>  # optional
outputs:
  - propertyName: documenteneavizate

action:
  kind: InvokeFlowTaskAction
  flowId: <Dataverse workflowid of a Skills-trigger flow>
  connectionProperties:
    $kind: ConnectionProperties
    diagnostics:
    mode: Invoker        # or Maker

outputMode: All
```
**CRITICAL (field-tested 2026-08-20): do NOT create `.action.` components via API.** They
save, render in the Studio UI, and survive PvaPublish — but at runtime the agent fails
with `FlowNotFound: Fluxul cu ID '<x>' nu a fost găsit în definiția robotului`, because
the PVA-internal flow registry is populated ONLY by the Studio "Add action/tool" picker
(same registry that breaks API-created type-17 triggers at publish). RECIPE: wire every
flow action through the Studio UI picker, then use the API to edit/rename the generated
component or redeploy the flow definition. The YAML below is for READING/EDITING
UI-created actions, not for creating them from scratch.

Inputs are NOT declared here — the platform reads them from the flow's Skills trigger
schema (titles/descriptions on trigger inputs are what the model sees). Each
`outputs.propertyName` must match a property name in the flow's Skills `Response` body.
`mode`: `Invoker` (UI default) calls the flow on behalf of the chatting user — required
for per-user flows with run-only setups; `Maker` pins the invocation to the maker. The
flow's INTERNAL connections follow its own connectionReferences (embedded/invoker),
not this field. A minimal variant with only `outputs` + `action` + `outputMode` also
works in production.
NOTE: `.action.` components are rendered ONLY by classic agents — a new-harness agent
ignores them (it uses `.tool.` WorkflowTool instead), and vice versa.

## Topic (type 9, `.topic.<Name>`)

```yaml
kind: AdaptiveDialog
startBehavior: CancelOtherTopics
beginDialog:
  kind: OnEscalate            # system topics: OnEscalate/OnConversationStart/...;
  id: main                    # custom topics: OnRecognizedIntent
  intent:
    displayName: Escalate
    triggerQueries:
      - Talk to agent
      - ...
  actions:
    - kind: SendActivity
      id: sendMessage_<suffix>
      activity: |-
        <message text>
```
Topics are full dialog programs — for generative-orchestration agents you rarely need to
author them via API beyond enabling/disabling; the heavy lifting sits in `.gpt.default`
instructions + `.action.` tools.

## `bots.configuration` (classic)

```json
{
  "$kind": "BotConfiguration",
  "settings": {"GenerativeActionsEnabled": true},
  "isAgentConnectable": true,
  "gPTSettings": {"$kind": "GPTSettings", "defaultSchemaName": "<botschema>.gpt.default"},
  "aISettings": {"$kind": "AISettings", "useModelKnowledge": true,
                 "isFileAnalysisEnabled": true, "isSemanticSearchEnabled": true,
                 "contentModeration": "Low", "optInUseLatestModels": false},
  "recognizer": {"$kind": "GenerativeAIRecognizer"}
}
```

## Migration recipe: new harness → classic

1. Source content: the new UI's **Download** button exports the agent YAML; equivalently
   read `bots.configuration` (instructions) + `.tool.` components via API.
2. The target owner creates a blank agent in the CLASSIC experience (UI) — bot shell
   creation via API (`PvaProvision`) is untested; UI is 2 minutes and sets ownership right.
   If someone OTHER than the owner performs steps 3-5 via API, the owner must first share
   the bot + components with them (GrantAccess recipe in sharing-visibility) or use the
   native classic Share UI — ordering matters: share BEFORE the API edits.
   Model note: classic runs GPT-family models (`modelNameHint`), new harness runs e.g.
   Sonnet5 — behavior/tone will shift; re-test instructions after migration.
3. Via API on the new bot: PATCH `.gpt.default` data with the instructions; POST
   `.action.<Name>` components (type 9) for each WorkflowTool, pointing at the same
   flowIds; knowledge files re-uploaded via UI.
4. Triggers: use the classic designer's Triggers UI (it builds the helper flow +
   type-17 component for you), or the API recipe from agent-anatomy. Confirmed available
   in the wizard (2026-08-20): SharePoint "When an item is created" / "item or file
   modified", Teams "When a new channel message is added". The wizard warns triggers are
   a BILLABLE feature ("will consume messages") — keep polling intervals moderate
   (3-5 min) and add trigger conditions where possible.
5. Publish (`PvaPublish` or UI) and share via the native Share UI (or GrantAccess).
6. Flows themselves need no migration — the same Skills-trigger flow serves either
   harness; only the wrapper component differs (`.tool.` vs `.action.`).

## Common mistakes

- Creating `.tool.` components on a classic agent or `.action.` on a new-harness agent —
  both save silently and never render.
- Editing instructions in `bots.configuration` on a classic agent — they live in the
  `.gpt.default` component there, not in configuration.
- Assuming the harness from the agent's age — check `recognizer.$kind` instead.
