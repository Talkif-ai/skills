# Talkif Skills

[Agent Skills](https://agentskills.io) for building AI phone agents on [Talkif](https://talkif.ai) with Claude Code, Codex, Cursor and any agent that reads `SKILL.md`. Each skill teaches the agent one job — the concepts, the exact API calls, and the mistakes to avoid — with examples in the TypeScript SDK, the Python SDK and plain REST.

| Skill | Teaches the agent to |
|---|---|
| [`quickstart`](skills/quickstart) | **start here** — sequence key → flow → number → connect → test → results |
| [`api-conventions`](skills/api-conventions) | base URL, casing, pagination, errors, rate limits, retries — load with any other skill |
| [`setup-api-key`](skills/setup-api-key) | authenticate: API keys, scopes, base URL, SDK clients |
| [`create-flow`](skills/create-flow) | build a flow definition (agents, transitions, providers), publish it, manage versions |
| [`write-prompts`](skills/write-prompts) | write System and Agent Prompts that work on a phone line, with `{{variables}}` |
| [`create-flow-function`](skills/create-flow-function) | let the agent call your backend mid-call (HTTP functions, bindings, signing) |
| [`make-call`](skills/make-call) | place an outbound call and handle `201` vs `202`, statuses, end reasons |
| [`create-campaign`](skills/create-campaign) | call a contact list with windows, timezones, retries, Do Not Call |
| [`manage-phone-numbers`](skills/manage-phone-numbers) | buy a number, connect a flow, pin a version |
| [`verify-webhooks`](skills/verify-webhooks) | verify the `Talkif-Signature` header on requests from Talkif |
| [`realtime-events`](skills/realtime-events) | stream call events over WebSocket, with replay |
| [`web-calls`](skills/web-calls) | put a voice agent on a web page with the browser SDK |
| [`read-call-results`](skills/read-call-results) | transcripts, recordings, cost breakdowns, AI insights |
| [`create-schedule`](skills/create-schedule) | call one contact on a recurrence, in their timezone, with retries |
| [`manage-contacts`](skills/manage-contacts) | CRM sync: create, update, search by phone, tag, timeline, import/export |
| [`do-not-call`](skills/do-not-call) | suppression list: add, bulk import, check, wire unsubscribes |
| [`read-billing`](skills/read-billing) | balance and buckets, charges, ledger, cost totals, invoices |
| [`handle-inbound-calls`](skills/handle-inbound-calls) | make a number answer: flow connection, unknown callers, busy reasons, inbound prompts |
| [`update-flow`](skills/update-flow) | get → modify → PUT → validate → publish; full-definition semantics; versions |
| [`test-a-flow`](skills/test-a-flow) | draft tests via the browser SDK, real test calls, the failure paths to exercise |
| [`choose-models-and-voices`](skills/choose-models-and-voices) | pick STT/LLM/TTS from the live catalogue; latency, quality, cost |
| [`troubleshoot-calls`](skills/troubleshoot-calls) | every failureCode and endReason → cause → fix; the error catalogue |

## Install

**Any agent (universal)**

```sh
npx skills add Talkif-ai/skills
# one skill only
npx skills add Talkif-ai/skills --skill make-call
```

**Claude Code**

```sh
/plugin marketplace add Talkif-ai/skills
/plugin install talkif-voice-agents@talkif-skills
```

**Manual** — copy a `skills/<name>/` folder into `.claude/skills/` (or your agent's skills directory).

## Conventions the skills assume

- Base URL `https://api.talkif.ai/api/v1`; header `Authorization: Bearer tif_live_…`. A key is scoped to one account — no account ID in requests.
- SDKs: [`@talkif/sdk`](https://www.npmjs.com/package/@talkif/sdk) (Node 18+) and [`talkif`](https://pypi.org/project/talkif/) (Python 3.10+). Browser: [`@talkif/webrtc`](https://www.npmjs.com/package/@talkif/webrtc).
- Every endpoint the skills mention is documented in the [API Reference](https://docs.talkif.ai/api-reference); the guides are at [docs.talkif.ai](https://docs.talkif.ai).

Skills track the public API. When an endpoint or SDK method changes, the affected skill is updated in the same release — file an issue if you find one that drifted.

## License

MIT
