# CLAUDE.md — Talkif-ai/skills

Public Agent Skills that teach coding agents to build on the Talkif API. **This repo is public**: the same hygiene as `Talkif-ai/docs` — no internal service names, hosts, infrastructure, PR numbers or people; brand is Talkif / Talkif-ai only.

## What a skill is here

One `skills/<name>/SKILL.md` per job an agent might be asked to do ("place a call", "verify a webhook"). Frontmatter `name` + `description` (the description is what an agent matches on — say *when* to use it). Body order: concepts → exact request (JSON) → TypeScript SDK → Python SDK → curl where useful → gotchas → docs link. 80–150 lines. No dashboard click-instructions: if a feature has no public endpoint (provider setup, Meta Lead Ads, WhatsApp, top-ups, API-key creation) it's a one-line pointer to the docs, not a skill.

## Sources of truth — every fact must come from one of these

| Fact | Source |
|---|---|
| Endpoints, fields, enums, query params | `https://api.talkif.ai/openapi.json` (mirrored as `fern/openapi.json` in `Talkif-ai/docs`) |
| SDK method names | the generated clients: `Talkif-ai/talkif-typescript` `src/api/resources/*/client/Client.ts`; Python is the snake_case of the same names |
| Behaviour, limits, defaults | docs.talkif.ai guides (they are verified against the platform); when the spec text and the guide disagree, the guide wins and the spec defect is filed upstream |
| Browser SDK | `Talkif-ai/webrtc-js` README |

Never write a method name or field from memory. Cross-check every `METHOD /path` against the spec before committing (a scripted check: extract all `` `GET /…` `` strings and assert they exist in `openapi.json`, allowing `/api/v1` and `{param}` differences).

## Recheck rule

Skills drift silently. Recheck all of them when any of these change: the public spec (`openapi.json`), an SDK release, `docs.yml` slugs (every `Docs:` link must resolve), the browser SDK API. Search `skills/` for the changed endpoint/method/slug and update every hit in the same PR; add a changelog line in `Talkif-ai/docs` if the change is customer-visible.

Known "not yet" facts baked into skills (flip when they ship): no client idempotency key on `POST /calls`; call transfer not available; the call list is `GET /calls`, `calls.listCalls` in SDK 0.3.0 (`getCallHistory` / `getActiveCalls` were removed in 0.2.0), and it returns one page, not a pager.

## Conventions

- Skill names are verbs or verb-phrases in kebab-case (`make-call`, `verify-webhooks`); `quickstart` and `api-conventions` are the two exceptions and the two entry points.
- Code samples must run as written with `TALKIF_API_KEY` set; placeholders in `<angle-brackets>`.
- Escape `\$` only where a Markdown renderer could treat `$…$` as math; keep prose plain.
- Commits: `feat: <skill>`, `fix: <skill>: <what>`, `chore:`. No attribution trailers.
- The README table lists every skill; update it when adding one.

## Checks before committing

```sh
grep -rniE "backend-rs|catflow|kamailio|nats|redis|postgres|consul|kubernetes|argo|terraform|pipecat|webhook-executor" skills README.md   # must be empty
```
plus the endpoint cross-check above.
