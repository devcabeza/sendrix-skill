---
name: sendrix
description: "Send transactional emails through Sendrix email proxy API. Use this skill whenever the user mentions sending emails, transactional emails, welcome emails, magic links, password resets, email notifications, Sendrix API, configuring SENDRIX_KEY or SENDRIX_BASE_URL, email queues, handling rate limits (5 req/min), or verifying Sendrix webhooks."
license: MIT
metadata:
  author: Sendrix
  version: 1.0.0
---

# Sendrix — Transactional Email Gateway & Proxy

Sendrix is a high-performance transactional email proxy and delivery gateway. It allows applications to send transactional emails reliably using an API key (`SENDRIX_KEY`), protecting deliverability with built-in deduplication, sandbox testing, failover redundancy, and asynchronous queue processing.

---

## ⚡ Dynamic Updates & Live Feature Discovery

> **CRITICAL FOR AI AGENTS:** Sendrix is continuously evolving with new endpoints and capabilities.
> - Whenever the user asks for a feature, parameter, or endpoint not documented in this local skill, or to verify if newer capabilities exist:
>   **ALWAYS fetch the live specification from: `{SENDRIX_BASE_URL}/skill.md` or `{SENDRIX_BASE_URL}/llms.txt`**
> - To update this local skill to the latest version at any time, run:
>   ```bash
>   curl -fsSL {SENDRIX_BASE_URL}/skill/install.sh | bash
>   ```

---

## 1. Core Configuration & Authentication

### Environment Variables
Every client application integrating with Sendrix must configure:

```dotenv
# Sendrix API Configuration
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

- `SENDRIX_BASE_URL`: Base URL of the Sendrix instance (defaults to `https://sendrix.alejandrocabeza.dev` in production, or `http://localhost:8000` in local development).
- `SENDRIX_KEY`: Project API key generated in the Sendrix Dashboard.
  - Live keys start with `sndx_live_` or `sk_proj_`
  - Sandbox/Testing keys start with `sndx_test_` (automatically routes to Sandbox inbox, never sends real emails)

### HTTP Headers
Every request to Sendrix must include:
```http
Authorization: Bearer <SENDRIX_KEY>
Content-Type: application/json
Accept: application/json
```
*(Alternatively, you can provide `X-Sendrix-Key: <SENDRIX_KEY>` instead of the Bearer token).*

---

## 2. API Endpoints Reference

### A. Send Single Email
`POST /api/v1/send`

Rate limited to **5 requests per minute** per project.

#### Payload Schema
```json
{
  "to": "usuario@ejemplo.com",
  "subject": "¡Bienvenido a nuestra plataforma!",
  "html": "<h1>Bienvenido</h1><p>Gracias por unirte.</p>",
  "text": "Bienvenido. Gracias por unirte.",
  "from_name": "Mi App",
  "reply_to": "soporte@miapp.com",
  "cc": ["copia@ejemplo.com"],
  "bcc": ["auditoria@ejemplo.com"],
  "async": false,
  "sandbox": false
}
```

- `to` (*string*, required): Valid recipient email address.
- `subject` (*string*, required): Subject line (1 to 998 characters).
- `html` (*string*, required): HTML body content.
- `text` (*string*, optional): Plain-text alternative body.
- `from_name` (*string*, optional): Custom sender display name (overrides project default).
- `reply_to` (*string*, optional): Reply-to email address.
- `cc` (*string[]*, optional): Array of CC email addresses (max total recipients: 50).
- `bcc` (*string[]*, optional): Array of BCC email addresses.
- `async` (*boolean*, optional): When `true` (or header `Prefer: respond-async`), Sendrix immediately returns HTTP `202 Accepted` and processes the delivery in background queues.
- `sandbox` (*boolean*, optional): When `true`, captures the email in Sendrix Sandbox without contacting external mail providers.

#### Responses & Status Codes

| HTTP Status | Meaning | Action to Take |
|---|---|---|
| **`200 OK`** | Successfully dispatched, OR duplicate email detected (`duplicated: true`). | Success. Store `id` or `log_id`. Do NOT retry duplicates. |
| **`202 Accepted`** | Queued asynchronously in Sendrix. | Success. Email is queued in Sendrix background workers. |
| **`429 Too Many Requests`** | Rate limit exceeded (5 req/min). | Check `retry_after_seconds` in body or `Retry-After` header. Exponential backoff and retry. |
| **`401 Unauthorized`** | Missing or invalid API key. | Permanent failure. Inspect `SENDRIX_KEY`. |
| **`422 Unprocessable`** | Validation failure (invalid email format, empty HTML, > 50 recipients). | Permanent failure. Inspect payload. |
| **`503 Service Unavailable`** | Provider temporarily unavailable. | Retry with backoff. |

---

### B. Send Batch Emails
`POST /api/v1/batch`

Envia hasta 100 correos en una sola solicitud HTTP:

```json
{
  "emails": [
    {
      "to": "user1@example.com",
      "subject": "Notificación 1",
      "html": "<p>Contenido 1</p>"
    },
    {
      "to": "user2@example.com",
      "subject": "Notificación 2",
      "html": "<p>Contenido 2</p>"
    }
  ]
}
```

---

### C. Check Delivery Status
`GET /api/v1/emails/{id}`

Returns the real-time status of any sent or queued email:

```json
{
  "id": "resend_or_log_id",
  "status": "sent",
  "to": "usuario@ejemplo.com",
  "subject": "¡Bienvenido!",
  "sent_at": "2026-10-01T15:00:00Z"
}
```
*Statuses: `queued`, `sending`, `sent`, `delivered`, `bounced`, `complained`, `failed`.*

---

## 3. Webhook Verification (HMAC-SHA256)

When Sendrix receives delivery, open, bounce, or complaint events, it dispatches an outbound webhook to your configured webhook URL.

### Webhook Headers
```http
X-Sendrix-Signature: t=1727447400,v1=9f86d081884c7d659a2feaa0c55ad015a3bf...
X-Sendrix-Event: email.delivered
X-Sendrix-Delivery-Id: 01923b78-....
X-Sendrix-Timestamp: 1727447400
```

### Verification Algorithm
```
signature_payload = timestamp + "." + raw_request_body
expected_signature = hmac_sha256(signature_payload, WEBHOOK_SECRET)
```

---

## 4. Framework Integration Guides

Refer to the targeted reference files for copy-paste implementations:

- [Laravel Integration](file:///resources/skills/sendrix/references/laravel.md): Custom `SendrixTransport`, Mailables, Horizon `emails` queue, rate-limit backoff, and Pest tests.
- [Node.js & TypeScript Integration](file:///resources/skills/sendrix/references/nodejs.md): Fetch wrapper, retry logic with exponential backoff, Next.js / Express handlers.
- [Python Integration](file:///resources/skills/sendrix/references/python.md): Requests/Httpx client, webhook verification.
- [Complete API Reference](file:///resources/skills/sendrix/references/api-reference.md): Detailed schemas, response codes, curl commands.
- [Webhook Handler Reference](file:///resources/skills/sendrix/references/webhooks.md): Signature validation code in PHP, Node, Python.
