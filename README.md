# Talkif Skills

[Agent Skills](https://agentskills.io) that teach AI coding agents — Claude Code, Codex, Cursor, and any tool that reads `SKILL.md` — how to build AI phone agents on [Talkif](https://talkif.ai).

Each skill covers one job: the concepts, the exact API requests, the same code in the TypeScript SDK, the Python SDK and REST, and the mistakes to avoid. Every endpoint, field and enum is verified against the public API definition.

## Install

**Any agent**

```sh
npx skills add Talkif-ai/skills            # all skills
npx skills add Talkif-ai/skills --skill make-call
```

**Claude Code**

```text
/plugin marketplace add Talkif-ai/skills
/plugin install talkif-voice-agents@talkif-skills
```

**Manual** — copy `skills/<name>/` into your agent's skills directory (`.claude/skills/` for Claude Code).

## Then just ask

> Set up Talkif, create a support flow from a template, connect it to my Talkif number and call my phone.

> Add a flow function that checks appointment availability on my API, and attach it to the booking agent.

> Write a webhook endpoint in Express that verifies the Talkif signature.

> Import this CSV, scrub it against Do Not Call, and start a campaign 9–5 in each contact's timezone.

> Stream every finished call into my CRM.

The agent loads `quickstart` for the sequence and then the skill for each step.

## What you need

- A Talkif account — [app.talkif.ai](https://app.talkif.ai) — with an API key (`Developer → Credentials`) in `TALKIF_API_KEY`.
- Node 18+ for `@talkif/sdk`, Python 3.10+ for `talkif`. Skills also show plain REST.
- A phone number, bought in the dashboard by the account Owner (API keys can't buy numbers). Until the account's first payment, the welcome credit can't pay for numbers; it covers building and browser test calls.

## Skills

**Start**

| Skill | Teaches the agent to |
|---|---|
| [`quickstart`](skills/quickstart) | sequence key → flow → number → connect → test → results — **load first** |
| [`api-conventions`](skills/api-conventions) | base URL, auth, casing, pagination, error envelope, rate limits, retries — load with any other skill |
| [`setup-api-key`](skills/setup-api-key) | API keys, scopes, IP allowlists, SDK clients |

**Build the agent**

| Skill | Teaches the agent to |
|---|---|
| [`create-flow`](skills/create-flow) | write a flow definition (main agent, sub-agents, transitions, functions, STT/LLM/TTS), publish, versions, rollback |
| [`update-flow`](skills/update-flow) | get → modify → PUT → validate → publish; full-definition semantics; optimistic locking |
| [`write-prompts`](skills/write-prompts) | System Prompt vs Agent Prompt, phone-call constraints, `{{variables}}` and helpers, ElevenLabs v3 delivery tags |
| [`create-flow-function`](skills/create-flow-function) | HTTP tools the agent calls mid-conversation: schema, bindings, context keys, signing, limits |
| [`choose-models-and-voices`](skills/choose-models-and-voices) | pick STT / LLM / TTS from the live catalogue; latency vs quality vs cost |
| [`test-a-flow`](skills/test-a-flow) | draft tests with the browser SDK, real test calls, the failure paths to exercise |

**Run calls**

| Skill | Teaches the agent to |
|---|---|
| [`make-call`](skills/make-call) | place an outbound call; `201` vs `202` queued; admission checks; safe retries |
| [`handle-inbound-calls`](skills/handle-inbound-calls) | make a number answer: flow connection, unknown callers, why a call is busy |
| [`create-campaign`](skills/create-campaign) | call a list: import, DNC scrub, concurrency, contact-local windows, lifecycle, analytics |
| [`create-schedule`](skills/create-schedule) | call one contact on a recurrence, in their timezone, with retries |
| [`manage-phone-numbers`](skills/manage-phone-numbers) | find a number to buy, connect a flow, pin a version, release; bring-your-own-carrier facts |
| [`manage-contacts`](skills/manage-contacts) | CRM sync: create, update, search by phone, tags, timeline, import/export |
| [`do-not-call`](skills/do-not-call) | suppression list: add, bulk import, check, wire unsubscribes |

**Read results and integrate**

| Skill | Teaches the agent to |
|---|---|
| [`read-call-results`](skills/read-call-results) | transcript, recording, cost breakdown, metrics, AI insights, history filters |
| [`read-billing`](skills/read-billing) | balance buckets, the admission gate, charges, ledger, cost totals, invoices |
| [`troubleshoot-calls`](skills/troubleshoot-calls) | every `failureCode` and `endReason` → cause → fix; the error catalogue |
| [`verify-webhooks`](skills/verify-webhooks) | verify `Talkif-Signature` (HMAC-SHA256), rotation, empty bodies — Node and Python |
| [`realtime-events`](skills/realtime-events) | WebSocket event stream: frames, event types, replay, limits |
| [`web-calls`](skills/web-calls) | a voice agent on a web page with the browser SDK and a publishable key |

## Conventions the skills assume

- Base URL `https://api.talkif.ai/api/v1`; `Authorization: Bearer tif_live_…`. A key is scoped to one account.
- SDKs: [`@talkif/sdk`](https://www.npmjs.com/package/@talkif/sdk) · [`talkif`](https://pypi.org/project/talkif/) · browser [`@talkif/webrtc`](https://www.npmjs.com/package/@talkif/webrtc).
- Guides: [docs.talkif.ai](https://docs.talkif.ai) · API reference: [docs.talkif.ai/api-reference](https://docs.talkif.ai/api-reference) · spec: `https://api.talkif.ai/openapi.json`.

## Keeping up to date

Skills track the public API and SDKs. When an endpoint or SDK method changes, the affected skill is updated in the same release and noted in the [changelog](https://docs.talkif.ai/changelog). Re-run the install command to refresh. If you find a skill that disagrees with the API, [open an issue](https://github.com/Talkif-ai/skills/issues) with the skill name and the request that failed.

## Contributing

PRs welcome for corrections and clearer examples. Every fact must trace to the public spec or the docs — see [`CLAUDE.md`](CLAUDE.md) for the sources of truth and the pre-commit checks. Keep the format: concepts → request → TypeScript → Python → gotchas → docs link.

## License

MIT — see [LICENSE](LICENSE).
