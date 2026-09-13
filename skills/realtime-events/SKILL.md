---
name: realtime-events
description: Stream Talkif call events to your backend over WebSocket instead of polling — account-wide call.created/status/ended, per-call transcript/turn/transition/recording events, the envelope and seq semantics, replay for late joiners, and connection limits. Use when code must react to calls as they happen.
---

# Real-time events over WebSocket

## Concepts

`wss://api.talkif.ai/api/v1/ws/events` — authenticate with your API key as `Authorization: Bearer …` on the upgrade (needs the `calls` scope). Then send **subscribe** frames: no `call_id` = the account's call lifecycle; with `call_id` = that call's conversation stream. One long-lived connection per service instance; re-subscribe after any reconnect.

## Frames

Client → server (JSON, ≤ 4 KB):
```json
{ "type": "subscribe" }
{ "type": "subscribe", "call_id": "018f4e9e-…", "replay": true }
{ "type": "unsubscribe", "call_id": "018f4e9e-…" }
{ "type": "ping" }
```

Server → client envelope:
```json
{ "v": 1, "type": "transcript", "call_id": "018f4e9e-…", "seq": 42, "ts": 1757700000123,
  "data": { "role": "user", "content": "I'd like to book a cleaning.", "timestamp": 12.4, "turnNumber": 3 } }
```
`seq` is per-connection and monotonic; a gap means droppable frames were skipped under backpressure on *this* connection. Control frames: `subscribed` / `unsubscribed` (echo `call_id`), `pong`.

## Event types

Account-wide: `call.created` (full call object + `flowName`), `call.status` (`callId, status, direction, source`, numbers, `contactId`, `campaignId`), `call.ended` (`endReason, failureCode, failureReason, duration, cost` (USD string), `recordingUrl`, campaign/schedule/contact ids).

Per call: `status`, `transcript` (`role` user|assistant, `content`, `timestamp`, `turnNumber`), `interim`*, `turn`, `node_transition` (`fromNode`, `toNode`), `tts`*, `tts_word`*, `speech`, `recording`, `voicemail`, `error`. `*` = high-frequency and droppable under backpressure; everything else is never dropped.

## Node.js

```javascript
import WebSocket from "ws";
const ws = new WebSocket("wss://api.talkif.ai/api/v1/ws/events", {
  headers: { Authorization: `Bearer ${process.env.TALKIF_API_KEY}` },
});
ws.on("open", () => ws.send(JSON.stringify({ type: "subscribe" })));
ws.on("message", raw => {
  const f = JSON.parse(raw.toString());
  if (f.type === "call.created") ws.send(JSON.stringify({ type: "subscribe", call_id: f.data.call.id, replay: true }));
  if (f.type === "transcript") console.log(f.call_id, f.data.role, f.data.content);
  if (f.type === "call.ended") { /* write outcome to CRM: f.data.callId, endReason, duration, cost */ }
});
ws.on("close", () => { /* reconnect with backoff, then re-subscribe */ });
```

## Python

```python
import asyncio, json, os, websockets
async def main():
    async with websockets.connect("wss://api.talkif.ai/api/v1/ws/events",
            additional_headers={"Authorization": f"Bearer {os.environ['TALKIF_API_KEY']}"}) as ws:
        await ws.send(json.dumps({"type": "subscribe"}))
        async for raw in ws:
            f = json.loads(raw)
            if f["type"] == "call.ended": print(f["data"]["callId"], f["data"].get("endReason"))
asyncio.run(main())
```

## Replay

`"replay": true` on a per-call subscribe streams the call's history first, then live. Late-joining consumers get the whole transcript. A live event racing the replay may appear twice — de-duplicate on `turnNumber`/`timestamp`, not `seq`.

## Limits and hygiene

Server ping every 25 s (answer pongs) · idle close after 300 s without an inbound frame (`1008`) · 50 subscriptions per connection · 20 connections per account · 3 malformed frames → close · on server restart the socket closes: reconnect with backoff, re-subscribe, use `replay: true` for in-progress calls.

Docs: https://docs.talkif.ai/integrate/real-time-events
