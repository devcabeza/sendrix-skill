# Sendrix API Reference

Especificación exhaustiva de endpoints, esquemas de payload, cabeceras y códigos de respuesta de la API v1 de Sendrix.

---

## Autenticación y Cabeceras

| Cabecera | Requerido | Descripción |
|---|---|---|
| `Authorization` | Sí | Token Bearer con tu API key: `Bearer sndx_live_...` |
| `Content-Type` | Sí | `application/json` |
| `Accept` | Sí | `application/json` |
| `X-Sendrix-Key` | Opcional | Alternativa a la cabecera `Authorization` |
| `Prefer` | Opcional | Si se envía `respond-async`, procesa el envío de forma asíncrona |

---

## 1. Envío Individual de Correos

### `POST /api/v1/send`

Despacha un correo transaccional de forma inmediata o asíncrona.

#### Parámetros del Payload
- `to` (*string*, requerido): Correo del destinatario.
- `subject` (*string*, requerido): Asunto (entre 1 y 998 caracteres).
- `html` (*string*, requerido): Cuerpo HTML del correo.
- `text` (*string*, opcional): Versión alternativa en texto plano.
- `from_name` (*string*, opcional): Sobrescribe el nombre del remitente del proyecto.
- `reply_to` (*string*, opcional): Buzón de respuestas.
- `cc` (*string[]*, opcional): Lista de direcciones en copia.
- `bcc` (*string[]*, opcional): Lista de direcciones en copia oculta.
- `async` (*boolean*, opcional): `true` para respuesta inmediata `202 Accepted` y despacho en cola de Sendrix.
- `sandbox` (*boolean*, opcional): `true` para capturar en el buzón Sandbox sin envío real.

#### Respuestas
- **`200 OK` (Entrega exitosa o duplicado prevenido):**
  ```json
  {
    "success": true,
    "id": "sndx_msg_01j98...",
    "log_id": "01J98ABCDEF...",
    "duplicated": false
  }
  ```
- **`202 Accepted` (Encolado asíncrono):**
  ```json
  {
    "status": "queued",
    "log_id": "01J98ABCDEF...",
    "message": "Request queued for async delivery"
  }
  ```
- **`429 Too Many Requests` (Límite de tasa 5 req/min excedido):**
  ```json
  {
    "error": "rate_limit_exceeded",
    "message": "Rate limit exceeded. Try again in 45 seconds.",
    "retry_after_seconds": 45
  }
  ```

---

## 2. Envío por Lotes (Batch)

### `POST /api/v1/batch`

Permite enviar hasta 100 correos en una única llamada HTTP.

#### Payload
```json
{
  "emails": [
    {
      "to": "destinatario1@ejemplo.com",
      "subject": "Factura 1",
      "html": "<p>Detalles factura 1</p>"
    },
    {
      "to": "destinatario2@ejemplo.com",
      "subject": "Factura 2",
      "html": "<p>Detalles factura 2</p>"
    }
  ]
}
```

---

## 3. Consulta de Trazabilidad y Estado

### `GET /api/v1/emails/{id}`

Obtiene el estado en tiempo real del correo usando el ID retornado por `/api/v1/send` o el ID del registro.

#### Respuesta
```json
{
  "id": "sndx_msg_01j98...",
  "status": "delivered",
  "to": "destinatario1@ejemplo.com",
  "subject": "Factura 1",
  "sent_at": "2026-10-01T15:20:00Z"
}
```
Estados posibles: `queued`, `sending`, `sent`, `delivered`, `bounced`, `complained`, `failed`.
