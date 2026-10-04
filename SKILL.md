---
name: sendrix
description: "Send transactional email through the Sendrix gateway/proxy. Use this skill whenever the user mentions sending emails, transactional emails, welcome emails, magic links, password resets, OTP codes, invoices with attachments, email templates, email notifications, email webhooks/delivery events, bounces, spam complaints, suppression lists, sandbox email testing, provider failover (Resend, Postmark, SMTP), Sendrix, the Sendrix API, or configuring SENDRIX_KEY / SENDRIX_BASE_URL. Covers single and batch sending, async queues, templates, attachments, webhook signature verification, error handling and framework integrations."
license: MIT
metadata:
  author: Sendrix
  version: 2.0.0
---

# Sendrix — Transactional Email Gateway & Proxy

Sendrix is a self-hosted **transactional email gateway and proxy**. Your applications send email to Sendrix over a single HTTP API using a project API key; Sendrix handles the rest: provider delivery, retries/queues, delivery logging, suppression lists, templates, sandbox testing, outbound webhooks and provider failover.

Instead of talking to an email provider (Resend, Postmark, SMTP) directly from every app, you point every app at Sendrix. Sendrix centralises credentials, deliverability protection and observability.

```
[ Your App(s) ] --HTTP + Bearer <project key>--> [ SENDRIX ] --provider--> [ Resend (primary) ]
                                                              \--failover--> [ Postmark / SMTP ]
                                                                   |
                                                     PostgreSQL + Redis + Horizon
```

**Under the hood:** Laravel 13 · Livewire 4 · Laravel Horizon (Redis queues) · PostgreSQL. Primary delivery provider is **Resend**; optional failover to **Postmark** or a generic **SMTP** server.

---

## ⚡ Dynamic Updates & Live Feature Discovery

> **CRITICAL FOR AI AGENTS:** Sendrix evolves. Before assuming a capability does not exist, or when asked for a feature not documented here, fetch the live spec:
> - `{SENDRIX_BASE_URL}/skill.md` — canonical, live SKILL.md
> - `{SENDRIX_BASE_URL}/llms.txt` — endpoint summary
> - `{SENDRIX_BASE_URL}/llms-full.txt` — SKILL.md + every reference concatenated
> - `{SENDRIX_BASE_URL}/skill/raw/<file>.md` — a single reference file
>
> To refresh this local skill to the latest version:
> ```bash
> curl -fsSL {SENDRIX_BASE_URL}/skill/install.sh | bash
> ```

---

## 1. Capability Map

| Capability | Endpoint / Mechanism | Reference |
|---|---|---|
| Send one email | `POST /api/v1/send` | [api-reference.md](references/api-reference.md) |
| Send up to 100 emails per request | `POST /api/v1/batch` | [api-reference.md](references/api-reference.md) |
| Async dispatch (Horizon queue) | `async: true` / `Prefer: respond-async` / `X-Sendrix-Async: 1` | [api-reference.md](references/api-reference.md) |
| Check delivery status | `GET /api/v1/emails/{id}` | [api-reference.md](references/api-reference.md) |
| Reusable templates with `{{ variables }}` | `template` + `variables` payload | [templates.md](references/templates.md) |
| Attachments (PDF, docs, …) | `attachments[]` (base64, ≤10 MB total) | [api-reference.md](references/api-reference.md) |
| Sandbox / test emails | `sndx_test_*` key or `sandbox: true` | [sandbox.md](references/sandbox.md) |
| Suppression list (bounces, complaints) | automatic + dashboard | [suppressions.md](references/suppressions.md) |
| Outbound webhooks (signed HMAC) | project webhook config | [webhooks.md](references/webhooks.md) |
| Provider failover | Resend → Postmark / SMTP | [providers-failover.md](references/providers-failover.md) |
| Health check | `GET /api/v1/health` | [api-reference.md](references/api-reference.md) |
| Self-hosting / infra | Docker + Coolify | [deployment.md](references/deployment.md) |

Framework-specific copy-paste integrations: [Laravel](references/laravel.md) · [Node.js / TypeScript](references/nodejs.md) · [Python](references/python.md).

---

## 2. Core Configuration & Authentication

### Environment variables (client application)

```dotenv
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

- `SENDRIX_BASE_URL` — base URL of the Sendrix instance (production default `https://sendrix.alejandrocabeza.dev`; local dev `http://localhost:8000`).
- `SENDRIX_KEY` — a **project API key** generated in the Sendrix dashboard → *Projects*. Keys are scoped to a single project and shown only once at creation.

### API key types

| Key prefix | Behaviour |
|---|---|
| `sndx_live_` | Live key. Emails are delivered through the project's configured provider. |
| `sndx_test_` | Test key. **Every request is forced into sandbox mode** — emails are captured in the Sandbox Inbox and never sent to a real provider. |

> API keys are hashed (SHA-256) at rest. There is no `X-Project-ID` header and no admin/JWT API in v1 — authentication is the key itself, which resolves the project.

### HTTP headers

Every request must include:

```http
Authorization: Bearer <SENDRIX_KEY>
Content-Type: application/json
Accept: application/json
```

Alternatively, send `X-Sendrix-Key: <SENDRIX_KEY>` instead of the `Authorization` header. If both are present, `Authorization: Bearer` wins.

---

## 3. API Endpoints (v1)

Base prefix: `/api/v1`.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `POST` | `/api/v1/send` | API key | Send a single email |
| `POST` | `/api/v1/batch` | API key | Send 1–100 emails in one request |
| `GET` | `/api/v1/emails/{id}` | API key | Fetch the status/log of an email you sent |
| `GET` | `/api/v1/health` | none | Liveness probe → `{"status":"ok","version":"v1"}` |
| `POST` | `/api/v1/webhooks/resend` | provider signature | Inbound provider events (Sendrix-internal; do not call from clients) |

---

## 4. Sending a Single Email — `POST /api/v1/send`

### Payload

```json
{
  "to": "customer@example.com",
  "subject": "¡Bienvenido a nuestra plataforma!",
  "html": "<h1>Bienvenido</h1><p>Gracias por unirte.</p>",
  "text": "Bienvenido. Gracias por unirte.",
  "from_name": "Mi App",
  "reply_to": "soporte@miapp.com",
  "cc": ["copia@ejemplo.com"],
  "bcc": ["auditoria@ejemplo.com"],
  "attachments": [
    {
      "filename": "factura-1042.pdf",
      "content": "<base64>",
      "content_type": "application/pdf"
    }
  ],
  "async": false,
  "sandbox": false
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `to` | string | yes | Single recipient, valid email (`rfc` + `filter`). |
| `subject` | string | yes\* | Max 998 chars. \*Not required if `template` is provided. |
| `html` | string | yes\* | HTML body. \*Not required if `template` is provided. |
| `text` | string | no | Plain-text alternative. |
| `template` | string | no | Template slug (or numeric id). If set, renders `subject`/`html`/`text`. |
| `variables` | object | no | Interpolation map for `{{ variable }}` in the template. |
| `from_name` | string | no | Overrides the project's sender display name (max 255). |
| `reply_to` | string | no | Overrides the project's reply-to. |
| `cc` | string[] | no | Array of valid emails. |
| `bcc` | string[] | no | Array of valid emails. |
| `attachments` | object[] | no | Max 10 items. Each: `filename` (required), `content` (required, base64), `content_type` (optional). Total decoded size ≤ **10 MB**. |
| `async` | boolean | no | Queue the send and return `202`. |
| `sandbox` | boolean | no | Capture in the Sandbox Inbox instead of sending. |

> The effective `From` is always `"<sender_name>" <sender_alias@verified_domain>` — the alias/domain come from the project + connected provider. `from_name` only changes the display name. You cannot spoof an arbitrary From address.

### The `From` address model

Each project has a `sender_name`, a `sender_alias` and an optional `reply_to`. The verified sending domain comes from the connected provider. So a request with `from_name: "Mi App"` on project alias `waitlist` + domain `creator.dev` sends from:

```
Mi App <waitlist@creator.dev>
```

### Responses

**`200 OK` — sent (or sandbox-captured):**

```json
{
  "id": "01923e77-7777-7000-8000-000000000077",
  "project_id": 1,
  "status": "sent",
  "resend_id": "resend-msg-abc123",
  "from": "Mi App <waitlist@creator.dev>",
  "to": "customer@example.com",
  "subject": "¡Bienvenido!",
  "sent_at": "2026-10-01T15:00:00+00:00"
}
```

- `id` — Sendrix **log id** (UUID). Use it with `GET /api/v1/emails/{id}`. This is the id you persist.
- `resend_id` — the downstream provider message id (may be `null` in sandbox/queued).
- `status` — `sent` (delivered to provider), `sandbox` (captured) or `queued` (async).

**`202 Accepted` — queued asynchronously:** same shape with `"status": "queued"` and `sent_at: null`.

### Error responses

| HTTP | `error` | Meaning | What the client should do |
|---|---|---|---|
| `401` | `Unauthorized` | Missing/invalid/revoked API key. | Permanent. Fix `SENDRIX_KEY`. |
| `422` | *(validation)* | Laravel validation error (invalid email, subject > 998, bad base64, attachments > 10 MB…). | Permanent. Inspect `errors`. |
| `404` | `TemplateNotFound` | `template` slug/id does not exist for this project. | Permanent. |
| `422` | `TemplateInactive` | Template exists but `is_active = false`. | Permanent. |
| `422` | `RecipientSuppressed` | Recipient is on the suppression list. Body also has `reason`. | Permanent for that recipient. |
| `429` | `QuotaExceeded` | Project exceeded its **daily** send limit. | Retry after the quota window resets (next day), or raise the project limit. |
| `400` | `ProviderNotConfigured` | No connected provider / missing verified domain. | Fix provider settings in the dashboard. |
| `502` | `ProviderError` | Upstream provider rejected the send. | Retry with backoff / inspect `message`. |

```json
// 422 RecipientSuppressed
{ "error": "RecipientSuppressed", "message": "Recipient is suppressed.", "reason": "hard_bounce" }
```

---

## 5. Sending Batches — `POST /api/v1/batch`

Send 1–100 emails in a single HTTP call. Each item is processed independently; the response reports per-item results.

```json
{
  "batch": [
    {
      "to": "user1@example.com",
      "subject": "Notificación 1",
      "html": "<p>Contenido 1</p>"
    },
    {
      "to": "user2@example.com",
      "template": "welcome",
      "variables": { "name": "Ana", "company": "Acme" }
    }
  ],
  "async": false
}
```

- The array **must** be named `batch` (not `emails`).
- Each item accepts the same fields as `/send` (`to`, `subject`, `html`, `text`, `template`, `variables`, `reply_to`, `from_name`, `cc`, `bcc`, `sandbox`).
- Top-level `async: true` queues **every** item. Per-item `sandbox` is also supported.

### Response body

```json
{
  "total": 2,
  "successful": 1,
  "failed": 1,
  "data": [
    { "index": 0, "to": "user1@example.com", "status": "sent", "id": "…", "provider_message_id": "…" },
    { "index": 1, "to": "user2@example.com", "status": "suppressed", "error": "RecipientSuppressed", "message": "…", "reason": "hard_bounce" }
  ]
}
```

| HTTP | When |
|---|---|
| `200` | All items succeeded. |
| `207` Multi-Status | Mixed: some succeeded, some failed. |
| `422` | Every item failed (or the request payload itself is invalid → `{"error":"ValidationFailed","errors":{…}}`). |

Per-item `status` values: `sent`, `queued`, `suppressed`, `quota_exceeded`, `provider_error`, `error` (template), `failed`.

---

## 6. Checking Delivery Status — `GET /api/v1/emails/{id}`

Pass the Sendrix log `id` returned by `/send` or `/batch`.

```json
{
  "id": "01923e77-7777-7000-8000-000000000077",
  "project_id": 1,
  "status": "delivered",
  "recipient": "customer@example.com",
  "subject": "¡Bienvenido!",
  "from_email": "waitlist@creator.dev",
  "from_name": "Mi App",
  "provider_message_id": "resend-msg-abc123",
  "sent_at": "2026-10-01T15:00:00+00:00",
  "created_at": "2026-10-01T15:00:00+00:00",
  "error_message": null,
  "metadata": {}
}
```

- Status lifecycle: `queued` → `sending` → `sent` → `delivered`; terminal failures: `failed`, `bounced`, `complained`. Sandbox captures use `sandbox`.
- `404 NotFound` if the id does not exist **or belongs to a different project** (keys are project-scoped).
- `metadata` carries extra info: `cc`/`bcc`, attachment descriptors, `open_count`/`click_count`, `events` audit trail, and `failover_used`/`failover_provider`/`primary_error` when failover kicked in.

---

## 7. Suppression & Deliverability Protection

Sendrix blocks sends to addresses that hard-bounced or complained, protecting your domain reputation.

- Reasons: `hard_bounce`, `spam_complaint` (global to the user), `manual`, `unsubscribe`.
- Suppressed recipients are rejected **synchronously with `422 RecipientSuppressed`** — including for `async` requests (checked *before* queueing).
- Bounces and spam complaints received from the provider automatically add entries.
- `spam_complaint` suppressions apply across **all** of the user's projects; other reasons are project-scoped.
- Sandbox mode bypasses suppression checks.

See [suppressions.md](references/suppressions.md).

---

## 8. Outbound Webhooks (Delivery Events)

Configure a webhook URL per project (dashboard → project → webhooks) and subscribe to events. Sendrix POSTs a signed JSON payload:

```http
X-Sendrix-Signature: t=1727447400,v1=9f86d081884c7d659a2feaa0c55ad015a3bf...
X-Sendrix-Event: email.delivered
X-Sendrix-Delivery-Id: 01923b78-....
X-Sendrix-Timestamp: 1727447400
```

```json
{
  "id": "01923b78-....",
  "event": "email.delivered",
  "created_at": "2026-10-01T15:00:05+00:00",
  "data": {
    "email_id": "01923e77-....",
    "provider_message_id": "resend-msg-abc123",
    "recipient": "customer@example.com",
    "subject": "¡Bienvenido!",
    "from_name": "Mi App",
    "from_email": "waitlist@creator.dev",
    "status": "delivered",
    "error_message": null,
    "metadata": {}
  }
}
```

Supported events: `email.delivered`, `email.bounced`, `email.complained`, `email.opened`, `email.clicked`, `email.failed`. A subscription of `*` receives everything. `ping` is used for the dashboard "send test" action.

### Signature verification (HMAC-SHA256)

```
signed_payload     = "{timestamp}.{raw_request_body}"
expected_signature = hex(hmac_sha256(signed_payload, WEBHOOK_SECRET))
```

Compare against the `v1=` value in `X-Sendrix-Signature`, using a constant-time comparison and a timestamp tolerance (~5 minutes) to prevent replay attacks. **Always verify against the raw body**, not a re-serialised object.

Delivery retries: 3 attempts with backoff `10s → 60s → 300s` on the `webhooks` queue. `4xx` responses fail immediately; `5xx`/timeouts are retried. Full PHP/Node/Python verification code: [webhooks.md](references/webhooks.md).

---

## 9. Providers & Failover

- **Primary:** Resend — connect the account's API key and pick a verified sending domain.
- **Failover:** Postmark (`server_token`) or SMTP (`host`, `port`, `encryption`, `username`, `password`).
- If the primary provider throws, Sendrix automatically retries through the configured failover provider, and records `failover_used`, `failover_provider` and `primary_error` in the email log metadata.
- Per-user settings; a project uses its owner's provider configuration.

See [providers-failover.md](references/providers-failover.md).

---

## 10. Queues, Quotas & Retries

- **Daily quota:** each project has a `daily_limit` (default **100**). When exceeded, `/send` and `/batch` return `429 QuotaExceeded`. Counts reset daily.
- **Async:** `async` sends are pushed to the Horizon `emails` queue and return `202` immediately. The background job retries up to 5 times with backoff `10s, 30s, 60s, 120s` (timeout 60s).
- **Provider rate limits:** when the provider returns HTTP `429`, the queued job releases itself back to the queue after **65 seconds** (never busy-loop the provider).
- **Idempotency for status:** pair every async send with the returned `id` and poll `GET /api/v1/emails/{id}` — do not assume delivery.

> There is no fixed "5 requests/minute" API limit in the server. The only API-level limit is the **per-project daily quota**. Still, queue non-urgent sends and back off on `429`/`5xx`.

---

## 11. Framework Integration Guides

- [Laravel](references/laravel.md) — custom `SendrixTransport` mailer, Mailables, Horizon `emails` queue, attachments, Pest tests.
- [Node.js / TypeScript](references/nodejs.md) — typed `fetch` client with retries and exponential backoff; Next.js / Express recipes.
- [Python](references/python.md) — `requests`/`httpx` client with retry wrapper.
- [Webhooks](references/webhooks.md) — HMAC-SHA256 verification in PHP, Node and Python.
- [Templates](references/templates.md) — template model, variables and the visual block builder.
- [Sandbox](references/sandbox.md) — test keys, sandbox mode and captured emails.
- [Suppressions](references/suppressions.md) — bounce/complaint handling and suppression management.
- [Providers & Failover](references/providers-failover.md) — connecting Resend, Postmark and SMTP.
- [Deployment](references/deployment.md) — Docker/Coolify self-hosting, services and environment variables.
- [API Reference](references/api-reference.md) — exhaustive schemas, response codes and curl examples.
