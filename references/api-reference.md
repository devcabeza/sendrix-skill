# Sendrix API Reference (v1)

Especificación exhaustiva de los endpoints, esquemas, cabeceras y códigos de respuesta de la API v1 de Sendrix. Refleja el comportamiento real del servidor.

- Prefijo base: `/api/v1`
- Formato: JSON (`Content-Type: application/json`)
- Autenticación: clave de proyecto (`Authorization: Bearer` o `X-Sendrix-Key`)

---

## Autenticación y cabeceras

| Cabecera | Requerida | Descripción |
|---|---|---|
| `Authorization` | Sí\* | `Bearer sndx_live_…` o `Bearer sndx_test_…` |
| `X-Sendrix-Key` | Sí\* | Alternativa a `Authorization`. Si ambas están presentes gana el Bearer. |
| `Content-Type` | Sí | `application/json` |
| `Accept` | Recomendada | `application/json` |
| `Prefer` | Opcional | `respond-async` fuerza el envío asíncrono. |
| `X-Sendrix-Async` | Opcional | `true` o `1` fuerza el envío asíncrono. |

\* Se requiere una de las dos cabeceras de autenticación.

### Claves de proyecto

| Prefijo | Comportamiento |
|---|---|
| `sndx_live_` | Clave productiva. Envía a través del proveedor conectado. |
| `sndx_test_` | Clave de test. Fuerza **modo sandbox**: captura en el Buzón Sandbox, nunca envía correos reales. |

Las claves se muestran una sola vez al crearlas y se almacenan con hash SHA-256. La clave resuelve el proyecto y el propietario; **no existe** cabecera `X-Project-ID` ni endpoints de administración JWT en v1.

### Errores de autenticación

```json
// 401 Unauthorized
{ "error": "Unauthorized", "message": "Missing Sendrix API Key. Provide it via Bearer token or X-Sendrix-Key header." }
```

```json
// 401 Unauthorized (clave inválida/revocada)
{ "error": "Unauthorized", "message": "Invalid or revoked Sendrix API Key." }
```

---

## 1. Enviar un correo individual

### `POST /api/v1/send`

Envía un correo transaccional de forma inmediata (`200`) o asíncrona (`202`).

#### Payload

| Campo | Tipo | Requerido | Restricciones |
|---|---|---|---|
| `to` | string | Sí | Email válido (`rfc` + `filter`). Un único destinatario. |
| `subject` | string | Sí\* | Máx. 998 caracteres. \*No requerido si se usa `template`. |
| `html` | string | Sí\* | Cuerpo HTML. \*No requerido si se usa `template`. |
| `text` | string | No | Alternativa en texto plano. |
| `template` | string | No | Slug de plantilla (o su id numérico). |
| `variables` | object | No | Mapa de sustitución para `{{ variable }}`. |
| `from_name` | string | No | Sobrescribe el nombre del remitente (máx. 255). |
| `reply_to` | string | No | Email de respuesta. |
| `cc` | string[] | No | Emails válidos. |
| `bcc` | string[] | No | Emails válidos. |
| `attachments` | object[] | No | Máx. 10. Cada uno: `filename` (req), `content` (req, base64), `content_type` (opcional). Total decodificado ≤ 10 MB. |
| `async` | boolean | No | `true` → `202 Accepted` y encolado. |
| `sandbox` | boolean | No | `true` → captura en Sandbox. |

Si se envía `template` **y** `subject`/`html`, los valores del payload tienen prioridad sobre los de la plantilla.

#### Ejemplo curl

```bash
curl -X POST "$SENDRIX_BASE_URL/api/v1/send" \
  -H "Authorization: Bearer $SENDRIX_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "to": "customer@example.com",
    "subject": "¡Bienvenido!",
    "html": "<h1>Bienvenido</h1>",
    "reply_to": "soporte@miapp.com"
  }'
```

#### Respuesta 200 OK (enviado)

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

#### Respuesta 200 OK (capturado en sandbox)

```json
{
  "id": "01923e77-…",
  "project_id": 1,
  "status": "sandbox",
  "resend_id": "sndx_sandbox_AbCdEf1234567890",
  "from": "Mi App <waitlist@sandbox.local>",
  "to": "customer@example.com",
  "subject": "¡Bienvenido!",
  "sent_at": "2026-10-01T15:00:00+00:00"
}
```

#### Respuesta 202 Accepted (encolado)

```json
{
  "id": "01923e77-…",
  "project_id": 1,
  "status": "queued",
  "resend_id": null,
  "from": "Mi App <waitlist@creator.dev>",
  "to": "customer@example.com",
  "subject": "¡Bienvenido!",
  "sent_at": null
}
```

#### Códigos de error

| HTTP | `error` | Descripción |
|---|---|---|
| `401` | `Unauthorized` | Clave ausente, inválida o revocada. |
| `422` | *(errores de validación)* | Payload inválido. P. ej. base64 inválido en `attachments.0.content`, tamaño > 10 MB en `attachments`. |
| `404` | `TemplateNotFound` | La plantilla no existe para el proyecto. |
| `422` | `TemplateInactive` | La plantilla existe pero está inactiva. |
| `422` | `RecipientSuppressed` | Destinatario en la lista de supresión (incluye `reason`). |
| `429` | `QuotaExceeded` | Cuota diaria del proyecto superada. |
| `400` | `ProviderNotConfigured` | Sin proveedor conectado o dominio verificado. |
| `502` | `ProviderError` | El proveedor rechazó el envío. |

```json
// 422 RecipientSuppressed
{ "error": "RecipientSuppressed", "message": "…", "reason": "hard_bounce" }

// 429 QuotaExceeded
{ "error": "QuotaExceeded", "message": "Project 'Mi App' has exceeded its daily quota of 100 emails." }

// 404 TemplateNotFound
{ "error": "TemplateNotFound", "message": "…" }

// 400 ProviderNotConfigured
{ "error": "ProviderNotConfigured", "message": "…" }

// 502 ProviderError
{ "error": "ProviderError", "message": "Resend API error: …" }
```

---

## 2. Enviar por lotes (Batch)

### `POST /api/v1/batch`

Envía entre **1 y 100** correos en una única petición. Cada ítem se procesa de forma independiente.

#### Payload

```json
{
  "batch": [
    { "to": "user1@example.com", "subject": "Notificación 1", "html": "<p>Uno</p>" },
    { "to": "user2@example.com", "template": "welcome", "variables": { "name": "Ana" } }
  ],
  "async": false
}
```

- El array **debe** llamarse `batch` (mínimo 1, máximo 100).
- Cada ítem admite los mismos campos que `/send` salvo `attachments`: `to`, `subject`, `html`, `text`, `template`, `variables`, `reply_to`, `from_name`, `cc`, `bcc` y `sandbox` por ítem.
- `async: true` a nivel raíz encola todos los ítems.
- Si falta `batch` o está vacío: `422` con `{"error":"ValidationFailed","errors":{…}}`.

#### Respuesta

```json
{
  "total": 2,
  "successful": 1,
  "failed": 1,
  "data": [
    { "index": 0, "to": "user1@example.com", "status": "sent", "id": "019…", "provider_message_id": "resend-msg-1" },
    { "index": 1, "to": "user2@example.com", "status": "suppressed", "error": "RecipientSuppressed", "message": "…", "reason": "hard_bounce" }
  ]
}
```

| HTTP | Cuándo |
|---|---|
| `200` | Todos los ítems exitosos. |
| `207` Multi-Status | Éxito parcial (algunos ítems fallan). |
| `422` | Todos los ítems fallan, o payload raíz inválido. |

Estados por ítem (`data[].status`): `sent`, `queued`, `suppressed`, `quota_exceeded`, `provider_error`, `error` (plantilla), `failed`.

```json
// 422 ValidationFailed
{ "error": "ValidationFailed", "message": "The batch request payload failed validation.", "errors": { "batch": ["The batch parameter is required."] } }
```

---

## 3. Consultar el estado de un correo

### `GET /api/v1/emails/{id}`

`{id}` es el **log id** (UUID) devuelto por `/send` o `/batch`.

#### Respuesta 200 OK

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

#### Estados posibles

| Estado | Significado |
|---|---|
| `queued` | Encolado para envío asíncrono. |
| `sending` | El worker lo está procesando. |
| `sent` | Entregado al proveedor. |
| `delivered` | Confirmado por el servidor de destino (webhook del proveedor). |
| `bounced` | Rebotó. |
| `complained` | El destinatario lo marcó como spam. |
| `failed` | Fallo definitivo o reintentos agotados. |
| `sandbox` | Capturado en el Buzón Sandbox (no se envió). |

`metadata` puede incluir: `cc`, `bcc`, descriptores de `attachments`, `open_count`, `click_count`, `events` (auditoría), y `failover_used` / `failover_provider` / `primary_error`.

```json
// 404 NotFound (no existe o pertenece a otro proyecto)
{ "error": "NotFound", "message": "Email not found or access denied for this project." }
```

---

## 4. Health check

### `GET /api/v1/health`

Sin autenticación.

```json
{ "status": "ok", "version": "v1" }
```

> Endpoints adicionales de infraestructura: `GET /up` (Laravel health) y `GET /health` (web, comprueba la conexión a base de datos).

---

## 5. Webhook entrante del proveedor

### `POST /api/v1/webhooks/resend`

Endpoint **interno** que recibe los eventos de Resend. Verifica la firma Svix (`svix-id`, `svix-timestamp`, `svix-signature`) con tolerancia de 5 minutos. No debe invocarse desde aplicaciones cliente.

Eventos procesados: `email.sent`, `email.delivered`, `email.bounced` (añade supresión `hard_bounce`), `email.complained` (añade supresión global `spam_complaint`), `email.opened`, `email.clicked`.

```json
// 401 InvalidSignature
{ "error": "InvalidSignature", "message": "Webhook signature verification failed." }

// 200 OK
{ "received": true, "result": { "processed": true, "log_id": "019…", "event_type": "email.delivered", "status": "delivered" } }
```

---

## 6. Resiliencia del cliente (recomendada)

Aunque la API v1 no aplica un límite fijo de peticiones por minuto, sí aplica **cuota diaria por proyecto** (`429 QuotaExceeded`) y el proveedor puede limitar. Recomendaciones:

1. Envía correos no urgentes con `async: true` (envío en cola).
2. Ante `429` o `5xx`, aplica backoff exponencial (≥ 60 s).
3. Persiste el `id` devuelto y consulta el estado con `GET /api/v1/emails/{id}`.
4. No reintentes errores permanentes (`401`, `404`, `422`).
5. Verifica la firma de los webhooks antes de confiar en el payload.
