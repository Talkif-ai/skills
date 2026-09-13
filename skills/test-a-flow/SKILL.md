---
name: test-a-flow
description: Test a Talkif flow before and after publishing — a real outbound test call to your own phone, the authenticated browser SDK against a draft definition, what to say to exercise failure paths, which metrics to read, and comparing versions. Use before any publish that reaches customers.
---

# Test a flow

## Three ways

| Method | Runs | Needs | Use for |
|---|---|---|---|
| **Dashboard Flow Tester** | the open draft, unsaved edits included | a browser + mic, ≥ $1.00 balance | fastest iteration on prompts; shows per-turn latency and tokens |
| **Browser SDK, authenticated mode** | any published flow, or a **draft definition** you pass | `@talkif/webrtc` + `accountId` + `auth` provider | automated conversational tests from your own tooling |
| **Real outbound call** | the published version | a from-number, `calls.makeCall` | the carrier leg, caller identity, contact variables, voicemail — the browser can't test these |

Test calls are billed like real calls and pass the same $1.00 admission gate. There is no "free test mode".

## Draft test from code (authenticated browser SDK)

```ts
import { TalkifCall } from "@talkif/webrtc";
const call = new TalkifCall({ baseUrl: "https://api.talkif.ai", accountId, auth: async () => `Bearer ${await getToken()}` });
call.on("transcript", ({ role, content }) => log(role, content));
call.on("ended", ({ reason }) => done(reason));
await call.start({ draftDefinition: definition });   // or { flowId } for a published flow
```
Runs in a browser context (WebRTC + microphone). The draft is validated the same way publish validates it, so an invalid draft fails here first.

## Real call from code

```typescript
const r = await client.calls.makeCall({ flowId, fromNumber, providerId, toNumber: "+1YOURPHONE", metadata: { test: true } });
```
Then `getCallDetails`, `getCallTranscript`, `getBillingCallCostBreakdown`, or subscribe via `realtime-events` with `replay: true`.

## What to exercise (the paths that fail in production)

1. Interrupt the agent mid-sentence — does it stop and recover?
2. Give a wrong or ambiguous answer ("next Tuesday-ish") — does it re-ask *one* thing?
3. Ask something off-topic — does it decline gracefully and return?
4. Go silent 15 s — does silence handling nudge, then end?
5. Trigger each function with bad data — is the error handled per the prompt?
6. Say goodbye — does the call end within one turn, with a farewell first?
7. Outbound only: let it hit voicemail (if detection is on).

## Read

`interruptedTurns`, `wastedTtsCharacters` (agent talks too long), time-to-first-response (> ~1 s → faster model), function-call durations (your endpoint), the transcript's last turns, `endReason` (`agent_completed` is the goal). Compare per version with `GET /flows/{id}/stats`.

## Before publishing to customers

Validate (`validateFlow`), run the seven paths above on the draft, publish with notes, then make **one real call** to your own phone on the published version. Keep the previous version's number: rollback is one call (`rollbackFlow`).

Docs: https://docs.talkif.ai/build/test-and-publish
