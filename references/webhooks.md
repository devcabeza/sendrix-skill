# Sendrix — Webhooks de eventos de entrega

Sendrix notifica a tu aplicación los eventos de entrega de cada correo (entregado, abierto, clic, rebotado, queja de spam, fallo) mediante un `POST` firmado a la URL configurada en el proyecto.

---

## 1. Cabeceras del webhook

Cada petición incluye:

```http
X-Sendrix-Signature: t=1727447400,v1=9f86d081884c7d659a2feaa0c55ad015a3bf...
X-Sendrix-Event: email.delivered
X-Sendrix-Delivery-Id: 01923b78-xxxx-xxxx-xxxx-xxxxxxxxxxxx
X-Sendrix-Timestamp: 1727447400
User-Agent: Sendrix-Webhook-Dispatcher/1.0
Content-Type: application/json
```

| Cabecera | Descripción |
|---|---|
| `X-Sendrix-Signature` | `t=<unix_timestamp>,v1=<hex_hmac_sha256>`. |
| `X-Sendrix-Event` | Tipo de evento (ver abajo). |
| `X-Sendrix-Delivery-Id` | UUID de este intento de entrega. |
| `X-Sendrix-Timestamp` | Unix timestamp usado en la firma. |

---

## 2. Eventos soportados

`email.delivered`, `email.bounced`, `email.complained`, `email.opened`, `email.clicked`, `email.failed` y `ping` (prueba desde el dashboard). Suscribirse a `*` recibe todos.

---

## 3. Payload

```json
{
  "id": "01923b78-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "event": "email.delivered",
  "created_at": "2026-10-01T15:00:05+00:00",
  "data": {
    "email_id": "01923e77-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
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

- `id`: id de la entrega del webhook (también en `X-Sendrix-Delivery-Id`).
- `data.email_id`: **log id** de Sendrix (el mismo que devuelve `/api/v1/send`).
- `data.metadata`: puede incluir `open_count`, `click_count`, `events` (auditoría), datos de rebote, etc.

---

## 4. Algoritmo de verificación

```
signed_payload     = timestamp + "." + raw_request_body
expected_signature = hex( hmac_sha256( signed_payload, WEBHOOK_SECRET ) )
```

Compara de forma constante (`hash_equals` / `crypto.timingSafeEqual`) contra el valor `v1=` de `X-Sendrix-Signature`. Verifica además que el `t=` esté dentro de una tolerancia (recomendado 5 minutos).

> **Importante:** usa el **cuerpo crudo** de la petición (bytes exactos), no un objeto re-serializado.

---

## 5. Verificación en PHP / Laravel

```php
<?php

declare(strict_types=1);

$signatureHeader = request()->header('X-Sendrix-Signature');
$rawBody = request()->getContent();
$secret = config('services.sendrix.webhook_secret'); // el secreto configurado en el proyecto

if (! $signatureHeader || ! $secret) {
    abort(401, 'Firma o secreto no provisto');
}

preg_match('/t=(\d+),v1=([a-f0-9]+)/', $signatureHeader, $matches);
$timestamp = $matches[1] ?? '0';
$signature = $matches[2] ?? '';

if (abs(time() - (int) $timestamp) > 300) {
    abort(401, 'Firma expirada');
}

$expected = hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret);

if (! hash_equals($expected, $signature)) {
    abort(401, 'Firma de webhook inválida');
}

$event = request()->header('X-Sendrix-Event');
$data  = request()->json('data');
```

---

## 6. Verificación en Node.js

```javascript
import crypto from 'crypto';

export function verifySendrixWebhook(rawBody, signatureHeader, secret, toleranceSeconds = 300) {
  const parts = Object.fromEntries(
    String(signatureHeader).split(',').map((p) => p.split('=', 2))
  );
  const timestamp = parts.t;
  const signature = parts.v1;

  if (!timestamp || !signature) return false;
  if (Math.abs(Math.floor(Date.now() / 1000) - Number(timestamp)) > toleranceSeconds) return false;

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${rawBody}`)
    .digest('hex');

  return crypto.timingSafeEqual(Buffer.from(signature, 'utf8'), Buffer.from(expected, 'utf8'));
}
```

En Express, asegúrate de usar el **body crudo**:

```javascript
app.post('/webhooks/sendrix', express.raw({ type: 'application/json' }), (req, res) => {
  const ok = verifySendrixWebhook(
    req.body.toString('utf8'),
    req.header('X-Sendrix-Signature'),
    process.env.SENDRIX_WEBHOOK_SECRET,
  );

  if (! ok) return res.status(401).send('invalid signature');

  const payload = JSON.parse(req.body.toString('utf8'));
  res.json({ received: true });
});
```

---

## 7. Verificación en Python

```python
import hmac
import hashlib
import time

def verify_sendrix_signature(raw_body: bytes, signature_header: str, secret: str, tolerance: int = 300) -> bool:
    parts = dict(p.split("=", 1) for p in signature_header.split(",") if "=" in p)
    timestamp, signature = parts.get("t"), parts.get("v1")
    if not timestamp or not signature:
        return False
    if abs(int(time.time()) - int(timestamp)) > tolerance:
        return False

    expected = hmac.new(
        secret.encode(),
        f"{timestamp}.{raw_body.decode()}".encode(),
        hashlib.sha256,
    ).hexdigest()

    return hmac.compare_digest(expected, signature)
```

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.post("/webhooks/sendrix")
async def sendrix_webhook(request: Request):
    raw = await request.body()
    if not verify_sendrix_signature(raw, request.headers.get("x-sendrix-signature", ""), SECRET):
        raise HTTPException(status_code=401, detail="invalid signature")

    event = request.headers.get("x-sendrix-event")
    payload = await request.json()
    return {"received": True}
```

---

## 8. Reintentos y buenas prácticas

- Sendrix reintenta hasta **3 veces** con backoff `10s → 60s → 300s` (cola `webhooks`).
- Una respuesta `4xx` de tu endpoint se considera fallo **permanente** (no se reintenta). `5xx`/timeout sí se reintentan.
- Responde `2xx` **rápido** (idealmente < 5 s) y procesa la lógica de forma asíncrona.
- El endpoint debe ser **idempotente**: usa `X-Sendrix-Delivery-Id` o `data.email_id` + `event` para deduplicar.
- El dashboard permite **reenviar** una entrega fallida y **enviar un test** (`ping`).
