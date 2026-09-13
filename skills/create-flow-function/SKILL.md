---
name: create-flow-function
description: Let a Talkif agent call your backend mid-conversation — define an HTTP flow function (method, URL, path/query/body JSON Schema), bind parameters to the model, static values or call context, set headers and timeout, attach it to an agent, and handle what your endpoint receives. Use when the agent must look something up or take an action in an external system.
---

# Create a flow function (HTTP tool)

## Concepts

A flow function is an HTTP request the **model may call** while on an agent. Talkif fills the arguments (from the conversation, from fixed values, or from call context you bind), signs the request, sends it, and feeds the response back to the model before its next reply. The caller is waiting the whole time — aim for well under a second; hard timeout 30 s.

Functions are account-level (`/flow-functions`) and attached to agents by reference (`{ type: "http_request", config: { functionId } }`).

## Define it

`POST /api/v1/flow-functions`

```json
{
  "name": "check_availability",
  "description": "Check whether a specific date and time is free for a hygiene appointment. Call before offering any slot to the caller.",
  "request": {
    "method": "POST",
    "url": "https://api.example.com/clinics/{clinicId}/availability",
    "pathParams":  { "type": "object", "properties": { "clinicId": { "type": "string" } }, "required": ["clinicId"] },
    "queryParams": { "type": "object", "properties": { "source": { "type": "string", "enum": ["phone", "web"] } } },
    "body": {
      "type": "object",
      "properties": {
        "date": { "type": "string", "format": "date", "description": "YYYY-MM-DD" },
        "time": { "type": "string", "pattern": "^\\d{2}:\\d{2}$", "description": "HH:MM, 24h" }
      },
      "required": ["date", "time"]
    }
  },
  "paramBindings": {
    "clinicId": { "source": "static", "value": "north-seattle" },
    "source":   { "source": "static", "value": "phone" }
  },
  "webhookHeaders": { "Authorization": "Bearer <your-backend-token>" },
  "timeoutMs": 5000
}
```

```typescript
const fn = await client.flowFunctions.createFlowFunction({ name, description, request, paramBindings, webhookHeaders, timeoutMs });
```
```python
fn = client.flow_functions.create_flow_function(name=..., description=..., request=..., param_bindings=..., webhook_headers=..., timeout_ms=5000)
```

Rules: parameter names unique across the three locations; every URL `{placeholder}` has a `pathParams` property and vice versa; path/query are primitives only; body allows nested objects/arrays to depth 5; `format`/`pattern`/`min`/`max` are enforced on the model's arguments before sending; `timeoutMs` 100–30000; `webhookHeaders` is encrypted, write-only (`hasWebhookHeaders: true` on read), `null` clears.

## Bindings

| `source` | Effect |
|---|---|
| `llm` (default) | in the model's tool schema; model fills it from conversation |
| `static` `{ value }` | fixed; hidden from the model |
| `call_context` `{ contextKey, onNull }` | from the live call; hidden. Keys: `caller.phone_number`, `caller.contact_id`, `caller.contact_name`, `caller.contact_first_name`, `caller.contact_last_name`, `caller.contact_email`, `call.id`, `call.direction`, `call.started_at`, `flow.id`, `flow.name`, `account.id` |

`onNull`: `reject` = the function is hidden from the model on calls where the value is empty (no matching contact); `fallback_to_llm` = the model asks for it instead. Use `reject` for identity-critical parameters.

## Attach to an agent

In the flow definition (`create-flow` skill):
```json
{ "type": "http_request", "name": "check_availability", "description": "…", "config": { "functionId": "<uuid>" }, "transitionTo": "confirm_agent" }
```
`transitionTo` moves the conversation after the call **whether or not it succeeded** — the model receives the error as the tool result and carries it into the next agent. Publishing snapshots the function's schema/bindings into the version; headers and timeout are read live.

## What your endpoint receives and must do

- Method/URL/body as declared, arguments merged in; `Content-Type: application/json` when there's a body; your `webhookHeaders`; `Talkif-Signature` once a signing secret exists — **verify it** (`verify-webhooks` skill).
- Reply with `2xx`–`3xx` and a small JSON body shaped for reading aloud: `{"available": false, "alternatives": ["14:30", "16:00"]}`. Non-JSON is passed as `{"raw": "…"}`.
- Limits: request ≤ 1 MB, response read ≤ 1 MB, no redirects, public addresses only (private/metadata ranges refused).
- On failure the model sees the error; tell the prompt what to do ("apologise and offer a callback").

Docs: https://docs.talkif.ai/build/call-your-backend
