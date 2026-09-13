---
name: verify-webhooks
description: Verify that an HTTP request came from Talkif — create a signing secret, parse the Talkif-Signature header (t + one or more v1), compute HMAC-SHA256 over "{t}.{rawBody}", compare in constant time, handle rotation overlap and empty bodies. Use when implementing the endpoint a flow function calls.
---

# Verify the `Talkif-Signature` header

## Concepts

Talkif signs every request it makes to your endpoints (flow function calls) once you create a **per-account signing secret** — dashboard **Developer → Credentials → Webhook signing**. The secret is `whsec_` + 43 URL-safe chars (49 total), shown once. Until you create one, requests are unsigned.

Header: `Talkif-Signature: t=1718983045,v1=3f9a…c2` — `t` is a Unix timestamp (seconds); `v1` is lowercase hex `HMAC-SHA256(secret, "{t}.{rawBody}")`. During rotation there are **two `v1` values**; accept if **any** matches.

Two mistakes account for nearly every failed verification:
1. **Decoding the secret.** Use the whole `whsec_…` string verbatim as the UTF-8 HMAC key. Do not strip the prefix, do not base64-decode (Svix-style verifiers do; Talkif does not).
2. **Re-serialising the body.** Sign over the exact raw bytes received. Frameworks that parse JSON first change key order/whitespace. Capture the raw body before parsing.

Empty body (`GET`, `HEAD`, or `{}` body): the signed payload is `"{t}."` — timestamp, dot, nothing.

## Node.js

```javascript
const crypto = require("crypto");

function verifyTalkifWebhook(rawBody, signatureHeader, secret) {
  let t; const v1s = [];
  for (const part of signatureHeader.split(",")) {
    const [k, v] = part.split("=");
    if (k === "t") t = v; else if (k === "v1") v1s.push(v);
  }
  if (!t || v1s.length === 0) return false;
  if (Math.abs(Date.now() / 1000 - Number(t)) > 300) return false;   // replay window
  const expected = crypto.createHmac("sha256", secret).update(`${t}.${rawBody}`, "utf8").digest("hex");
  return v1s.some(v1 => v1.length === expected.length && crypto.timingSafeEqual(Buffer.from(v1), Buffer.from(expected)));
}
// Express: app.post("/hook", express.raw({ type: "*/*" }), (req, res) => { verifyTalkifWebhook(req.body.toString("utf8"), req.get("Talkif-Signature") ?? "", process.env.TALKIF_SIGNING_SECRET) ... })
```

## Python

```python
import hashlib, hmac, time

def verify_talkif_webhook(raw_body: bytes, signature_header: str, secret: str) -> bool:
    t, v1s = None, []
    for part in signature_header.split(","):
        key, _, value = part.partition("=")
        if key == "t": t = value
        elif key == "v1": v1s.append(value)
    if not t or not v1s: return False
    if abs(time.time() - int(t)) > 300: return False
    expected = hmac.new(secret.encode(), t.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return any(hmac.compare_digest(v1, expected) for v1 in v1s)
```

## Rotation

**Rotate** issues a new secret and keeps the old one valid for an overlap window; requests carry `v1=<new>,v1=<old>`. Deploy the new secret at your leisure inside the window; a correct verifier passes throughout.

## Also check

- Reject requests older than ~5 minutes (`t`).
- Optionally allowlist Talkif's egress addresses — but the signature, not the source IP, is the proof. Addresses: https://docs.talkif.ai/reference/network-and-ip-allowlists

Docs: https://docs.talkif.ai/integrate/webhooks
