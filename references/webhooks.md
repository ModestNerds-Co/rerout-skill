# Rerout webhooks: events, delivery, and signature verification

## Event types

Subscribe an endpoint to any of these (`events` array on create):

```
link.created        link.updated        link.deleted        link.clicked
domain.added        domain.verified     domain.failed       domain.removed
api_key.created     api_key.revoked
qr.generated        qr.regenerated      qr.scanned
qr_branding.updated qr_branding.reset
qr_preset.created   qr_preset.applied   qr_preset.deleted
project.created     project.updated
```

`GET /v1/projects/me/webhooks` also returns the live `event_types` list, so you
can render choices dynamically instead of hardcoding.

## Delivery semantics

- Rerout sends a signed JSON `POST` to your endpoint URL when a subscribed event
  fires. Click/scan events are dispatched fire-and-forget after the redirect, so
  your endpoint latency never affects redirect latency.
- **Redirects are not followed.** A `2xx` response marks the delivery succeeded;
  anything else retries with backoff until the attempt budget is exhausted.
- Respond `2xx` quickly and do heavy work asynchronously.

## Signature verification (do this on every delivery)

Header: `X-Rerout-Signature: t={unix_seconds},v1={hex_hmac}`.

Algorithm:
1. Parse `t` and `v1` from the header.
2. Reject if `t` is older than your tolerance (recommended **300s**) — replay
   protection.
3. Compute `HMAC-SHA256` over the exact string `"{t}.{raw_request_body}"` keyed
   by the endpoint's `signing_secret` (`whsec_…`).
4. Compare your hex digest to `v1` in **constant time**. Reject on mismatch.

Critical: verify against the **raw, unmodified request body**. If you parse JSON
and re-serialize before hashing, byte differences will break verification.

### SDK verifier (preferred)
Each SDK ships a verifier so you don't reimplement this:

```ts
// TypeScript
import { verifyReroutSignature } from '@rerout/sdk'
const ok = verifyReroutSignature({
  rawBody, // string, exactly as received
  signatureHeader: req.headers['x-rerout-signature'] as string,
  secret: process.env.REROUT_WEBHOOK_SECRET!,
})
```

Equivalents: Python `verify_rerout_signature` / `rerout.webhooks`, Ruby
`Rerout::Webhooks`, PHP `SignatureVerifier`, Go `VerifySignature`, Java/Kotlin
`verifyReroutSignature`, .NET `WebhookSigning`, Dart `ReroutWebhookSignature`,
Rust `webhooks` module. Framework adapters (Express-less, ASP.NET, Spring,
Laravel, Django, Rails) wrap this into middleware/controllers.

### Manual verification (no SDK)
```python
import hashlib, hmac, time

def verify(raw_body: bytes, header: str, secret: str, tolerance=300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t, v1 = parts.get("t"), parts.get("v1")
    if not t or not v1:
        return False
    if abs(time.time() - int(t)) > tolerance:
        return False
    mac = hmac.new(secret.encode(), f"{t}.".encode() + raw_body, hashlib.sha256)
    return hmac.compare_digest(mac.hexdigest(), v1)
```

```js
// Node, no SDK
import { createHmac, timingSafeEqual } from 'node:crypto'
function verify(rawBody, header, secret, tolerance = 300) {
  const parts = Object.fromEntries(header.split(',').map((p) => p.split('=')))
  const { t, v1 } = parts
  if (!t || !v1) return false
  if (Math.abs(Date.now() / 1000 - Number(t)) > tolerance) return false
  const expected = createHmac('sha256', secret).update(`${t}.${rawBody}`).digest('hex')
  const a = Buffer.from(expected), b = Buffer.from(v1)
  return a.length === b.length && timingSafeEqual(a, b)
}
```

## Lifecycle

- **Create** via API (`POST /v1/projects/me/webhooks`) → capture `signing_secret`
  once.
- **Rotate secret / send test / inspect deliveries / retry** → dashboard only.
- **Delete** via API (`DELETE /v1/projects/me/webhooks/{id}`) → abandons pending
  deliveries.
