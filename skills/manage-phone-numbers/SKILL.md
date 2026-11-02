---
name: manage-phone-numbers
description: Route Talkif phone numbers by API — find the system provider, list countries and pricing, search for a number (buying is dashboard-only, by the account Owner), connect or disconnect a published flow, pin a version, set a friendly name or recording override, release a number; plus what SIP/Twilio/WhatsApp providers need (dashboard-only). Use when code must give a flow a phone line.
---

# Manage phone numbers

## Concepts

A number belongs to a **provider**: the platform's system provider (Talkif-bought numbers), your SIP trunk, your own Twilio account, or a WhatsApp Business Account. Inbound calls are answered by the **one flow connected** to the number (latest published version, or a pinned one). A number can be the **from number** for outbound calls if it has the voice capability (`canInitiateCalls: true` — WhatsApp numbers are receive-only).

**Buying a number is dashboard-only.** Only the account **Owner**, signed in to the dashboard with email and phone verified, can buy. An API key can search, connect, update and release numbers, but not buy them.

**Until the account's first payment**, bonus and promotional credit (including the welcome credit) can't pay for a number's purchase or rental — the purchase fails with `insufficient_balance`. After the first payment (a top-up or auto-recharge that wasn't refunded or disputed), any credit pays for numbers, in the usual order. Rental that can't be paid → `past_due` → 7 days → `suspended` (no calls) → released 14 days later, unrecoverable.

## Find a number to buy (system provider)

Search by API, so the user can pick one:

```typescript
const providers = await client.phoneProviders.listPhoneProviders();
const system = providers.providers.find(p => p.isSystemProvider)!;

await client.phoneNumbers.listAvailableCountries();
await client.phoneNumbers.getPricing({ providerId: system.id, countryCode: "US" });
const avail = await client.phoneNumbers.listAvailableNumbers({ providerId: system.id, countryCode: "US", areaCode: "206", excludeAllAddressRequired: true });   // random batch; search again for more
```
```python
system = next(p for p in client.phone_providers.list_phone_providers().providers if p.is_system_provider)
avail = client.phone_numbers.list_available_numbers(provider_id=system.id, country_code="US", area_code="206", exclude_all_address_required=True)
```
`addressRequirements` on an available number (`none | any | local | foreign`) — anything but `none` needs regulatory paperwork on file first.

## Buy it (dashboard) and pick it up by API

Never call `purchasePhoneNumber` with an API key: `POST /phone/numbers/twilio/purchase` accepts only the Owner's dashboard session and answers a key with `403 api_key_not_allowed`. Ask the user to buy the number in the dashboard (the Owner), then find it by its E.164 number:

```typescript
const { phoneNumbers } = await client.phoneNumbers.listPhoneNumbers({ limit: 100 });
const number = phoneNumbers.find(n => n.phoneNumber === "+12065550123");
```
```python
page = client.phone_numbers.list_phone_numbers(limit=100)
number = next(n for n in page.phone_numbers if n.phone_number == "+12065550123")
```

## Connect a flow (inbound)

```typescript
await client.phoneNumbers.connectFlow(number.id, { flowId });                          // follows latest published
await client.phoneNumbers.connectFlow(number.id, { flowId, flowVersion: "1.0.7" });   // pinned
await client.phoneNumbers.disconnectFlow(number.id);                                   // inbound → busy until reconnected
```
Only published flows can be connected. A pinned number keeps its version until you publish *over* that exact version (then it moves forward). No flow connected → inbound calls get a busy signal and a failed record (`no_connected_flow`).

## Use as caller ID (outbound)

`POST /calls` with `fromNumber` = the number and `providerId` = its `providerId` (on the number record). Campaigns and schedules take the same pair.

## Update / release

```typescript
await client.phoneNumbers.updatePhoneNumber(id, { friendlyName: "Sales – Berlin", recordingOverride: "disabled" });  // use_account_default | enabled | disabled
await client.phoneNumbers.releasePhoneNumber(id);   // permanent; refused while the number is a Meta Lead Ads caller ID
```
Recording resolves **flow → number → account**; a flow that forces recording on still records on a number set to `disabled`.

## Record fields worth reading

`status` (`ACTIVE | PENDING | SUSPENDED | RELEASED`), `capabilities` (`VOICE | SMS | MMS | WHATSAPP`), `channelType` (`pstn | whatsapp`), `canInitiateCalls`, `billingStatus` (`active | processing | past_due | suspended`), `monthlyCost`, `providerId`, `connectedFlow` (id, version, connected at), `nextBillingDate`.

## Bringing your own carrier (dashboard-only)

Providers are created in the dashboard (**Phone Numbers → Providers → Add**), not via API:
- **SIP trunk**: host, port, protocol (UDP/TCP/TLS), username/password, and the carrier's public source IPs/CIDR. Talkif has one SIP address per region, both on port 5060: **EU `sip.talkif.ai` / `63.176.211.13`** and **US `sip.us.talkif.ai` / `44.218.212.38`**. Outbound calls leave from the address of the provider's region (picked from where the carrier's IPs are; US when that can't be determined), so the carrier must allowlist that address — **both if unsure**. Inbound calls can go to either address and run in that region. Talkif does not REGISTER. Numbers activate immediately.
- **Your Twilio account**: Account SID + Auth Token; pick numbers to register; Talkif sets their voice webhook.
- **WhatsApp Business**: Embedded Signup or WABA credentials; numbers are `PENDING` until Meta enables calling; receive-only; no telephony charge.

Docs: https://docs.talkif.ai/telephony/buy-a-number · https://docs.talkif.ai/telephony/inbound-and-outbound-routing · https://docs.talkif.ai/telephony/bring-your-own-sip-trunk
