---
name: update-flow
description: Change an existing Talkif flow safely from code — get → modify → PUT /flows/{id} → validate → publish, full-definition replacement semantics, editing prompts/transitions/functions/services, per-flow settings, tags, versions, optimistic locking and rollback. Use for any programmatic edit of a flow that is already live.
---

# Update a flow

## Semantics you must know

- `PUT /flows/{flowId}` replaces the fields you send. **`definition` is all-or-nothing**: send the whole `FlowDefinition` (`initialNode`, `services`, `nodes`), not a patch — omitting a node deletes it. Read first, modify, write back.
- Saving edits the **draft**. Live calls keep running the last published version until you `publishFlow`.
- `userIdle` and `maxCallDuration` blocks, when sent, **fully replace** that block; omit them to leave them unchanged.
- Publishing snapshots flow-function schemas into the version; editing a function later needs a republish to take effect on calls.

## The loop

```typescript
const flow = await client.flows.getFlow(flowId);                       // draft definition + settings
const def = structuredClone(flow.definition);
def.nodes.main_agent.agentPrompt = "Greet the caller, then …";
def.nodes.booking_agent.transitions.push({ targetAgentId: "main_agent", keywords: "The caller asks about anything other than booking." });
def.services.llm = { provider: "openai", model: "gpt-4o-mini", temperature: 0.4 };

await client.flows.updateFlow(flowId, { definition: def, tags: ["support", "v2"] });
await client.flows.validateFlow(flowId);   // 200 { warnings[] } when valid; throws a 400 `validation_failed` error whose body.errors[] lists { field, code, message }
await client.flows.publishFlow(flowId, { notes: "return transition from booking; gpt-4o-mini", expectedVersion: flow.version });
```
```python
flow = client.flows.get_flow(flow_id)
d = flow.definition; d.nodes["main_agent"].agent_prompt = "Greet the caller, then …"
client.flows.update_flow(flow_id, definition=d)
client.flows.validate_flow(flow_id)   # raises BadRequestError with errors[] when invalid
client.flows.publish_flow(flow_id, notes="…", expected_version=flow.version)
```
`expectedVersion` makes publish fail with `409 flow_version_conflict` if someone else published in between — re-read and retry rather than overwrite.

## Common edits

| Change | Where |
|---|---|
| Prompt wording | `nodes.<id>.systemPrompt` (main only) / `agentPrompt` |
| Add an agent | new key in `nodes` with `type: "agent"`, `name`, `agentPrompt`; add a transition to it and a return path |
| Attach a backend call | `nodes.<id>.functions[] += { type: "http_request", name, description, config: { functionId } }` |
| Swap a model/voice | `services.llm` / `.stt` / `.tts` — validate: retired models fail `invalid_model` |
| Silence / max length / voicemail | top-level `userIdle`, `maxCallDuration`, `voicemailEnabled` + message fields |
| Recording / analysis | `recordingOverride`, `analysisOverride`, `analysisInstructions` |
| Organise | `name`, `description`, `tags` (≤ 10) |

Validation error codes worth handling: `required`, `missing_main_agent`, `multiple_main_agents`, `entry_agent_only` (systemPrompt on a sub-agent), `invalid_reference` / `self_reference` (transitions), `duplicate_agent_name`, `duplicate_function_name`, `invalid_provider`, `invalid_model`, `out_of_range`, `unsupported_parameter`, `unknown_builtin`.

## Versions

`GET /flows/{id}/versions/{version}` for any past definition; `POST /flows/{id}/rollback/{version}` republishes it as a **new** version (history never rewinds). Numbers on *latest* follow immediately; numbers pinned to the version you replaced are moved forward. Compare cost/outcomes per version with `GET /flows/{id}/stats` before and after.

## Delete

`DELETE /flows/{flowId}` — refused (`400`) while any phone number is connected to it; disconnect or reconnect them first. Calls that referenced it keep their history.

Docs: https://docs.talkif.ai/concepts/flows-and-versions · https://docs.talkif.ai/build/test-and-publish
