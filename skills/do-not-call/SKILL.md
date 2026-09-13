---
name: do-not-call
description: Enforce Do Not Call on Talkif — add and remove numbers, bulk-import a suppression list, check numbers before building a campaign, wire an unsubscribe flow to the API, and understand where the check runs. Use for compliance code and for keeping Talkif in sync with your CRM's opt-outs.
---

# Do Not Call list

## Concepts

The DNC list is a per-account list of **phone numbers**, checked before **every** outbound call — direct, campaign, schedule, API, Meta lead. On the list → the call is refused (`contact_on_dnc_list`) or, in a campaign, the contact is **skipped** with reason `dnc_list`. No per-call override. Inbound calls from a listed number are still answered. It is not a contact flag: deleting a contact doesn't block their number; this does.

## Endpoints

| | SDK |
|---|---|
| `POST /dnc` `{ phoneNumber, reason?, source? }` | `doNotCall.addDncEntry` / `do_not_call.add_dnc_entry` |
| `POST /dnc/bulk` `{ phoneNumbers[1..10000], reason?, source? }` | `bulkImportDnc` |
| `POST /dnc/check` `{ phoneNumbers[1..100] }` → `{ results: { "+1…": true } }` | `checkDncStatus` |
| `GET /dnc?search=&source=&limit=&offset=` | `listDncEntries` |
| `DELETE /dnc/{phoneNumber}` | `removeDncEntry` |

`source` records provenance: `manual`, `api`, `call_opt_out` (the person asked during a call), `bulk_import`. Use `api` from your own systems.

```typescript
await client.doNotCall.addDncEntry({ phoneNumber: "+15551234567", reason: "Unsubscribed via email", source: "api" });
const { results } = await client.doNotCall.checkDncStatus({ phoneNumbers: numbers.slice(0, 100) });
```
```python
client.do_not_call.add_dnc_entry(phone_number="+15551234567", reason="Unsubscribed via email", source="api")
```

## Patterns

- **Unsubscribe webhook → `POST /dnc`.** Talkif can't see a stop request that landed in your inbox or CRM; the moment your system records one, add the number.
- **Scrub before a campaign.** `POST /dnc/check` in batches of 100, so your "contacts to call" count is right and skipped rows don't surprise anyone.
- **Registry refresh.** Import your regulatory suppression list with `POST /dnc/bulk` (10,000 per request) on its cycle; `reason` = registry name + date.
- **Audit.** `GET /dnc?source=call_opt_out` shows who asked during a call — proof of honouring requests.

Docs: https://docs.talkif.ai/calls/do-not-call
