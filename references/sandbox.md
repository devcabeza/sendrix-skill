# Sendrix — Sandbox (Buzón virtual de pruebas)

El modo **Sandbox** captura los correos dentro de Sendrix sin contactar a ningún proveedor. Es ideal para desarrollo, staging y tests automatizados.

---

## 1. Cómo activar el sandbox

Hay **tres** formas, todas equivalentes:

1. **Clave de test** (`sndx_test_*`): cualquier petición autenticada con una clave de test se fuerza a sandbox automáticamente.
2. **Flag por petición**: `"sandbox": true` en el payload de `/api/v1/send`.
3. **Proyecto sandbox**: si el proyecto tiene `is_sandbox = true`, todos sus envíos van a sandbox.

```json
{
  "to": "test@example.com",
  "subject": "Prueba",
  "html": "<p>Hola</p>",
  "sandbox": true
}
```

---

## 2. Comportamiento

Cuando un envío entra en sandbox:

- **No** se contacta al proveedor (no consume cuota del proveedor).
- **No** se comprueba la lista de supresión.
- **No** se requiere proveedor conectado ni dominio verificado.
- El dominio del remitente cae a `sandbox.local` si no hay dominio verificado.
- El correo se guarda en la tabla `sandbox_emails` y se puede ver en el **Buzón Sandbox** del dashboard.
- El registro en `email_logs` queda con estado `sandbox` y `provider_message_id` sintético (`sndx_sandbox_<random>`).

### Respuesta

```json
{
  "id": "01923e77-…",
  "project_id": 1,
  "status": "sandbox",
  "resend_id": "sndx_sandbox_AbCdEf1234567890",
  "from": "Mi App <waitlist@sandbox.local>",
  "to": "test@example.com",
  "subject": "Prueba",
  "sent_at": "2026-10-01T15:00:00+00:00"
}
```

---

## 3. Qué se captura

Cada correo de sandbox almacena:

- Remitente (`from_name`, `from_email`), `reply_to`.
- Destinatario, asunto.
- `cc`, `bcc`.
- Cuerpo `html_body` y `text_body`.
- Cabeceras MIME generadas (`From`, `To`, `Subject`, `Date`, `Message-ID`, `Reply-To`, `Cc`, `Bcc`).
- **Adjuntos** (nombre, tipo, contenido base64).
- `raw_size_bytes`.

---

## 4. Adjuntos en sandbox

Los adjuntos se aceptan igual que en producción y se almacenan en el buzón:

```json
{
  "to": "test@example.com",
  "subject": "Prueba con adjunto",
  "html": "<p>Hola</p>",
  "sandbox": true,
  "attachments": [
    { "filename": "archivo.txt", "content": "<base64>", "content_type": "text/plain" }
  ]
}
```

Recuerda: máx. 10 adjuntos y ≤ 10 MB decodificados en total.

---

## 5. Buenas prácticas

- Usa claves `sndx_test_*` en local/CI y claves `sndx_live_*` solo en producción.
- En tests de integración puedes apuntar `SENDRIX_BASE_URL` a tu instancia local y usar una clave de test.
- Nunca pongas una clave `sndx_live_*` en entornos no productivos.
- El buzón es una herramienta de diagnóstico; no sustituye a las pruebas contra un proveedor real en preproducción.
