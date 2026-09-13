---
name: manage-contacts
description: Keep Talkif contacts in sync with your CRM — create, update, search by phone, tag and segment, read a contact's call timeline, soft-delete and restore, import and export. Use when code owns the contact list that campaigns, schedules and prompt variables draw on.
---

# Manage contacts

## Concepts

A contact is the context an agent gets about the person on the line, and the handle that ties calls together. Its fields feed `{{contact.*}}` prompt variables; its **tags** are how campaign lists get built; its **timezone** (explicit, or derived from the phone number) drives call windows and schedules. Primary phone is the identity — one contact per primary phone per account.

Talkif also **creates or enriches** contacts from calls after analysis (unknown caller → new contact; details the caller mentioned → empty fields only, never overwriting yours).

## Create / update

`POST /api/v1/contacts` · `PUT /api/v1/contacts/{contactId}`

```json
{
  "name": "Ada Lovelace",
  "primaryPhone": "+442071234567",
  "secondaryPhones": ["+442071234568"],
  "email": "ada@example.com",
  "company": "Analytical Engines",
  "occupation": "Founder",
  "address": { "street": "1 Pine St", "city": "London", "state": null, "zip": "N1 1AA", "country": "GB" },
  "notes": "Prefers mornings. Renewal due in November.",
  "tags": ["vip", "renewal-q4"],
  "interests": ["automation"],
  "timezone": "Europe/London",
  "language": "en"
}
```
`name` and `primaryPhone` (E.164) required; up to 10 secondary phones, 50 tags, 5,000-char notes.

```typescript
const c = await client.contacts.createContact({ name, primaryPhone: "+442071234567", tags: ["vip"] });
await client.contacts.updateContact(c.id, { notes: "Called 12 Sep — wants a quote." });
```
```python
c = client.contacts.create_contact(name="Ada Lovelace", primary_phone="+442071234567", tags=["vip"])
client.contacts.update_contact(c.id, notes="Called 12 Sep — wants a quote.")
```

## Find

- `GET /contacts/search/phone/{phone}` — exact by number; the right lookup when a call comes in from your telephony or a CRM record.
- `GET /contacts?search=&tag=&tags=&tagMode=any|all&company=&includeDeleted=&limit=&offset=` — `search` matches name, company, occupation, email. SDK returns a pager.
- `GET /contacts/tags` — every tag with counts.

## Tags

```typescript
await client.contacts.addTags(contactId, { tags: ["renewal-q4"] });
await client.contacts.removeTags(contactId, { tags: ["lead"] });
```
Tags are the bridge to campaigns: `bulkAddCampaignContacts` takes `{ filter: { tags, tagMode, company, search } }`.

## Timeline

`GET /contacts/{contactId}/calls` — every call the contact was on, any direction or source, each linking to transcript/recording/cost. `lastContactedAt` on the record is the latest.

## Delete / restore

`DELETE /contacts/{contactId}` is a **soft delete**: out of lists, campaigns and search; history kept; `POST /contacts/{contactId}/restore` brings it back. Deleting does **not** stop calls to the number — use the `do-not-call` skill for that.

## Import / export

`POST /contacts/import` `{ fileContent: <base64>, format: "csv" | "json" | "vcf", defaultCountryCode?: "44", confirmed: false }` returns a dry-run preview (to create / skipped duplicates / errors); repeat with `confirmed: true` to run. Duplicates by primary phone are skipped. `GET /contacts/export?format=csv|json|vcf` returns the list in the same shapes.

## Sync pattern (CRM → Talkif)

1. Look up by phone; create if missing, else update changed fields only.
2. Write your CRM id into `notes` or a tag (`crm:12345`) — there is no external-id field.
3. Mirror unsubscribes to `POST /dnc`, not to contact deletion.

Docs: https://docs.talkif.ai/calls/contacts · https://docs.talkif.ai/calls/import-contacts
