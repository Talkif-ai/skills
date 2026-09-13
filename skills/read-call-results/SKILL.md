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

`GET /calls` — one list for live and finished calls (`status=inprogress` for live). Filters `status, direction, source, phoneNumber, isLead, flowId, contactId, scheduleId, campaignId, providerType, search, startDate, endDate`; `sortBy` (`status | direction | duration | totalCost | startedAt`), `sortDirection`; `limit`/`offset` (starts at 0). The 0.1.x SDKs expose this list as `getCallHistory` / `get_call_history` (and `getActiveCalls` for live calls). Per contact: `GET /contacts/{contactId}/calls`.

```typescript
for await (const c of await client.calls.getCallHistory({ status: "completed", startDate: "2026-09-01T00:00:00Z" })) {
  console.log(c.id, c.endReason, c.duration);
}
```
```python
for c in client.calls.get_call_history(status="completed", start_date="2026-09-01T00:00:00Z"):
    print(c.id, c.end_reason, c.duration)
```
SDK list calls return a pager; iterate to page automatically.

## Aggregates

`GET /flows/{flowId}/stats` (calls, outcomes, cost per version) · `GET /campaigns/{id}/analytics` · `GET /billing/costs` (totals by category over a period).

Docs: https://docs.talkif.ai/calls/transcripts-and-recordings · https://docs.talkif.ai/account/cost-breakdown
