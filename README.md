# copilot-studio-skills

Agent skills for programmatic authoring of Microsoft Copilot Studio agents, through the Dataverse API. Reverse engineered and field tested on a live production tenant in August 2026. Most of what is documented here does not exist in Microsoft's official docs.

Written as [Claude Code skills](https://docs.claude.com/en/docs/claude-code), but the recipes are plain HTTP calls against Dataverse. You can use them with curl and a bearer token if you want.

## Why this exists

Microsoft is currently running two Copilot Studio experiences in parallel. The new one (internally a different harness, visibly a different UI) is where all new features land. It is also missing basic things: there is no Share button, no Triggers section, and no way to see why your API-created component silently refuses to render.

I needed to share a new-harness agent with a colleague. There was no button for it. So I went under the hood. The hood turned out to be Dataverse all the way down, and once you know the tables, you can do more from the API than from either UI.

## The two harnesses

Every agent is a row in the `bots` table plus child rows in `botcomponents`. The two experiences use the same tables with different shapes. You can tell them apart from one field:

```
GET bots(<id>)?$select=configuration
configuration.recognizer.$kind
  "GenerativeAIRecognizer"  -> classic (old harness)
  "CLICopilotRecognizer"    -> new harness
```

Do not guess from the agent's age. Check the field.

| | Classic (old) | New harness |
|---|---|---|
| Instructions | botcomponent type 15, `<botschema>.gpt.default`, YAML | `bots.configuration` JSON, `agentSettings.instructions` |
| Model | GPT family (`modelNameHint: GPT5Auto` etc.) | Claude family (`series: Sonnet5`, `claude-opus-5` etc.) |
| Flow tools | botcomponent type 9, `.action.<Name>`, kind TaskDialog | botcomponent type 9, `.tool.<Name>`, kind WorkflowTool |
| Triggers UI | yes, full wizard | none. API only |
| Share UI | yes, native button | none. API only |
| Billing | classic licensing | Copilot credits after the grace period |
| Topics | AdaptiveDialog YAML, full designer | reduced role |

The flows behind tools do not care which harness calls them. A Skills-trigger Power Automate flow serves either side. Only the wrapper component differs.

## What you can do from the API

Everything below is field tested, not inferred:

- List every agent you can see, read and write instructions, publish (`PvaPublish`, bound action, async: poll `publishedon`).
- **Share a new-harness agent** even though the UI has no button: `GrantAccess` on the bot row, on every `botcomponents` row, and ReadAccess on the `<botschema>.cr.*` connection references. The bot-to-component relationship is NoCascade, so sharing the bot alone does nothing for the components. Verified end to end: the agent showed up and opened in a colleague's Studio.
- Make a CLI-created flow appear natively in Studio's flow picker: `PATCH workflows(<id>) {"modernflowtype": 1}`. This field is the entire visibility mechanism. Solution membership is irrelevant, proven by counterexamples both ways. Warning: the change is one way. Patching back to 0 fails with `InvalidModernFlowOperation`.
- Read the anatomy of any tool, topic or trigger and edit it in place.

## What you cannot do from the API

This part cost a day, so read it twice.

Copilot Studio keeps an internal registry of which flows belong to which agent. Only the Studio UI writes to it. Consequences:

- A trigger component (type 17) created via API passes validation, saves, and then `PvaPublish` fails with `CloudFlow with id '<x>' not found`. The id is correct. The registry entry is missing. Nothing you patch on the workflow row fixes it.
- A tool component created via API saves, renders in the UI, survives publish, and then dies at runtime with `FlowNotFound`. Same registry, different symptom. The component looks alive. It is not.

The working split: **wire tools and triggers through the UI once, then do everything else through the API.** UI-created components can be freely edited, renamed and redeployed programmatically, and the flows behind them can be fully rewritten with a deploy. The registry entry survives all of that.

## Migrating an agent between harnesses

I migrated a production agent from new to classic in about an hour. Direction matters for the reasons, not for the mechanics.

Why we went new to classic: after the grace period the new harness bills per message in Copilot credits. The classic one does not. Classic also has the Triggers wizard and the Share button. For a production agent used by a whole department, predictable cost plus working UI beat a nicer model.

What you give up: the model. Our source agent ran on claude-opus-5. Classic runs GPT. Tone and reasoning shift. Retest your instructions after migration, do not assume they behave the same.

The mechanics:

1. Export the source. The new harness has a Download button that gives you the full agent YAML. Or read `bots.configuration` and the `.tool.` components via API. Same content.
2. The target owner creates a blank agent in the classic experience. Two minutes in the UI. This also sets ownership correctly, which matters because reassigning a flow or bot owner later requires an admin privilege (`prvChangeOwnerIdOfWorkflow`) that normal makers do not have.
3. PATCH the instructions into the `.gpt.default` component. Adapt anything that references new-harness features.
4. Wire the tools through the classic UI picker (see the registry section above for why). The same flows, no changes to them.
5. Create triggers with the classic wizard, then customize the generated helper flows via deploy: fix the trigger operation, add conditions, replace the default agent prompt with deterministic actions. The wizard output is a starting point, not a finished product. Ours needed the event type corrected and a guard condition added.
6. Publish. Share with the native button.

One design lesson from the migration, harness independent: do not route notifications through the agent. A trigger prompt that says "notify this person" gets recognized as a human-escalation intent and lands in the Escalate system topic, which politely does nothing. Notifications are deterministic. Put a Teams action in the flow and let the agent do what only the agent can do: understand language and call tools.

## The skills

| Skill | Covers |
|---|---|
| `copilot-studio-api-client` | Auth, base calls, PVA actions, publish, the rules that keep you out of trouble |
| `copilot-studio-agent-anatomy` | New-harness shapes: configuration JSON, tool YAML (WorkflowTool, ConnectorTool, McpTool), the 2-piece trigger anatomy, naming conventions |
| `copilot-studio-sharing-visibility` | `modernflowtype`, the access model per table, the full agent-sharing recipe, access masks decoded |
| `copilot-studio-classic-harness` | Classic shapes: gpt.default, .action. tools, AdaptiveDialog topics, harness detection, the migration recipe |

## Setup

1. Drop the skill folders into `.claude/skills/` in your project.
2. Replace the placeholders: `<yourorg>.crm4.dynamics.com`, `<tenantId>`, connection reference names.
3. You need an `az login` session with access to the environment. Tokens are requested per call and never stored.
4. Some recipes reference a companion `power-automate` CLI for flow pull and deploy. Any tool that can GET and PATCH `workflows.clientdata` works the same.

## Honesty section

- Everything here was validated on one tenant, in August 2026, on the harness versions live at that time. Microsoft ships changes to the new experience weekly. Verify before you trust.
- `PvaStartConversation` and `PvaProvision` have confirmed signatures but I have not executed them.
- The internal flow registry is observed behavior, not documented behavior. If you find the table it lives in, open an issue. I looked.

## License

MIT. Use it, break it, tell me what you found.
