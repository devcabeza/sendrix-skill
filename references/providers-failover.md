# Sendrix — Proveedores y Failover

Sendrix entrega a través de un **proveedor primario (Resend)** y puede conmutar automáticamente a un **proveedor de respaldo (Postmark o SMTP)** si el primario falla.

---

## 1. Proveedor primario: Resend

Configuración por usuario (dashboard → *Proveedor*):

1. Introduce tu **API key de Resend** (`re_...`).
2. Sendrix verifica la clave consultando los dominios de la cuenta.
3. Selecciona el **dominio verificado** que se usará como remitente.

Con esto, cada proyecto genera remitentes de la forma:

```
"<project.sender_name>" <project.sender_alias@verified_domain>
```

> La API key de Resend se guarda en la configuración del usuario (`user_provider_settings`) y nunca se expone a los clientes: los proyectos usan sus propias claves `sndx_*`.

---

## 2. Failover

Se configura por usuario y se activa solo si hay proveedor + credenciales.

### Postmark

- `server_token`: Server API Token de Postmark.

### SMTP genérico

- `host`: host SMTP.
- `port`: puerto (por defecto 587).
- `encryption`: `tls` / `ssl` / `null` (si el puerto es 465 se asume TLS).
- `username` / `password` (opcionales).

Opcionalmente, un **dominio verificado de failover** distinto del primario.

---

## 3. Cómo funciona el failover

1. Sendrix intenta el envío con Resend.
2. Si Resend lanza un error, y el failover está configurado (`canFailover()`), reintenta con el proveedor de respaldo usando un remitente con el dominio efectivo de failover.
3. Si el failover tiene éxito, se registra:
   - `metadata.failover_used = true`
   - `metadata.failover_provider = "postmark" | "smtp"`
   - `metadata.primary_error = "<error del primario>"`
4. Si ambos fallan, el log queda `failed` con un mensaje combinado del primario y del failover.

En modo asíncrono, el proveedor primario Resend puede devolver `429`; el job `SendProxiedEmailJob` se libera y reintenta (65 s) antes de rendirse.

---

## 4. Probar el failover

Desde el dashboard se puede **probar la conexión** del proveedor de respaldo:

- **SMTP**: handshake contra el servidor.
- **Postmark**: consulta la información del servidor.

La respuesta incluye `success`, `duration_ms` y `message`.

---

## 5. Diagnóstico

- Revisa el estado del proveedor primario en el dashboard.
- Consulta `GET /api/v1/emails/{id}` y mira `metadata.failover_used` y `metadata.primary_error`.
- Si todos los envíos fallan con `400 ProviderNotConfigured`: falta API key o dominio verificado.
- Si fallan con `502 ProviderError`: revisa la API key/dominio en el proveedor.

---

## 6. Buenas prácticas

- Verifica el dominio de envío (SPF/DKIM) en el proveedor primario antes de activar el tráfico.
- Configura el failover con un dominio verificado independiente cuando sea posible.
- Alerta si el porcentaje de failover sube de forma sostenida.
- Rota las API keys del proveedor periódicamente.
