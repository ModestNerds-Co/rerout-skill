# Rerout REST API reference

Base URL: `https://api.rerout.co`. All `/v1/*` routes below require
`Authorization: Bearer <rrk_… key>`. Request/response bodies are JSON. The full
public docs live at <https://rerout.co/docs>.

> Scope: this file documents the **API-key surface** — the routes a project API
> key (`rrk_…`) can call. The web app uses many more `/v1/projects/:id/…`
> session routes (workspaces, domains, QR branding/presets, Slack, audit, etc.);
> those authenticate with a login session, **not** an API key, and are out of
> scope here.

## Authentication & errors

- Header: `Authorization: Bearer rrk_xxxxxxxx`. The key resolves to exactly one
  project; account-scoped routes therefore use `/v1/projects/me/…` with no id.
- Health check (no auth): `GET /health` → `{ "ok": true, "service": "rerout-api" }`.
- Error shape (non-2xx): `{ "code": "<stable_code>", "message": "<human text>" }`.
  Surface `code`, not just the HTTP status. Codes you may see:
  - `bad_request` (400), `unauthorized` (401), `forbidden` (403),
    `not_found` (404)
  - `code_conflict` (409) — requested custom path already in use
  - `code_collision` (500) — could not allocate a code after retries
  - `conflict` (409), `payload_too_large` (413)
  - `rate_limited` (429) — sends `Retry-After: 60`
  - `service_unavailable` (503), `internal_error` (500)
- Rate limits: link creation, batch creation, and conversion recording are each
  rate-limited per API key; exceeding returns `429 rate_limited` with a
  `Retry-After` header.

## Links

### `POST /v1/links` — create
Request (`target_url` is the only required field):
```json
{
  "target_url": "https://example.com/sale",   // required
  "domain_hostname": "go.brand.com",           // optional, verified custom domain
  "code": "summer-sale",                        // optional, REQUIRES domain_hostname
  "expires_at": 1735689600,                     // optional, unix seconds, must be future
  "seo_title": null,                            // optional social-preview overrides
  "seo_description": null,
  "seo_image_url": null,
  "seo_canonical_url": null,
  "seo_noindex": false,
  "tag_ids": ["tag_…"],                         // optional, attach existing project tags
  "password": "hunter2",                        // optional, password-protect the link
  "max_clicks": 1000,                           // optional, auto-disable after N clicks
  "track_conversions": false,                   // optional, enable conversion attribution
  "routing_rules": [                            // optional, geo/device/browser routing
    { "condition_type": "geo", "condition_op": "eq",
      "condition_value": "US", "target_url": "https://example.com/us" }
  ],
  "ab_variants": [                              // optional, weighted A/B destinations
    { "target_url": "https://example.com/a", "weight": 50 },
    { "target_url": "https://example.com/b", "weight": 50 }
  ]
}
```
Response `201`: a `Link` (see shape below).
Notes: a custom `code` is only accepted alongside a `domain_hostname` — without
one you get `400 bad_request` ("code can only be set when domain_hostname is
provided"). A taken path returns `409 code_conflict`.

### `POST /v1/links/batch` — bulk create
Create up to **500** links in one request.
```json
{
  "links": [
    { "target_url": "https://example.com/a" },
    { "target_url": "https://example.com/b", "code": "promo", "domain_hostname": "go.brand.com" },
    { "target_url": "https://example.com/c", "expires_at": 1735689600 }
  ]
}
```
Each item accepts `target_url` (required), `code`, `expires_at`,
`domain_hostname`. Response `207` (multi-status) — items succeed/fail
independently:
```json
{
  "created": 2,
  "total": 3,
  "results": [
    { "index": 0, "ok": true,  "code": "abc123" },
    { "index": 1, "ok": false, "error": "That custom path is already in use." },
    { "index": 2, "ok": true,  "code": "def456" }
  ]
}
```

### `GET /v1/links?cursor=&limit=&tag_ids=` — list
Query params: `limit` (1–100, default 50), `cursor` (numeric id from the
previous page's `next_cursor`), `tag_ids` (comma-separated, filter by tag).
Response `200`: `{ "links": Link[], "next_cursor": number | null }`.

### `GET /v1/links/{code}` — get one
### `PATCH /v1/links/{code}` — update (only provided fields change)
Accepts any of: `target_url`, `expires_at` (or `null` to clear), `is_active`,
the `seo_*` fields (value or `null` to clear), `tag_ids`, `password` (`null` to
remove protection), `max_clicks` (`null` to clear), `track_conversions`,
`routing_rules` (full replace), `ab_variants` (full replace). At least one field
must be present, else `400 bad_request`.
### `DELETE /v1/links/{code}` — soft-delete → `{ "deleted": true }`
### `GET /v1/links/{code}/stats?days=30` — per-link analytics (see Analytics)

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
  "password_protected": false,
  "max_clicks": null,
  "click_count": 0,
  "track_conversions": false,
  "routing_rules": [
    { "condition_type": "geo", "condition_op": "eq",
      "condition_value": "US", "target_url": "https://example.com/us" }
  ],
  "ab_variants": [
    { "id": 1, "target_url": "https://example.com/a", "weight": 50 }
  ],
  "created_at": 1700000000, "updated_at": 1700000000
}
```
The raw `password` is never returned — only the `password_protected` boolean.

## Conversions

### `POST /v1/conversions` — record a conversion
Attributes a downstream conversion to an earlier click via its click id. Use
when a link has `track_conversions` enabled. **Idempotent** on
`(click_id, event_name)`.
```json
{
  "click_id": "clk_…",        // required, the click id captured at redirect (≤64 chars)
  "event_name": "purchase",    // required, [A-Za-z0-9._:-]{1,64}
  "value_cents": 4999,         // optional, 0 – 1,000,000,000,000
  "currency": "USD"            // optional, ISO-4217 (USD,EUR,GBP,CAD,AUD,JPY,…)
}
```
Response: `201` first time → `{ "recorded": true, "duplicate": false }`; a
repeat of the same `(click_id, event_name)` returns `200` with
`"duplicate": true`. An unknown `click_id` is rejected (must belong to this
project).

## QR codes

`GET /v1/links/{code}/qr` → SVG (`Content-Type: image/svg+xml`). Send the Bearer
header. Query params:
- `size` — module size in px, 1–32, default **8**
- `margin` — quiet-zone modules, 0–16, default **4**
- `ecc` — error-correction level `L`|`M`|`Q`|`H`, default **H**
- `domain` — encode a specific verified domain's short URL
- `refresh` — any non-empty value forces regeneration (emits `qr.regenerated`
  instead of `qr.generated`)

## Projects

- `GET /v1/projects/me` → `{ "id", "name", "slug" }` (the project owning the key).
- `GET /v1/projects/me/stats?days=30` → aggregate stats (`days` 1–90, default 30):
  ```json
  {
    "days": 30, "total_clicks": 0, "qr_scans": 0,
    "daily": [{ "day": 1700000000, "clicks": 0, "qr_scans": 0 }],
    "countries": [{ "value": "US", "clicks": 0 }],
    "referrers": [], "devices": [], "browsers": [], "top_codes": []
  }
  ```
  Every breakdown entry is `{ "value": string, "clicks": number }`.

## Analytics

- **Per-link** — `GET /v1/links/{code}/stats?days=30` (`days` 1–90, default 30):
  ```json
  {
    "code": "abc123", "days": 30,
    "total_clicks": 0, "qr_scans": 0,
    "countries": [{ "value": "US", "clicks": 0 }],
    "referrers": [{ "value": "google.com", "clicks": 0 }]
  }
  ```
  Per-link stats carry **countries and referrers only** — no daily series,
  device, or browser breakdowns. Those are project-level (`/v1/projects/me/stats`).

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
only** (web-session routes under `/v1/projects/:id/webhooks/…`) — not exposed to
API keys.
