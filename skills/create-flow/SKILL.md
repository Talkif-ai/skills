---
name: create-flow
description: Build a Talkif flow programmatically — the FlowDefinition shape (main agent, sub-agents, transitions, functions, STT/LLM/TTS services), creating from a template, validation rules, publishing, versions, rollback, and which version a call runs. Use when code must create or update a conversation agent rather than the visual builder.
---

# Create and publish a flow

## Concepts

A **flow** is a graph of agents. Exactly one `main_agent` is the entry point and owns the call-wide **`systemPrompt`** (persona, tone, hard limits). Every agent — main included — has its own **`agentPrompt`** (its job). Agents hand off through **transitions** (an edge plus a description the model reads); each may call **functions** (`end_call` built-in, or your HTTP endpoints). Three **services** — `stt`, `llm`, `tts` — are required to publish.

A flow has a **draft** (editable) and immutable **published versions** (`1.0.0`, `1.0.1`…). Only published versions take calls. Phone numbers follow *latest* or are pinned to a version.

## Fastest path: start from a template

```typescript
const flow = await client.flowTemplates.instantiateTemplate("customer-support", { name: "Support line" });
```
`GET /flow-templates` lists slugs: `appointment-reminder`, `customer-support`, `customer-survey`, `lead-qualification`, `payment-reminder`. Then edit prompts with `updateFlow` and publish.

## Full definition

`POST /api/v1/flows` — `{ name, description?, definition, recordingOverride?, analysisOverride?, analysisInstructions?, userIdle?, maxCallDuration?, voicemailEnabled?, … }`

```json
{
  "name": "Dental booking line",
  "definition": {
    "initialNode": "main_agent",
    "services": {
      "stt": { "provider": "deepgram", "model": "nova-3-general", "language": "en" },
      "llm": { "provider": "openai", "model": "gpt-4o-mini", "temperature": 0.5, "maxTokens": 300 },
      "tts": { "provider": "elevenlabs", "model": "eleven_flash_v2_5", "voiceId": "<voice-id>" }
    },
    "nodes": {
      "main_agent": {
        "type": "main_agent",
        "name": "Reception",
        "systemPrompt": "You are Mia, the phone assistant for Northwind Dental. This is a live call: one or two short sentences per reply. Never give medical advice.",
        "agentPrompt": "Greet the caller and find out whether they want to book, move or cancel an appointment. When they do, hand off to the booking agent.",
        "transitions": [
          { "targetAgentId": "booking_agent", "keywords": "The caller wants to book, move or cancel an appointment." }
        ],
        "functions": [
          { "type": "builtin", "name": "end_call", "description": "The caller has no further questions and has said goodbye." }
        ]
      },
      "booking_agent": {
        "type": "agent",
        "name": "Booking",
        "agentPrompt": "Collect, one at a time: full name, date of birth, preferred day and time. Read them back and get a yes. Then thank them, say goodbye, and end the call.",
        "functions": [
          { "type": "http_request", "name": "check_availability", "description": "Check whether a date and time is free. Call before offering any slot.", "config": { "functionId": "<flow-function-uuid>" } },
          { "type": "builtin", "name": "end_call", "description": "The booking is confirmed and the caller has said goodbye." }
        ]
      }
    }
  }
}
```

Rules the validator enforces at publish: one `main_agent` and it is `initialNode`; `systemPrompt` on the main agent **only**; `agentPrompt` on **every** agent; transition targets exist; agent and function names unique; `stt`/`llm`/`tts` present with models from the current catalogue (`GET /models`); builtin functions are `end_call` only; `http_request` functions reference an existing flow function (see `create-flow-function`). Settings a model doesn't support (e.g. temperature on a reasoning model) are stripped, not rejected.

`preActions` / `postActions` support `{ "type": "tts_say", "config": { "text": "…" } }` for a fixed spoken line on entering/leaving an agent.

## Publish, versions, rollback

```typescript
await client.flows.validateFlow(flowId);                       // POST /flows/{id}/validate — errors without publishing
const pub = await client.flows.publishFlow(flowId, { notes: "shorter greeting" });  // → new version, e.g. 1.0.4
await client.flows.rollbackFlow(flowId, "1.0.3");              // republishes 1.0.3's content as a NEW version
const v = await client.flows.getFlowVersion(flowId, "1.0.3");  // exact definition of any version
```

- `publishFlow` accepts `version` (explicit `X.Y.Z`) and `expectedVersion` (optimistic lock → `409` if someone published meanwhile).
- Editing the draft never affects live calls; only publish does.
- Numbers on *latest* serve the new version immediately; numbers pinned to the replaced version are moved forward; other pins are untouched. `POST /calls` without `flowVersion` runs latest.
- Rollback never rewinds history — it creates a newer version with older content.

## Call-level settings (per flow)

`userIdle` (silence handling; on by default: timeout 10 s, 2 prompts), `maxCallDuration` (off by default; 30–14400 s), `voicemailEnabled` + `voicemailMessage` / `voicemailMessagePrompt` (outbound, Twilio-routed only), `recordingOverride` / `analysisOverride` (`use_account_default` | `enabled` | `disabled`), `analysisInstructions`.

Docs: https://docs.talkif.ai/concepts/flows-and-versions · https://docs.talkif.ai/build/design-a-conversation · https://docs.talkif.ai/build/test-and-publish
