# Example: create a link + a webhook

End-to-end with curl (no SDK), then the TypeScript SDK equivalent.

## curl

```bash
export REROUT_API_KEY=rrk_xxxxxxxxxxxxxxxx

# 1. Create a short link
curl -sS -X POST https://api.rerout.co/v1/links \
  -H "Authorization: Bearer $REROUT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"target_url":"https://example.com/summer-sale"}'
# => { "code":"abc123","short_url":"https://rerout.co/abc123", ... }

# 2. Subscribe a webhook to link + click events
curl -sS -X POST https://api.rerout.co/v1/projects/me/webhooks \
  -H "Authorization: Bearer $REROUT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Order events",
        "url": "https://example.com/hooks/rerout",
        "events": ["link.created", "link.clicked"]
      }'
# => { "endpoint": { "id":"wh_…", ... }, "signing_secret":"whsec_…" }
#    Store signing_secret now — it is never shown again.

# 3. List endpoints (and supported event types)
curl -sS https://api.rerout.co/v1/projects/me/webhooks \
  -H "Authorization: Bearer $REROUT_API_KEY"

# 4. Delete an endpoint
curl -sS -X DELETE https://api.rerout.co/v1/projects/me/webhooks/wh_… \
  -H "Authorization: Bearer $REROUT_API_KEY"
```

## TypeScript SDK

```ts
import { Rerout } from '@rerout/sdk'

const rerout = new Rerout({ apiKey: process.env.REROUT_API_KEY! })

const link = await rerout.links.create({
  target_url: 'https://example.com/summer-sale',
})
console.log(link.short_url)

const { endpoint, signing_secret } = await rerout.webhooks.create({
  name: 'Order events',
  url: 'https://example.com/hooks/rerout',
  events: ['link.created', 'link.clicked'],
})
// Persist signing_secret to your secret store — shown once.
await saveSecret('REROUT_WEBHOOK_SECRET', signing_secret)
```
