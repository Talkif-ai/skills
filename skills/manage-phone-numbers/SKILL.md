---
name: manage-phone-numbers
description: Provision and route Talkif phone numbers by API — find the system provider, list countries and pricing, search and buy a number, connect or disconnect a published flow, pin a version, set a friendly name or recording override, release a number; plus what SIP/Twilio/WhatsApp providers need (dashboard-only). Use when code must give a flow a phone line.
---

# Manage phone numbers

## Concepts

A number belongs to a **provider**: the platform's system provider (Talkif-bought numbers), your SIP trunk, your own Twilio account, or a WhatsApp Business Account. Inbound calls are answered by the **one flow connected** to the number (latest published version, or a pinned one). A number can be the **from number** for outbound calls if it has the voice capability (`canInitiateCalls: true` — WhatsApp numbers are receive-only).

Purchases and monthly rental are paid from **purchased** balance only — bonus/promotional credit (including the welcome credit) never buys a number. Rental that can't be paid → `past_due` → 7-day grace → `suspended` (calls stop until topped up).

## Buy a number (system provider)

```typescript
const providers = await client.phoneProviders.listPhoneProviders();
const system = providers.providers.find(p => p.isSystemProvider);

await client.phoneNumbers.listAvailableCountries();
await client.phoneNumbers.getPricing({ providerId: system.id, countryCode: "US" });
const avail = await client.phoneNumbers.listAvailableNumbers({ providerId: system.id, countryCode: "US", areaCode: "206", excludeAllAddressRequired: true });   // random batch; search again for more
const number = await client.phoneNumbers.purchasePhoneNumber({
  phoneNumber: avail.numbers[0].phoneNumber, providerId: system.id, friendlyName: "Support line",
});
```
```python
system = next(p for p in client.phone_providers.list_phone_providers().providers if p.is_system_provider)
number = client.phone_numbers.purchase_phone_number(phone_number="+12065550123", provider_id=system.id, friendly_name="Support line")
```
`addressRequirements` on an available number (`none | any | local | foreign`) — anything but `none` needs regulatory paperwork on file first.

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
- **SIP trunk**: host, port, protocol (UDP/TCP/TLS), username/password, and the carrier's source IPs/CIDR. The carrier must allowlist Talkif's SIP address **`sip.talkif.ai` / `63.176.211.13`** (port 5060) and route inbound calls to it; Talkif does not REGISTER. Numbers activate immediately.
- **Your Twilio account**: Account SID + Auth Token; pick numbers to register; Talkif sets their voice webhook.
- **WhatsApp Business**: Embedded Signup or WABA credentials; numbers are `PENDING` until Meta enables calling; receive-only; no telephony charge.

Docs: https://docs.talkif.ai/telephony/buy-a-number · https://docs.talkif.ai/telephony/inbound-and-outbound-routing · https://docs.talkif.ai/telephony/bring-your-own-sip-trunk
