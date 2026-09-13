---
name: setup-api-key
description: Authenticate against the Talkif API — create an API key with the right scopes, configure the base URL and Bearer header, initialise the TypeScript or Python SDK client, and keep keys out of browsers. Use first, before any other Talkif skill.
---

# Set up a Talkif API key

## Concepts

- Every request acts for **one account**. An API key is bound to that account; you never send an account ID.
- Key format `tif_live_…`, sent as `Authorization: Bearer <key>`. Base URL `https://api.talkif.ai/api/v1`.
- Keys carry **scopes** (`calls:*`, `contacts:read`, `campaigns:write`, `*`…), optionally an **IP allowlist** and an **expiry**. Up to 25 keys per account.
- The full key is shown **once** at creation. Store it in a secret manager; put it in `TALKIF_API_KEY`.

## Create a key

Dashboard: **Developer → Credentials → Create API Key** — name it for the system that will hold it, choose scopes, copy the key. (Key creation is dashboard-only; there is no API endpoint for it.)

Scopes to request per job:

| Job | Scopes |
|---|---|
| Place calls, read results | `calls:*` |
| Load contacts, run campaigns | `contacts:*`, `campaigns:*`, `dnc:read` |
| Manage flows programmatically | `flows:*`, `flow_functions:*` |
| Finance dashboard | `billing:read`, `analytics:read` |
| Everything (avoid in production) | `*` |

## Initialise a client

```typescript
import { TalkifClient } from "@talkif/sdk";           // npm install @talkif/sdk
const client = new TalkifClient({ token: process.env.TALKIF_API_KEY });
```

```python
import os
from talkif import Talkif                              # pip install talkif
client = Talkif(token=os.environ["TALKIF_API_KEY"])   # AsyncTalkif for asyncio
```

```bash
curl https://api.talkif.ai/api/v1/calls \
  -H "Authorization: Bearer $TALKIF_API_KEY"
```

`token` also accepts a function, so a rotated key is picked up without rebuilding the client.

## Verify it works

`GET /calls` returns `200 { "calls": [], "meta": … }` on a fresh account (SDK: `calls.getActiveCalls()` / `get_active_calls()`). `401` = missing/invalid key; `403` with a scope error code = the key lacks that scope.

## Rules

- **Never ship an API key to a browser or mobile app.** For voice calls from a web page use a publishable key with the browser SDK — see the `web-calls` skill.
- Rotate by creating a new key, deploying it, confirming usage moved, then revoking the old one. Revocation is immediate and permanent.
- All API errors share one shape: `{ "code", "title", "status", "detail", "requestId" }`. Branch on `code`; quote `requestId` to support. Catalogue: https://docs.talkif.ai/api-reference/errors

Docs: https://docs.talkif.ai/integrate/authentication · SDKs: https://docs.talkif.ai/integrate/sdks
