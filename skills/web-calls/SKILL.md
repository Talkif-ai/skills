---
name: web-calls
description: Put a Talkif voice agent on a web page — enable web calls on a flow, get a publishable key, allowlist origins, use @talkif/webrtc or the useTalkifCall React hook, render live transcript/caption events, and respect the public-call limits. Use for browser-based "call us" buttons and demos; never for backend integrations.
---

# Web calls (browser SDK)

## Concepts

The browser never holds an API key. A **publishable key** (`pk_live_…`) is bound to **one flow**, valid only from **allowlisted origins**, and rate-limited per key/session/IP. The SDK exchanges it for a short-lived session token, then runs the call over WebRTC and streams events over a WebSocket. Both packages are headless — you build the UI.

Enable: **Flow Builder → flow → Flow Settings → Web calls**: enable, copy the key (shown once), add origins (`https://www.example.com`), set concurrent calls (default 2) and calls per visitor session (default 3). Publish the flow — web calls run the published version.

```sh
npm install @talkif/webrtc @talkif/webrtc-react   # react bindings optional; React ≥ 18
```

## Config — two modes

```ts
// Public (embed): key is safe in page source
const config = { baseUrl: "https://api.talkif.ai", publishableKey: "pk_live_…" };
// Authenticated: your signed-in app; can call any published flow or a draft definition
const config = { baseUrl: "https://api.talkif.ai", accountId: "<account-id>", auth: async () => `Bearer ${await getToken()}` };
```
`baseUrl` is required. If the flow's bot gate is on, the SDK solves the Cloudflare Turnstile challenge invisibly; pass `turnstileToken: () => Promise<string>` to supply your own.

## React

```tsx
import { useTalkifCall } from "@talkif/webrtc-react";

export function CallButton() {
  const { state, durationSecs, muted, start, hangup, toggleMute } = useTalkifCall({
    config,
    onTranscript: ({ role, content }) => append(role, content),
    onTtsWord: ({ word }) => caption(word),
    onEnded: reason => console.log("ended", reason),
    onError: error => console.error(error.code, error.message),
  });
  if (state === "connected") return <><span>{durationSecs}s</span><button onClick={toggleMute}>{muted ? "Unmute" : "Mute"}</button><button onClick={hangup}>Hang up</button></>;
  return <button onClick={() => start({})} disabled={state === "requesting" || state === "connecting"}>Call us</button>;
}
```
Hook returns `{ state, callId, botId, error, durationSecs, muted, call, start, hangup, toggleMute, sendAppMessage }`; callbacks `onAppMessage onCallEvent onTranscript onTtsWord onTtsChunk onEnded onError`. One active call per hook; unmount tears down mic and peer connection.

## Core (no framework)

```ts
import { TalkifCall } from "@talkif/webrtc";
const call = new TalkifCall(config);
call.on("connected", ({ callId }) => …);
call.on("transcript", ({ role, content }) => …);
call.on("ended", ({ reason }) => …);          // local-hangup | peer-left | terminal-status | external
await call.start({});                           // public mode: flow implied by the key. Authenticated: start({ flowId }) or { draftDefinition }
call.setMuted(true); call.hangup(); call.dispose();  // instances are single-use
```
Events: `statechange` (`requesting → connecting → connected → ended`), `connected`, `track`, `tick`, `mutechange`, `appmessage`, `renegotiated`, `ended`, `error`; conversation: `transcript`, `interim`*, `ttschunk`* (sentence ahead of audio), `ttsword`* (word timing at playback pace — captions), `*` droppable.

## Limits (public mode)

20 key exchanges/IP/hour · 10 calls/IP/hour · 60 calls/key/hour · per-key concurrency (yours, default 2) · per-session calls (yours, default 3), session lifetime 30 min, one active call per session · account concurrency and the $1.00 balance gate apply · disabling web calls or regenerating the key stops new calls immediately. Calls are billed like phone calls minus telephony and show `providerType: webrtc`.

Docs: https://docs.talkif.ai/integrate/web-calls · source: https://github.com/Talkif-ai/webrtc-js
