---
name: create-schedule
description: Call one contact on a recurrence with Talkif — once, daily, weekly, monthly or cron — at a wall-clock time in the contact's own timezone, with per-run retries, a run cap or end date, and pause/resume. Use for check-ins, reminders and follow-ups tied to a single person.
---

# Create a schedule

## Concepts

A schedule is a standing instruction: *call this contact, with this flow, at this time, this often*. Each run is an ordinary outbound call — it passes Do Not Call, legal hours, balance and capacity checks and goes through the queue (scheduled calls sit ahead of campaign calls; a run that can't be placed within 10 minutes is dropped for that occurrence rather than placed late).

The time is a wall-clock `HH:MM` **in the contact's timezone** — their explicit zone, else the zone derived from their phone number. You don't pass a timezone. DST is handled.

## Create

`POST /api/v1/schedules`

```json
{
  "name": "Weekly check-in — Ada",
  "contactId": "<contact-uuid>",
  "flowId": "<published-flow-uuid>",
  "fromPhoneNumber": "+15559876543",
  "providerId": "<provider-uuid of that number>",
  "frequency": "weekly",
  "daysOfWeek": [1, 4],
  "scheduleTime": "09:30",
  "retryOnFailure": true,
  "retryDelayMinutes": 30,
  "maxRetries": 2,
  "endDate": "2026-12-31",
  "maxRuns": null
}
```

| `frequency` | Extra fields |
|---|---|
| `once` | — (fires at the next `scheduleTime`) |
| `daily` | — |
| `weekly` | `daysOfWeek` 1=Mon … 7=Sun |
| `monthly` | `dayOfMonth` 1–28 (28 is the ceiling on purpose — no month is skipped) |
| `custom_cron` | `cronExpression` |

Retries: `retryOnFailure` (default off), `retryDelayMinutes` 1–1440 (default 30), `maxRetries` 0–10 (default 2). A run whose call ends `no_answer` / `busy` / `failed` / `canceled` is retried; an answered call is a successful run. Stop with `endDate` and/or `maxRuns` — whichever first.

```typescript
const s = await client.schedules.createSchedule({ name, contactId, flowId, fromPhoneNumber, providerId,
  frequency: "weekly", daysOfWeek: [1, 4], scheduleTime: "09:30", retryOnFailure: true, maxRetries: 2 });
```
```python
s = client.schedules.create_schedule(name=..., contact_id=..., flow_id=..., from_phone_number="+15559876543",
  provider_id=..., frequency="weekly", days_of_week=[1, 4], schedule_time="09:30", retry_on_failure=True, max_retries=2)
```

## Manage

`GET /schedules?status=active|paused|completed|canceled` · `GET /schedules/{id}` · `PUT /schedules/{id}` (any rule field) · `POST /schedules/{id}/pause` · `POST /schedules/{id}/resume` · `DELETE /schedules/{id}` (cancels; no further runs).

Status: `active` → `paused` ⇄ `active`; `completed` when `once` fired or the cap/end date is reached; `canceled` on delete.

## Reading runs

Calls placed by a schedule carry `source: "scheduled"` and `scheduleId` — filter `GET /calls?scheduleId=…` or watch `call.ended` events with `scheduleId` in `realtime-events`.

## Gotchas

- Runs use the flow's **latest published** version at the time each fires.
- If the contact's number is added to Do Not Call, every subsequent run is refused — the schedule doesn't cancel itself; delete or pause it.
- Several schedules per contact are fine; each keeps its own history.

Docs: https://docs.talkif.ai/calls/schedules
