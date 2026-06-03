# Example: receive & verify a webhook (Node / Express)

Verify the signature against the **raw body** before trusting the payload.

```ts
import express from 'express'
import { verifyReroutSignature } from '@rerout/sdk'

const app = express()

// Capture the raw body — required for signature verification.
app.post(
  '/hooks/rerout',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const raw = req.body.toString('utf8')
    const ok = verifyReroutSignature({
      rawBody: raw,
      signatureHeader: req.header('x-rerout-signature') ?? '',
      secret: process.env.REROUT_WEBHOOK_SECRET!,
    })
    if (!ok) return res.status(400).end('bad signature')

    const event = JSON.parse(raw)
    switch (event.type) {
      case 'link.clicked':
        // handle click (event.data has country, device, browser, referrer…)
        break
      case 'link.created':
        break
      default:
        break
    }
    res.status(200).end() // ack fast; do heavy work async
  },
)

app.listen(3000)
```

## Python / Flask (no SDK)

```python
import os, hashlib, hmac, time, json
from flask import Flask, request, abort

app = Flask(__name__)
SECRET = os.environ["REROUT_WEBHOOK_SECRET"]

def verify(raw: bytes, header: str, tolerance=300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t, v1 = parts.get("t"), parts.get("v1")
    if not t or not v1 or abs(time.time() - int(t)) > tolerance:
        return False
    mac = hmac.new(SECRET.encode(), f"{t}.".encode() + raw, hashlib.sha256)
    return hmac.compare_digest(mac.hexdigest(), v1)

@app.post("/hooks/rerout")
def hook():
    raw = request.get_data()  # raw bytes, unparsed
    if not verify(raw, request.headers.get("X-Rerout-Signature", "")):
        abort(400)
    event = json.loads(raw)
    # dispatch on event["type"]
    return "", 200
```
