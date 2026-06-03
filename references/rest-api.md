# Rerout REST API reference

Base URL: `https://api.rerout.co`. All `/v1/*` routes require
`Authorization: Bearer <rrk_… key>`. Request/response bodies are JSON. The full
public docs live at <https://rerout.co/docs>.

## Authentication & errors

- Header: `Authorization: Bearer rrk_xxxxxxxx`.
- Health check (no auth): `GET /health` → `{ "ok": true, "service": "rerout-api" }`.
- Error shape (non-2xx): `{ "code": "<stable_code>", "message": "<human text>" }`.
  Common codes: `unauthorized` (401), `forbidden` (403), `not_found` (404),
  `bad_request` (400), `rate_limited` (429), `server_error` (5xx).
- Rate limits: API writes are limited per API key (~1000/min on `/v1/links`);
  exceeding returns `429 rate_limited`.

## Links

### `POST /v1/links` — create
Request:
```json
{
  "target_url": "https://example.com/sale",   // required
  "domain_hostname": "go.brand.com",           // optional, verified custom domain
  "code": "summer-sale",                        // optional, requires domain_hostname
  "expires_at": 1735689600,                     // optional, unix seconds (future)
  "seo_title": null,                            // optional social-preview overrides
  "seo_description": null,
  "seo_image_url": null,
  "seo_canonical_url": null,
  "seo_noindex": false
}
```
Response `201`: a `Link` (see shape below).

### `GET /v1/links?cursor=&limit=` — list
Response `200`: `{ "links": Link[], "next_cursor": number | null }`.

### `GET /v1/links/{code}` — get one
### `PATCH /v1/links/{code}` — update (only provided fields change)
Body accepts `target_url`, `expires_at` (or `null` to clear), `is_active`, and
the `seo_*` fields.
### `DELETE /v1/links/{code}` — soft-delete → `{ "deleted": true }`
### `GET /v1/links/{code}/stats?days=30` — per-link analytics

`Link` shape:
```json
{
  "code": "abc123",
  "short_url": "https://rerout.co/abc123",
  "domain_hostname": null,
  "target_url": "https://example.com",
  "project_id": "prj_…",
  "expires_at": null,
  "is_active": true,
  "seo_title": null, "seo_description": null, "seo_image_url": null,
  "seo_canonical_url": null, "seo_noindex": false, "seo_updated_at": null,
  "tags": [{ "id": "tag_…", "name": "campaign", "color": "#ff8800" }],
  "created_at": 1700000000, "updated_at": 1700000000
}
```

## QR codes

`GET /v1/links/{code}/qr` → SVG. Query params: `size` (1–32 px module, default 8),
`margin` (0–16 modules, default 4), `ecc` (`L`|`M`|`Q`|`H`), `domain` (encode a
specific verified domain), `refresh` (cache-bust). Send the Bearer header.

## Projects

- `GET /v1/projects/me` → `{ "id", "name", "slug" }` (the project owning the key).
- `GET /v1/projects/me/stats?days=30` → aggregate stats:
  ```json
  {
    "days": 30, "total_clicks": 0, "qr_scans": 0,
    "daily": [{ "day": 1700000000, "clicks": 0, "qr_scans": 0 }],
    "countries": [{ "value": "US", "clicks": 0 }],
    "referrers": [], "devices": [], "browsers": [], "top_codes": []
  }
  ```

## Webhooks (endpoint management)

API-key authenticated; project resolved from the key (no id in path).

### `POST /v1/projects/me/webhooks` — create
Request:
```json
{
  "name": "Order events",                       // required
  "url": "https://example.com/hooks/rerout",    // required, public https URL
  "events": ["link.created", "link.clicked"],   // required, non-empty
  "is_active": true,                            // optional, default true
  "payload_format": "json"                      // optional: "json" | "slack"
}
```
Response `201`:
```json
{ "endpoint": { /* Webhook */ }, "signing_secret": "whsec_…" }
```
`signing_secret` is returned **once**. `Webhook` shape:
```json
{
  "id": "wh_…", "project_id": "prj_…", "name": "Order events",
  "url": "https://example.com/hooks/rerout",
  "events": ["link.created"], "is_active": true, "payload_format": "json",
  "created_at": 1700000000, "updated_at": 1700000000,
  "last_delivery_at": null, "last_success_at": null, "last_failure_at": null
}
```

### `GET /v1/projects/me/webhooks` — list
Response `200`: `{ "endpoints": Webhook[], "event_types": string[] }`.

### `DELETE /v1/projects/me/webhooks/{endpoint_id}` — delete
Response `200`: `{ "deleted": true }`. Idempotent.

Constraints: endpoint URL must be a public `https` URL — no embedded
credentials, no localhost/private-IP literal, no `rerout.co` host. Slack
incoming-webhook URLs must be configured via the dashboard's Slack integration,
not this API.

Update, rotate-secret, test-send, and delivery inspection/retry are **dashboard
only** — not exposed to API keys.
