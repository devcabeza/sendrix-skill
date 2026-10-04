# Sendrix — Plantillas (Templates)

Sendrix permite definir **plantillas reutilizables** por proyecto, con variables dinámicas y un constructor visual de bloques. Se gestionan desde el dashboard (Livewire) y se usan desde la API con `template` + `variables`.

---

## 1. Modelo de plantilla

| Campo | Descripción |
|---|---|
| `name` | Nombre legible. |
| `slug` | Identificador único por proyecto (se usa en la API). |
| `subject` | Asunto, admite variables. |
| `html_content` | HTML compilado/editable, admite variables. |
| `text_content` | Alternativa en texto plano (opcional). |
| `sample_variables` | JSON con valores de ejemplo para previsualización. |
| `design_blocks` | JSON con los bloques del constructor visual (opcional). |
| `is_active` | Si está inactiva, la API responde `422 TemplateInactive`. |

- Único por `(project_id, slug)`.
- Se pueden duplicar bloques desde el editor.

---

## 2. Variables dinámicas

Sintaxis en `subject`, `html_content` y `text_content`:

```
{{ name }}
{{ user.email }}
{{ code }}
```

- Se permite notación con puntos (`{{ user.email }}`).
- Las variables **no provistas se sustituyen por cadena vacía**.
- Solo se interpolan valores escalares (o con `__toString`).
- Al enviar por API, pasa el mapa en `variables`:

```json
{
  "to": "user@example.com",
  "template": "welcome-email",
  "variables": { "name": "Alejandro", "company": "Sendrix", "code": "98765" }
}
```

### Comportamiento de resolución

- `template` acepta el **slug** o el **id numérico** de la plantilla.
- Si la plantilla no existe para el proyecto → `404 TemplateNotFound`.
- Si existe pero `is_active = false` → `422 TemplateInactive`.
- Si en la misma petición envías `subject`/`html`, esos valores **sobrescriben** los de la plantilla (útil para personalizaciones puntuales).

### Ejemplo curl

```bash
curl -X POST "$SENDRIX_BASE_URL/api/v1/send" \
  -H "Authorization: Bearer $SENDRIX_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "user@example.com",
    "template": "welcome-email",
    "variables": { "name": "Alejandro", "company": "Sendrix" }
  }'
```

### Ejemplo en lote

```json
{
  "batch": [
    { "to": "a@example.com", "template": "welcome", "variables": { "name": "Ana" } },
    { "to": "b@example.com", "template": "welcome", "variables": { "name": "Beto" } }
  ]
}
```

---

## 3. Constructor visual de bloques

El editor construye `design_blocks` (array de bloques) y un compilador genera HTML "bulletproof" (tablas, compatible con clientes de correo) y su versión en texto plano.

Tipos de bloque:

| Tipo | Contenido / ajustes |
|---|---|
| `heading` | `content`, `settings.level` (`h1`/`h2`/`h3`), `align`, `color` |
| `paragraph` | `content`, `align`, `color`, `font_size` |
| `button` | `content` (texto), `settings.url`, `align`, `bg_color`, `text_color`, `border_radius` |
| `image` | `content`/`settings.url`, `alt`, `link_url`, `align`, `width` |
| `divider` | `settings.color` |
| `spacer` | `settings.height` |
| `callout` | `content`, `settings.style` (`info`/`success`/`warning`/`neutral`) |
| `footer` | `content`, `align`, `color`, `font_size` |

Ejemplo de bloque:

```json
{
  "id": "uuid",
  "type": "button",
  "content": "Comenzar",
  "settings": {
    "url": "{{ action_url }}",
    "align": "center",
    "bg_color": "#2563eb",
    "text_color": "#ffffff"
  }
}
```

> Los bloques se compilan a HTML + texto. Al enviar por API normalmente solo necesitas `template` + `variables`; el HTML ya está materializado en `html_content`.

---

## 4. Buenas prácticas

- Mantén una plantilla por caso de uso (`welcome`, `password-reset`, `invoice`, `otp`).
- Previsualiza con `sample_variables` antes de activar.
- No incluyas datos sensibles en `sample_variables`.
- Desactiva plantillas obsoletas en vez de borrarlas: los envíos fallarán de forma explícita (`422 TemplateInactive`).
