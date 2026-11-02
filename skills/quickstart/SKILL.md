---
name: quickstart
description: End-to-end orchestration for a first Talkif integration — API key → flow from a template → phone number → connect → test call → read results — naming which skill to load at each step. Use as the entry point when asked to "set up Talkif", "build a phone agent", or "make my first call" from code.
---

# Talkif quickstart (orchestration)

Load the named skill for detail at each step; this skill only sequences them.

1. **Authenticate** — `setup-api-key`. Result: `client` with a key that has `calls:*`, `flows:*`, `flow_templates:*`, `phones:*`, `contacts:*`. Verify with `GET /calls`.
2. **Create a flow** — `create-flow`. Fastest: `flowTemplates.instantiateTemplate("customer-support", { name })`, then `flows.updateFlow(id, { definition })` to set `systemPrompt`/`agentPrompt` (`write-prompts`). Attach any backend calls (`create-flow-function`).
3. **Validate and publish** — `flows.validateFlow(id)` → fix → `flows.publishFlow(id, { notes })`. Only a published flow takes calls.
4. **Get a line** — `manage-phone-numbers`: system provider → search by API → **the account Owner buys it in the dashboard** (an API key can't buy: `403 api_key_not_allowed`) → find it with `phoneNumbers.listPhoneNumbers`. Until the account's first payment, the welcome credit can't pay for a number: the Owner tops up first.
5. **Connect inbound** — `phoneNumbers.connectFlow(numberId, { flowId })`. Dialing the number now reaches the agent (`handle-inbound-calls`).
6. **Test** — `test-a-flow`: place a real call to the user's own phone with `calls.makeCall`, or run the draft in the browser with the authenticated SDK. Treat `202` as success (`make-call`).
7. **Read results** — `read-call-results`: `getCallDetails`, `getCallTranscript`, `getBillingCallCostBreakdown`. For live updates, `realtime-events`.
8. **Scale** — a list → `create-campaign` (+ `manage-contacts`, `do-not-call`); one person on a cadence → `create-schedule`; a web button → `web-calls`; spend alerts → `read-billing`. Anything failing → `troubleshoot-calls`.

Minimal working script (TypeScript):
```typescript
import { TalkifClient } from "@talkif/sdk";
const client = new TalkifClient({ token: process.env.TALKIF_API_KEY });

const flow = await client.flowTemplates.instantiateTemplate("customer-support", { name: "Support line" });
await client.flows.publishFlow(flow.id, { notes: "initial" });

// Buying a number is dashboard-only (the account Owner); use one the account already has.
const { phoneNumbers } = await client.phoneNumbers.listPhoneNumbers({ limit: 100 });
const number = phoneNumbers.find(n => n.phoneNumber === "+1YOURTALKIFNUMBER")!;
await client.phoneNumbers.connectFlow(number.id, { flowId: flow.id });

const result = await client.calls.makeCall({ flowId: flow.id, fromNumber: number.phoneNumber, providerId: number.providerId, toNumber: "+1YOURPHONE" });
console.log(result.type === "initiated" ? result.callId : `queued #${result.position}`);
```
Docs: https://docs.talkif.ai/getting-started/quickstart
