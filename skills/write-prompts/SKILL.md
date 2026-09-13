---
name: write-prompts
description: Write and audit prompts for Talkif voice agents — what belongs in the System Prompt versus each Agent Prompt, phone-call constraints, the {{contact/call/account/system}} variables and Handlebars helpers resolved at call time, and the patterns that fail on a phone line. Use when creating or reviewing agent instructions.
---

# Write prompts for a phone agent

## Two prompts, two jobs

| | Where | Scope | Contains |
|---|---|---|---|
| **System Prompt** | `main_agent` only | whole call, survives transitions | persona, tone, hard limits — a few lines; it is resent on every turn |
| **Agent Prompt** | every agent | while the conversation is on that agent | goal, what to collect, when to call a function, when to move on or end |

Publishing fails if `systemPrompt` appears on a non-main agent or an agent lacks `agentPrompt`.

## What works on a phone line

```text title=System Prompt
You are Mia, the phone assistant for Northwind Dental in Seattle.
This is a live phone call: keep replies to one or two short sentences and never read lists aloud.
You are not a clinician — never give medical advice; offer to take a message instead.
If the caller is upset, stay calm and offer a human callback.
```

```text title=Agent Prompt (booking step)
Goal: book, move or cancel a hygiene appointment.
Collect, one at a time: full name, date of birth, preferred day and time.
Read the details back and get an explicit yes before calling book_appointment.
If the slot is unavailable, offer the two nearest alternatives from the function result.
When the booking is confirmed, thank them, say goodbye, and then end the call.
```

Do: state the goal first · name every datum, **one at a time** · say when to call each function and what to do with its result · say when to end (the built-in `end_call` is silent — the goodbye must come first) · put hard limits in the System Prompt.

Don't: write scripts ("Say: 'Hello, thank you for…'") — they sound robotic and break on interruption · use markdown formatting — it's read aloud · include what this step doesn't need — every token is paid on every turn · contradict the System Prompt.

Split work across agents: triage → specialists, or open → do → close. Each agent's prompt describes one job; transitions are the checkpoints.

## Emotion tags (ElevenLabs v3 only)

If the flow's TTS is `eleven_v3` / `eleven_v3_conversational`, the voice performs inline tags like `[warmly]`, `[laughs]`, `[whispers]`, `[hesitates]`. Then, and only then, add to the System Prompt: *"Use at most one bracketed delivery tag per reply, only where it changes meaning; never tag routine sentences."* On any other voice tags are read aloud as text. Tags also appear verbatim in transcripts, and v3 honours only the *stability* voice control. ElevenLabs guide: https://elevenlabs.io/docs/best-practices/prompting/eleven-v3

## Dynamic prompts

Both prompts are templates resolved **when the call starts**:

```handlebars
You are calling {{default contact.firstName "the customer"}}
{{#ifExists contact.company}}from {{contact.company}}{{/ifExists}}.
Today is {{formatDate system.date "long"}}.
{{#if (eq call.direction "inbound")}}Thank them for calling.{{else}}Confirm it's a good time to talk.{{/if}}
{{#ifExists contact.notes}}Background: {{contact.notes}}{{/ifExists}}
```

| Namespace | Fields | Populated |
|---|---|---|
| `contact.*` | `id name firstName lastName primaryPhone email company occupation notes timezone language interests tags address.street .city .state .postalCode .country` | when the call has a contact (`contactId`, or a matching number); else only `primaryPhone` |
| `call.*` | `id direction fromNumber toNumber` | always |
| `account.*` | `id name` | always |
| `system.*` | `date` (YYYY-MM-DD) `time` (HH:MM:SS) `timezone timestamp` | always |

Helpers: `default`, `#ifExists…{{else}}`, `#if` with `eq ne gt lt gte lte`, `and or not`, `includes` (array/string contains), `join`, `formatDate` (`short` MM/DD/YYYY · `long` Month DD, YYYY · `time` · `datetime`), `match`/`replace` (regex), `json`. Missing values render **empty** — always guard contact fields with `default` or `#ifExists`.

## Audit checklist

- [ ] System Prompt ≤ ~8 lines; identity, tone, limits only
- [ ] Each Agent Prompt: goal in line 1; data to collect listed; function use stated; end condition stated
- [ ] No scripted lines, no markdown, no duplicated policy across agents
- [ ] Every `{{contact.*}}` guarded; camelCase field names (`firstName`, not `first_name`)
- [ ] Tested the failure paths in the Flow Tester: interruption, wrong answer, off-topic, silence, goodbye

Docs: https://docs.talkif.ai/build/write-prompts
