# Sendrix — Verificación y Procesamiento de Webhooks

Sendrix notifica eventos de entrega de correo a tu aplicación en tiempo real (entregado, abierto, rebotado, queja de spam).

---

## 1. Cabeceras del Webhook

Cada petición POST a tu URL de webhook incluye:

```http
X-Sendrix-Signature: t=1727447400,v1=9f86d081884c7d659a2feaa0c55ad015a3bf...
X-Sendrix-Event: email.delivered
X-Sendrix-Delivery-Id: 01923b78-xxxx-xxxx-xxxx-xxxxxxxxxxxx
X-Sendrix-Timestamp: 1727447400
```

---

## 2. Verificación de Firma en PHP / Laravel

```php
<?php

$signatureHeader = request()->header('X-Sendrix-Signature');
$rawBody = request()->getContent();
$secret = config('services.sendrix.webhook_secret'); // 'whsec_...'

if (! $signatureHeader || ! $secret) {
    abort(401, 'Firma o secreto no provisto');
}

preg_match('/t=(\d+),v1=([a-f0-9]+)/', $signatureHeader, $matches);
$timestamp = $matches[1] ?? '0';
$signature = $matches[2] ?? '';

// Verificar tolerancia de tiempo (ej. 5 minutos para prevenir replay attacks)
if (abs(time() - (int) $timestamp) > 300) {
    abort(401, 'Firma expirada');
}

$expectedSignature = hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret);

if (! hash_equals($expectedSignature, $signature)) {
    abort(401, 'Firma de webhook inválida');
}

// Procesar el evento
$event = request()->header('X-Sendrix-Event');
$data = request()->json('data');
```

---

## 3. Verificación de Firma en Node.js

```javascript
import crypto from 'crypto';

export function verifySendrixWebhook(rawBody, signatureHeader, secret) {
  const parts = signatureHeader.split(',');
  const timestamp = parts.find(p => p.startsWith('t='))?.slice(2);
  const signature = parts.find(p => p.startsWith('v1='))?.slice(3);

  if (!timestamp || !signature) return false;

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${rawBody}`)
    .digest('hex');

  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```
