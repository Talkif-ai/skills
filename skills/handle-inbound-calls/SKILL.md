---
name: handle-inbound-calls
description: Make a Talkif number answer calls — what happens on an inbound call (lookup, flow, balance, version), designing a flow for inbound, unknown-caller contact creation, prompts that branch on call.direction, and the reasons an inbound call is answered busy. Use when the job is receiving calls rather than placing them.
---

# Handle inbound calls

## What happens when someone dials your number

1. The number is looked up. Not yours → `unknown_phone_number`.
2. It must have a **connected flow** (`phoneNumbers.connectFlow`). None → answered **busy**, failed call recorded with `failureCode: no_connected_flow`. Archived or unpublished flow behaves the same way — connect a published one.
3. Balance ≥ $1.00 and account active. Else → busy, `insufficient_balance`, `endReason: balance_depleted`.
4. Version resolved (*latest* or pinned) → capacity → the agent answers. `direction: inbound`, `source: direct`.

Inbound calls are **not queued**: if every concurrent slot is busy the call is refused — size the account limit for peak inbound.

## Contacts and inbound

If the caller's number matches a contact's primary or secondary phone, `{{contact.*}}` variables are filled and the call lands on their timeline. Unknown callers get only `{{contact.primaryPhone}}`; after the call is analysed, Talkif **creates a contact** from what the caller said (name, company, email) — so repeat callers are recognised next time.

Look a caller up yourself when your systems get the `call.created` event: `GET /contacts/search/phone/{phone}`.

## Designing an inbound flow

- Let the prompt branch on direction: `{{#if (eq call.direction "inbound")}}Thank them for calling and ask how you can help.{{else}}Confirm it's a good time.{{/if}}`.
- Triage → specialists is the natural inbound shape (`create-flow`): the main agent classifies, sub-agents handle billing / support / booking, each with a return transition.
- Give the main agent an **End Call** function with a clear end condition, and turn on the flow's **maximum call length** — inbound callers can keep an agent talking indefinitely.
- Identity-gated actions: put a flow function with `caller.contact_id` bound `onNull: reject` on a dedicated agent, so an unknown caller can't reach the sensitive step (`create-flow-function`).
- Voicemail detection doesn't apply inbound; silence handling does (on by default).

## Recording and consent

Recording resolves flow → number → account. If your jurisdiction needs an announcement, make the main agent say it (prompt) or add a `tts_say` pre-action so it's verbatim every time; set `recordingOverride: "disabled"` on numbers that must never record.

## Multiple lines

One flow can be connected to many numbers; each number can override recording and carries its own friendly name. Use `{{call.toNumber}}` in the prompt if one flow should behave differently per line ("Northwind Dental — Pine St" vs "— Bellevue").

## Watching it

`realtime-events` account stream: `call.created` with `direction: "inbound"` → subscribe to the `call_id` for transcript. A rising count of `no_connected_flow` or `insufficient_balance` failed calls in `GET /calls?direction=inbound&status=failed` is the alert that matters.

Docs: https://docs.talkif.ai/telephony/inbound-and-outbound-routing · https://docs.talkif.ai/getting-started/how-a-call-works
