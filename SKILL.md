---
name: rerout
description: >-
  Integrate Rerout — branded short links, QR codes, and webhooks — into an app.
  Use when the user wants to create or manage short links, render/customize QR
  codes, read link analytics, or set up and verify Rerout webhooks. Triggers on
  "rerout", "rerout.co", an "rrk_" API key, "@rerout/sdk", or branded/short link
  + QR + webhook tasks against api.rerout.co.
---

# Rerout integration

Rerout is branded link infrastructure: short links on custom domains, QR codes,
click analytics, and signed webhooks. The HTTP API is at `https://api.rerout.co`,
authenticated with a project API key (`rrk_…`) sent as a Bearer token. Official
SDKs exist for 10 languages; prefer the SDK for the project's language and fall
back to raw REST otherwise.

## Decide the approach first

1. **Detect the language/runtime** of the project you're integrating into.
2. **Prefer the matching SDK** (see `references/sdk-quickstart.md`). It handles
   auth, retries, error types, and webhook signature verification for you.
3. **Fall back to REST** (`references/rest-api.md`) for languages without an SDK
   or for quick scripts/curl.

Never hardcode the API key. Read it from an environment variable
(`REROUT_API_KEY`) or the project's secret store.

## Authentication

Every `/v1/*` request needs `Authorization: Bearer <rrk_… key>`. Keys are minted
in the project dashboard and are scoped to **one project** — the API resolves the
project from the key, which is why account-scoped routes use `/v1/projects/me/…`
(no project id in the path). API keys are for backend use; they do **not** sign
into the web app.

```bash
curl https://api.rerout.co/v1/projects/me \
  -H "Authorization: Bearer $REROUT_API_KEY"
```

## Core tasks

These are the workflows agents are usually asked for. Each links to depth.

### Create a short link
Minimum input is a `target_url`; you get back a `code` and `short_url`. Optional:
custom `domain_hostname` (must be a verified custom domain), custom `code`,
`expires_at`, and SEO/social-preview overrides. See `references/rest-api.md` →
Links.

```bash
curl -X POST https://api.rerout.co/v1/links \
  -H "Authorization: Bearer $REROUT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"target_url":"https://example.com/sale"}'
```

### Render a QR code
Every link has a QR endpoint: `GET /v1/links/{code}/qr` (SVG), tunable via
`size`, `margin`, `ecc`, `domain`, `refresh`. SDKs expose both a pure URL builder
and an authenticated fetch. See `references/rest-api.md` → QR.

### Read analytics
Per-link: `GET /v1/links/{code}/stats?days=30`. Whole project:
`GET /v1/projects/me/stats?days=30`. Returns totals plus country/referrer/
device/browser breakdowns.

### Manage webhook endpoints
Create/list/delete endpoints that receive event deliveries. **API-key
authenticated** under `/v1/projects/me/webhooks`. The `signing_secret`
(`whsec_…`) is returned **once** on create — persist it immediately; it verifies
deliveries and is never shown again. See `references/webhooks.md`.

```bash
curl -X POST https://api.rerout.co/v1/projects/me/webhooks \
  -H "Authorization: Bearer $REROUT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"Order events","url":"https://example.com/hooks/rerout","events":["link.created","link.clicked"]}'
```

Update, rotate-secret, test-send, and delivery inspection/retry are
dashboard-only — they are not exposed to API keys.

### Receive & verify webhooks
Rerout POSTs signed JSON to your endpoint. **Always verify the signature before
trusting the payload.** The `X-Rerout-Signature` header is `t={unix},v1={hex}`;
recompute `HMAC-SHA256("{t}.{rawBody}", signing_secret)` and compare in constant
time, rejecting timestamps older than ~300s. SDKs ship a verifier
(`verifyReroutSignature` and equivalents). Full algorithm + per-language snippets
in `references/webhooks.md`. **Verify against the raw, unparsed request body** —
re-serialized JSON will not match.

## Reference files

- `references/sdk-quickstart.md` — install + client construction + webhook
  management usage for all 10 SDK languages.
- `references/rest-api.md` — the REST contract: endpoints, request/response
  shapes, error format, rate limits.
- `references/webhooks.md` — event types, delivery semantics, and signature
  verification (algorithm + code).
- `examples/` — runnable end-to-end snippets (create link, verify webhook).

## Gotchas (these are what agents get wrong)

- **Key placement.** `Authorization: Bearer rrk_…`, not a query param, not a
  custom header.
- **`/v1/projects/me/…`** for account-scoped routes — there is no project id in
  the path; it comes from the key.
- **Signing secret is shown once** on webhook create. If lost, rotate it in the
  dashboard.
- **Verify webhooks on the raw body**, within the timestamp tolerance, in
  constant time. Don't parse-then-reserialize before verifying.
- **Custom `code`/`domain_hostname`** only work with a verified custom domain on
  the project.
- **Errors** are JSON `{ "code": "...", "message": "..." }` with a matching HTTP
  status (401 unauthorized, 429 rate_limited, etc.). Surface `code`, not just the
  status.
