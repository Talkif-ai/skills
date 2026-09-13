---
name: api-conventions
description: The rules shared by every Talkif API endpoint — base URL and auth, JSON casing, pagination and the SDK pagers, the error envelope and requestId, rate limits and 429 handling, retries and the no-idempotency-key rule, timestamps and phone formats, and the public spec/SDK sources. Load alongside any other Talkif skill when writing client code.
---

# Talkif API conventions

- **Base URL** `https://api.talkif.ai/api/v1`. **Auth** `Authorization: Bearer tif_live_…`; the key implies the account (no account header). `setup-api-key` for scopes.
- **JSON casing**: camelCase in bodies and responses (`fromNumber`, `endReason`); a few list-endpoint query params are snake_case (`/billing/charges?charge_type=`). Follow the reference per endpoint.
- **IDs** are UUIDs. **Phone numbers** are E.164 (`+15551234567`) everywhere. **Timestamps** are RFC 3339 UTC; wall-clock times (`scheduleTime`, campaign windows) are `HH:MM`.
- **Pagination**: `limit` + `offset` (starts at **0**); responses carry the items under a resource key (`calls`, `contacts`, `campaigns`, …) plus `meta` with totals. SDK list methods return a **pager** — iterate (`for await` / `for`) and it fetches pages. TTS voices use `page`/`pageSize`/`nextPageToken`.
- **Success codes**: `200`; `201` on create; **`202` on `POST /calls` means queued — success**.
- **Error envelope** (every endpoint): `{ "code", "title", "status", "detail", "requestId", "errors"?: [{ field, code, message }] }`. Branch on `code`; log `requestId`; show `detail` to humans. `GET /errors` is the machine-readable catalogue. SDKs raise typed errors per status (`Talkif.PaymentRequiredError`, `TooManyRequestsError`, …; Python `talkif.errors.*`) with `statusCode` and `body`.
- **Rate limits**: 60 call placements/min per account (`rate_limit`); auth endpoints are burst-limited per IP; assistant 10/min. On `429`: exponential backoff with jitter; never a hot loop.
- **Retries**: SDKs retry `408/429/5xx` twice with backoff. Safe for everything except **`POST /calls`** (no idempotency key) — set `maxRetries: 0` there and check `GET /calls?contactId=&startDate=` before re-issuing after a timeout (`make-call`).
- **Concurrency**: default 10 calls in flight per account; beyond it outbound calls queue (depth 100) — design for `202`.
- **Versioning**: paths are `/api/v1`; additive changes ship without a bump; the public OpenAPI is the contract: `https://api.talkif.ai/openapi.json`. SDKs: `@talkif/sdk` (npm), `talkif` (PyPI), generated from that spec — minor for additions, major for removals.
- **Money**: USD decimal strings (`"12.50"`) in balances and costs; microcents in some breakdown fields — read the unit on the field.
- **Public repo hygiene for anything you generate**: never embed an API key client-side; use publishable keys + the browser SDK for web (`web-calls`).

Docs: https://docs.talkif.ai/integrate/authentication · https://docs.talkif.ai/integrate/idempotency-and-rate-limits · https://docs.talkif.ai/api-reference
