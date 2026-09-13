---
name: make-call
description: Place an outbound phone call with the Talkif API — required fields, the 201-initiated vs 202-queued responses, what is checked before dialing (Do Not Call, legal hours, balance, capacity), call statuses and end reasons, and safe retries. Use whenever code needs to start a call to a person.
---

# Make an outbound call

## Concepts

An outbound call needs four things: a **published flow** (the conversation), a **from number** your account owns (caller ID; its provider routes the call), the **provider ID** that number belongs to, and the **to number** in E.164. Optionally a `contactId` (fills `{{contact.*}}` prompt variables and links the call to the contact's timeline) and free-form `metadata`.

Talkif checks, in order: Do Not Call list → legal calling hours for the recipient's jurisdiction → balance ≥ $1.00 → rate limit (60/min) → concurrency (default 10; beyond it the call **queues**). Then it dials.

## Request

`POST /api/v1/calls`

```json
{
  "flowId": "550e8400-e29b-41d4-a716-446655440000",
  "fromNumber": "+15559876543",
  "providerId": "550e8400-e29b-41d4-a716-446655440001",
  "toNumber": "+15551234567",
  "contactId": "550e8400-e29b-41d4-a716-446655440002",
  "flowVersion": null,
  "metadata": { "orderId": "A-1029" }
}
```

`fromNumber` + `providerId` come from `GET /phone/numbers` (each number record carries its `providerId`). `flowVersion` omitted = latest published version.

```typescript
const result = await client.calls.makeCall({
  flowId, fromNumber: "+15559876543", providerId, toNumber: "+15551234567",
});
if (result.type === "initiated") console.log("dialing", result.callId);
else console.log("queued", result.queueId, "position", result.position);
```

```python
result = client.calls.make_call(flow_id=flow_id, from_number="+15559876543",
                                provider_id=provider_id, to_number="+15551234567")
if result.type == "initiated": print("dialing", result.call_id)
else: print("queued at", result.position)
```

```bash
curl -X POST https://api.talkif.ai/api/v1/calls \
  -H "Authorization: Bearer $TALKIF_API_KEY" -H "Content-Type: application/json" \
  -d '{"flowId":"…","fromNumber":"+15559876543","providerId":"…","toNumber":"+15551234567"}'
```

## Responses

| Status | `type` | Meaning |
|---|---|---|
| `201` | `initiated` | dialing now; `callId`, `status` |
| `202` | `queued` | account at concurrent-call limit; `queueId`, `position`, `estimatedWaitSeconds`. Placed automatically — **treat as success** |

Refusals (4xx, `code` field): `contact_on_dnc_list`, `outside_callable_hours`, `insufficient_balance`, `rate_limit`, `capacity_limit` (queue full), `flow_not_published`, `phone_not_active`, `invalid_number` (emergency numbers are blocked).

## Following the call

`status` moves `queued → initiated → ringing → inprogress → completed | busy | noanswer | canceled | failed`. `agentStatus` tracks the AI leg separately (`pending → connecting → active → completed`). When terminal, `endReason` says why (`caller_hangup`, `agent_completed`, `agent_timeout`, `no_answer`, …) and `failureCode` categorises failures.

Poll `GET /calls/{callId}` sparingly, or subscribe to the WebSocket stream — see the `realtime-events` skill. Results (transcript, recording, cost, insights): `read-call-results` skill.

## Safe retries

There is no idempotency key on call creation. After a **timeout** (not a 4xx), check before re-issuing: `GET /calls/history?contactId=…&startDate=<a few minutes ago>` (or `phoneNumber=`). A call created since your request → done; nothing → retry. Set the SDK's `maxRetries: 0` for `makeCall` so it doesn't auto-retry a possibly-succeeded POST.

For a list of people, use the `create-campaign` skill instead of a loop — campaigns pace, retry and skip DNC for you.

Docs: https://docs.talkif.ai/calls/make-a-call · lifecycle: https://docs.talkif.ai/getting-started/how-a-call-works
