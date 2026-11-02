---
name: read-call-results
description: Read what a finished Talkif call produced — the record with status/endReason/failureCode, the transcript, the recording (signed URL, deletion, retention), the per-line cost breakdown, conversation metrics, AI insights and on-demand analysis, plus listing and filtering call history. Use when code must fetch outcomes or build reporting.
---

# Read call results

## The record

`GET /calls/{callId}` → `status` (terminal: `completed | busy | noanswer | canceled | failed`), `agentStatus`, `endReason` (`caller_hangup | agent_completed | agent_timeout | agent_error | no_answer | busy | canceled | telephony_error | balance_depleted | reconciled | …`), `failureCode` (on `failed`: `insufficient_balance | capacity_limit | rate_limit | flow_not_published | phone_not_active | no_connected_flow | invalid_number | provider_error | …`), `direction`, `source` (`direct | scheduled | campaign`), `duration`, `contactId`, `campaignId`, `flowId`, timestamps, `isRecorded`, `isAnalyzed`, conversation metrics (time to first response, turns, interrupted turns, function-call count/time, user/bot speech seconds, tokens, TTS characters incl. wasted).

```typescript
const call = await client.calls.getCallDetails(callId);
```

## Transcript

`GET /calls/{callId}/transcript` — every turn with `role`, `content`, timing; the model used per message. Available immediately at call end (and live via `realtime-events`).

```python
t = client.calls.get_call_transcript(call_id)
```

## Recording

`GET /calls/{callId}/recording` → a **short-lived signed URL**; fetch when needed, don't store the URL. `DELETE /calls/{callId}/recording` removes the audio permanently (transcript stays). Whether a call was recorded follows flow → number → account settings; retention is an account setting; storage is billed per GB-month.

## Cost

`GET /billing/costs/calls/{callId}` → `lineItems[]` (telephony minutes rounded up; STT seconds; LLM input/output tokens; TTS characters; infrastructure seconds; analysis tokens), each with the **rate that applied at the time**, and `totalUsd`.

```typescript
const cost = await client.billing.getBillingCallCostBreakdown(callId);
```

## Insights

If analysis is on (account default, overridable per flow), the record carries `summary`, `sentiment`, `intents`, `topics`, `actionItems`, `outcome`, `callerInfo`. Otherwise, or to re-run with a newer model:

```typescript
await client.calls.analyzeCall(callId);   // requires a terminal call with a transcript
```
Per-flow `analysisInstructions` are appended to the analysis prompt ("extract budget and decision timeline").

## Listing and filtering

`GET /calls` — one list for live and finished calls (`status=inprogress` for live). Filters `status, direction, source, phoneNumber, isLead, flowId, contactId, scheduleId, campaignId, providerType, search, startDate, endDate`; `sortBy` (`status | direction | duration | totalCost | startedAt`), `sortDirection`; `limit`/`offset` (starts at 0, at most 100 per page). SDK: `calls.listCalls` / `list_calls` (0.2.0 replaced `getCallHistory` and `getActiveCalls`). Per contact: `GET /contacts/{contactId}/calls` (SDK `contacts.getContactCalls`).

`listCalls` returns **one page** — `calls` plus `meta` (`limit`, `offset`, `total`) — not an auto-pager; page with `offset`:
```typescript
let offset = 0;
for (;;) {
  const page = await client.calls.listCalls({ status: "completed", startDate: "2026-09-01T00:00:00Z", limit: 100, offset });
  for (const c of page.calls) console.log(c.id, c.endReason, c.duration);
  offset += page.calls.length;
  if (page.calls.length === 0 || offset >= page.meta.total) break;
}
```
```python
from datetime import datetime, timezone

offset = 0
while True:
    page = client.calls.list_calls(status="completed", start_date=datetime(2026, 9, 1, tzinfo=timezone.utc), limit=100, offset=offset)
    for c in page.calls:
        print(c.id, c.end_reason, c.duration)
    offset += len(page.calls)
    if not page.calls or offset >= page.meta.total:
        break
```
`contacts.getContactCalls` is a pager: `for await` over it fetches every page.

## Aggregates

`GET /flows/{flowId}/stats` (calls, outcomes, cost per version) · `GET /campaigns/{id}/analytics` · `GET /billing/costs` (totals by category over a period).

Docs: https://docs.talkif.ai/calls/transcripts-and-recordings · https://docs.talkif.ai/account/cost-breakdown
