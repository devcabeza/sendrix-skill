# Sendrix — Supresiones y protección de entregabilidad

Sendrix mantiene una **lista de supresión** para no volver a enviar a direcciones problemáticas. Esto protege la reputación del dominio y evita rebotes repetidos y quejas de spam.

---

## 1. Motivos de supresión

| `reason` | Origen | Alcance |
|---|---|---|
| `hard_bounce` | Webhook `email.bounced` del proveedor (o manual) | Proyecto |
| `spam_complaint` | Webhook `email.complained` del proveedor | **Global** (todos los proyectos del usuario) |
| `manual` | Alta manual desde el dashboard | Proyecto |
| `unsubscribe` | Baja del suscriptor | Proyecto |

---

## 2. Comportamiento automático

Cuando Sendrix recibe un evento del proveedor:

- **`email.bounced`** → añade la dirección con `hard_bounce` (detalle con el mensaje de rebote).
- **`email.complained`** → añade la dirección con `spam_complaint` **a nivel global** del usuario.

Los correos dirigidos a una dirección suprimida se **rechazan antes de enviar** (también antes de encolar en modo `async`).

---

## 3. Respuesta de la API

```json
// 422 Unprocessable Entity
{
  "error": "RecipientSuppressed",
  "message": "Recipient is suppressed.",
  "reason": "hard_bounce"
}
```

- Se aplica tanto a `/api/v1/send` como a cada ítem de `/api/v1/batch` (estado `suppressed`).
- **No** se aplica en modo sandbox.
- Es un error **permanente** para ese destinatario: no reintentes.

---

## 4. Ejemplo en lote (resultado mixto)

```json
{
  "total": 2,
  "successful": 1,
  "failed": 1,
  "data": [
    { "index": 0, "to": "happy@example.com", "status": "sent", "id": "…" },
    { "index": 1, "to": "blocked@example.com", "status": "suppressed", "error": "RecipientSuppressed", "reason": "hard_bounce" }
  ]
}
```

HTTP `207 Multi-Status` cuando hay éxito parcial.

---

## 5. Gestión desde el dashboard

La sección **Supresiones** permite:

- Listar y filtrar direcciones suprimidas.
- Añadir una dirección manualmente (`manual`).
- **Eliminar** una supresión (reactivar la dirección) cuando corresponde.

> Quitar una supresión `hard_bounce` conlleva riesgo: solo hazlo si tienes certeza de que el buzón vuelve a ser válido.

---

## 6. Buenas prácticas

- Respeta SIEMPRE la lista de supresión: es tu principal defensa de entregabilidad.
- Gestiona las bajas (`unsubscribe`) como supresiones para no volver a contactar.
- Monitoriza las tasas de rebote y queja en el dashboard; umbrales sanos:
  - Rebotes ≤ 2 % (aviso), ≤ 4 % (crítico).
  - Quejas ≤ 0,1 %.
- Tras limpiar una supresión, reanuda los envíos poco a poco.
