---
name: read-billing
description: Read Talkif billing programmatically — balance and its buckets, the $1.00 call-admission gate and account suspension semantics, charges and their types/statuses, the balance transaction ledger, cost totals by period, invoices, and public pricing. Use for spend monitoring, alerts, finance exports and cost attribution.
---

# Read billing

## Concepts

Talkif is prepaid. The balance is made of **buckets** spent in order: `subscription` → `promotional` → `bonus` → `purchased`. Calls and analysis can use any bucket; **phone purchases and rental use only purchased/subscription**. A call needs **≥ $1.00 available** to start and is never cut off mid-call; if a charge takes the balance ≤ 0 the account is **suspended** (no new calls, everything else open) until a top-up — unless auto-recharge is on. Topping up, cards and auto-recharge are dashboard-only; the API is read-only for money.

## Balance

`GET /billing/balance` → `{ available, purchasedBalance, bonusBalance, earliestExpiration }` (USD strings).

```typescript
const b = await client.billing.getBalanceSummary();
if (Number(b.available) < 25) alert(`Talkif balance low: $${b.available}`);
```
```python
b = client.billing.get_balance_summary(); print(b.available, b.purchased_balance)
```
Alert threshold: a day of spend at your busiest, above your auto-recharge threshold if you use one.

## Charges — the billable events

`GET /billing/charges?charge_type=&status=&start_date=&end_date=&limit=&offset=` · `GET /billing/charges/{chargeId}` (line items, associated entity, status history).

`charge_type`: `call_usage`, `call_analysis`, `voicemail_classification`, `phone_purchase`, `phone_rental`, `recording_storage`, `assistant_usage`, `reservation` (pre-auth hold during a call), `adjustment`.
`status`: `pending` → `completed` | `failed` | `refunded` | `partially_refunded` | `disputed` | `voided`. A call's charge is `pending` while it runs.

## Transactions — the ledger

`GET /billing/transactions?limit=&offset=` → every balance movement with `balanceBefore`/`balanceAfter` and its bucket. Types: `purchase` (incl. auto-recharge), `usage`, `refund`, `refund_deduction`, `adjustment`, `expiration`, `subscription_grant`, `dispute_hold`, `dispute_release`, `dispute_loss`.

## Cost totals and per-call cost

`GET /billing/costs?period=today|week|month|custom`GET /billing/costs?period=&startDate=&endDate=` → totals by categorystartDate=`GET /billing/costs?period=&startDate=&endDate=` → totals by categoryendDate=` → totals by category (LLM, STT, TTS, telephony) with period comparison. `GET /billing/costs/calls/{callId}` → one call's line items (telephony minutes rounded up; STT seconds; LLM in/out tokens; TTS characters; infrastructure seconds; analysis tokens), each with the rate that applied at the time.

Attribution: every call record has `flowId`, `campaignId`, `contactId` — join charges to them via `GET /calls?…` for cost per flow, campaign or customer. `GET /flows/{flowId}/stats` and `GET /campaigns/{id}/analytics` give ready-made rollups.

## Invoices and pricing

`GET /billing/invoices` · `GET /billing/invoices/{invoiceId}` (receipts for top-ups). `GET /pricing` (no auth) and `GET /models` publish current rates — never hard-code a price.

Docs: https://docs.talkif.ai/concepts/credits-and-pricing · https://docs.talkif.ai/account/cost-breakdown
