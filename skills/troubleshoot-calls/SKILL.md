---
name: troubleshoot-calls
description: Diagnose a Talkif call that failed, went wrong, or never happened — read status / agentStatus / failureCode / endReason together, every failureCode and endReason value with its cause and fix, the API error catalogue, and what to check in transcript and metrics. Use when a call didn't behave as expected.
---

# Troubleshoot calls

## Read the three fields together

`GET /calls/{callId}` → `status` (telephony leg), `agentStatus` (AI leg), and when terminal: `failureCode` (technical category, on `failed`) and `endReason` (why it ended, on every terminal call). Then the transcript and metrics.

## `failureCode` → cause → fix

| Code | Cause | Fix |
|---|---|---|
| `insufficient_balance` | available < $1.00 or account suspended | top up; enable auto-recharge |
| `capacity_limit` | queue full (default depth 100) | slow the producer; raise limits with support |
| `rate_limit` | > 60 call placements/min | pace; use campaigns instead of loops |
| `flow_not_published` | no published version, or `flowVersion` doesn't exist | `publishFlow`; check version string |
| `phone_not_active` | from-number suspended/released/pending | fix rental (`billingStatus`) or use another number |
| `invalid_number` | not E.164, or an emergency number | normalise with a default country code |
| `no_connected_flow` | inbound number has no flow | `connectFlow` |
| `unknown_phone_number` | inbound to a number not in the account | check routing at the carrier |
| `flow_not_found` | connected flow was deleted | connect another |
| `config_invalid` | flow config rejected at call time (e.g. retired model) | re-validate and publish |
| `bot_unavailable` / `agent_timeout` / `agent_error` / `agent_disconnected` | the AI leg failed to start or dropped | usually transient; if persistent on one flow, check provider models in `GET /models` |
| `provider_error` / `network_error` / `routing_error` / `service_error` | carrier or vendor side | retry; if it clusters on one number/provider, inspect that provider |
| `busy` / `no_answer` / `rejected` / `canceled` / `timeout` | far end didn't connect | not an error — campaigns retry these |
| `system_error` / `unknown` | platform | quote `requestId`/`callId` to support |

## `endReason` on completed calls

`caller_hangup` (normal), `agent_completed` (agent ended via end_call — the intended path), `agent_timeout` (silence handling hung up — caller went quiet; check idle settings), `agent_error` / `agent_crashed` / `agent_unavailable` (AI leg failed mid-call — see the last transcript turn and `error` events), `telephony_error`, `no_answer` / `busy` / `canceled`, `balance_depleted` (inbound refused at the balance gate), `reconciled` (a stuck call force-ended by the platform — report if frequent).

## Status pairs that mean something

| Seen | Meaning |
|---|---|
| `status: inprogress`, `agentStatus: connecting` for > a few s | line up, agent not speaking: provider warm-up slow; look at `agentConnectedAt` |
| `status: completed`, `agentStatus: errored` | conversation happened then the AI failed; transcript tail + `error` event |
| `status: queued` for long | at concurrency limit; scheduled/campaign calls expire after 10 min / 24 h |
| `202` returned, call never appears | queue entry expired (direct: 60 s) — the account was saturated |

## Conversation problems (call "worked" but badly)

- **Agent talks over the caller / long monologues** → `interruptedTurns`, `wastedTtsCharacters` high → shorten reply guidance in the prompt (`write-prompts`).
- **Slow replies** → time-to-first-response > ~1 s → faster LLM/TTS (`choose-models-and-voices`); check function-call durations (your endpoint latency is silence).
- **Wrong names/dates** → STT model/language; prefer newer conversational STT models; test with real data.
- **Never calls your function** → its `description` doesn't say when; or `onNull: reject` hid it (no contact on this call); or arguments failed schema validation (tighten `description`, e.g. "date as YYYY-MM-DD").
- **Loops on goodbye / never ends** → add an End Call function with an explicit condition; tell the prompt to say goodbye *then* end.
- **Reads `[tags]` aloud** → v3-only delivery tags used on a non-v3 voice.

## API errors

Every error: `{ code, title, status, detail, requestId, errors?[] }`. `GET /errors` returns the full catalogue (code → status, category, description) — use it to build client-side handling rather than string-matching messages. Common: `validation_failed` (see `errors[]` per field), `resource_not_found`, `forbidden` (scope or ownership), `flow_version_conflict` (409 on publish with stale `expectedVersion`).

## Where to look, in order

1. `GET /calls/{callId}` fields above.
2. Transcript (`/transcript`) — the last two turns before the problem.
3. `realtime-events` with `replay: true` on the call — `error`, `status`, `node_transition` frames show the sequence.
4. Cost breakdown — which provider was in use.
5. If a number/provider is common to the failures: its `status`, `billingStatus`, carrier allowlist (SIP).

Docs: https://docs.talkif.ai/getting-started/how-a-call-works · https://docs.talkif.ai/api-reference/errors
