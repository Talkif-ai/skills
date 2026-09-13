---
name: create-campaign
description: Call a list of contacts with one Talkif flow — import contacts, scrub against Do Not Call, create a campaign with concurrency, call windows in contact-local time, start/pause/resume/cancel/restart, read per-contact statuses and analytics. Use for any outbound calling of more than a handful of people.
---

# Create and run a campaign

## Concepts

A campaign = contacts + a published flow + a from-number + pacing rules. Talkif places calls up to your **concurrency**, only inside your **call windows** (evaluated in each contact's local time by default), retries `no_answer`/`busy`/`failed` up to **3 attempts**, skips anyone on the **Do Not Call** list, and never redials an answered contact. Legal calling hours by jurisdiction are enforced on top of your windows.

Lifecycle: `draft → pending → running ⇄ (draining) → paused | completed | canceled`; `restart` re-queues every non-completed contact.

## 1. Get contacts in

```typescript
// preview first (confirmed:false), then run
await client.contacts.importContacts({ fileContent: base64Csv, format: "csv", defaultCountryCode: "1", confirmed: false });
await client.contacts.importContacts({ fileContent: base64Csv, format: "csv", defaultCountryCode: "1", confirmed: true });
```
CSV columns: `name, primary_phone` (required), `email, company, occupation, notes, tags|pipe|separated, interests, timezone, language, secondary_phones`. Duplicates by primary phone are skipped. **Tag on import** (`tags` column) so the list can be added to a campaign by filter.

Scrub before building: `POST /dnc/check` with up to 100 numbers returns which are blocked.

## 2. Create

`POST /api/v1/campaigns`
```json
{
  "name": "Q4 renewals",
  "flowId": "<published-flow-uuid>",
  "fromPhoneNumber": "+15559876543",
  "providerId": "<provider-uuid of that number>",
  "concurrentCalls": 5,
  "callWindows": [ { "startTime": "09:00", "endTime": "17:00", "dayOfWeek": null } ],
  "respectContactTimezone": true,
  "startAt": null,
  "endAt": "2026-10-01T00:00:00Z"
}
```
`concurrentCalls` 1–50 (also bounded by the account limit, default 10). `callWindows` are HH:MM; `dayOfWeek` 1=Mon…7=Sun or `null` for every day; with `respectContactTimezone` they mean the contact's local time. `startAt` null = manual start; `endAt` auto-pauses.

## 3. Add contacts

```typescript
await client.campaigns.addCampaignContacts(campaignId, { contactIds: [...] });          // up to 1,000 per request
await client.campaigns.bulkAddCampaignContacts(campaignId, { filter: { tags: ["renewal-q4"], tagMode: "all", company: "Acme" } });
```
Filters combine with AND; `tagMode` `any` | `all`; `search` matches name/company/occupation/email.

## 4. Run

```typescript
await client.campaigns.startCampaign(campaignId);
await client.campaigns.pauseCampaign(campaignId);    // drains active calls, then paused
await client.campaigns.resumeCampaign(campaignId);
await client.campaigns.cancelCampaign(campaignId);
await client.campaigns.restartCampaign(campaignId);  // from paused/completed/canceled: re-queue non-completed
await client.campaigns.skipCampaignContact(campaignId, contactId, { reason: "manual" });
```
```python
client.campaigns.start_campaign(campaign_id)  # pause_campaign / resume_campaign / cancel_campaign / restart_campaign
```

Auto-pause reasons you'll see: `insufficient_balance`, `system_limit`, `provider_error`, end time reached. Fix and resume.

## 5. Read progress

- `GET /campaigns/{id}` — counts: total, contacted, remaining, not called, active.
- `GET /campaigns/{id}/contacts` — per-contact `status`: `pending | queued | calling | completed | no_answer | failed | skipped` (derived from the latest call).
- `GET /campaigns/{id}/analytics` — outcomes %, attempt distribution, voicemail/human/undetermined, failure causes, duration histogram, cost breakdown, performance.
- Live: subscribe account-wide in the `realtime-events` skill; every campaign call carries `campaignId` and `campaignContactId`.

## Gotchas

- Don't loop `POST /calls` for a list — you'll hit the 60/min rate limit and re-implement retries and DNC badly.
- Watch the first few transcripts before letting it run; a prompt problem shows at call 3, not call 300.
- Outbound voicemail detection is a **flow** setting (`voicemailEnabled`), not a campaign one.
- Deleting a contact does not stop calls to their number; the DNC list does (`POST /dnc`).

Docs: https://docs.talkif.ai/calls/campaigns · https://docs.talkif.ai/calls/import-contacts · https://docs.talkif.ai/calls/do-not-call
