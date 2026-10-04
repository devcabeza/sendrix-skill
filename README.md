# Sendrix AI Agent Skill

[![skills.sh](https://img.shields.io/badge/skills.sh-sendrix--skill-blue?style=flat-square)](https://skills.sh)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)

Paquete oficial de [skills.sh](https://skills.sh) para que agentes de IA (**Antigravity**, **Claude Code**, **Cursor**, **Windsurf**, **GitHub Copilot**) integren y operen **Sendrix**, la pasarela/proxy de correo transaccional, con contexto completo del producto.

---

## ⚡ Instalación

Instala la skill directamente en tu proyecto con el CLI estándar de `skills`:

```bash
npx skills add devcabeza/sendrix-skill
```

Esto configura la skill en el directorio de tu agente (p. ej. `.agents/skills/sendrix`, `.claude/skills/sendrix` o `.cursor/rules/sendrix.mdc`).

### Instalación desde una instancia de Sendrix

También puedes instalar la última versión desde el propio servidor:

```bash
curl -fsSL $SENDRIX_BASE_URL/skill/install.sh | bash
```

---

## 🤖 Qué aprende tu agente

Una vez instalada, tu agente entiende **todo el producto Sendrix**:

1. **Autenticación y configuración**
   - `SENDRIX_BASE_URL` y `SENDRIX_KEY`.
   - Claves de proyecto `sndx_live_*` (producción) y `sndx_test_*` (sandbox).
   - Cabeceras `Authorization: Bearer` o `X-Sendrix-Key`.

2. **Envío de correos**
   - Individual: `POST /api/v1/send`.
   - Por lotes (1–100): `POST /api/v1/batch` con la clave `batch`.
   - Asíncrono con Horizon: `async: true`, `Prefer: respond-async`, `X-Sendrix-Async`.
   - Adjuntos en base64 (≤ 10 MB) y plantillas con `{{ variables }}`.

3. **Trazabilidad**
   - Estado de entrega: `GET /api/v1/emails/{id}`.
   - Ciclo de vida: `queued → sending → sent → delivered` (+ `failed`, `bounced`, `complained`, `sandbox`).

4. **Protección de entregabilidad**
   - Supresiones por `hard_bounce`, `spam_complaint`, `manual`, `unsubscribe`.
   - Rechazo `422 RecipientSuppressed` antes de enviar/encolar.

5. **Webhooks salientes firmados**
   - Verificación HMAC-SHA256 (`X-Sendrix-Signature`), eventos `email.*`, reintentos e idempotencia.

6. **Proveedores y failover**
   - Resend primario; Postmark o SMTP como respaldo con fallback automático.

7. **Integraciones por framework**
   - **Laravel**: `SendrixTransport` (mailer nativo), Mailables, cola Horizon `emails`, adjuntos, Pest.
   - **Node.js / TypeScript**: cliente `fetch` tipado, lotes, estados y verificación de webhooks.
   - **Python**: cliente `requests`/`httpx` con reintentos.

8. **Despliegue**
   - Self-hosting con Docker/Coolify, PostgreSQL + Redis + Horizon, variables de entorno y health checks.

---

## 📚 Referencias

- [SKILL.md](./SKILL.md) — instrucciones principales para el agente.
- [api-reference.md](./references/api-reference.md) — endpoints, esquemas y códigos de respuesta.
- [templates.md](./references/templates.md) — plantillas, variables y constructor visual.
- [sandbox.md](./references/sandbox.md) — modo sandbox y buzón de pruebas.
- [suppressions.md](./references/suppressions.md) — rebotes, quejas y lista de supresión.
- [webhooks.md](./references/webhooks.md) — firma HMAC y procesamiento de eventos.
- [providers-failover.md](./references/providers-failover.md) — Resend, Postmark y SMTP.
- [laravel.md](./references/laravel.md) — integración Laravel.
- [nodejs.md](./references/nodejs.md) — integración Node.js / TypeScript.
- [python.md](./references/python.md) — integración Python.
- [deployment.md](./references/deployment.md) — despliegue y operación.

---

## 💡 Prompt de ejemplo

Tras instalar, pídele a tu agente:

> *"He instalado la skill de Sendrix. Integra el envío de correos transaccionales con Sendrix para bienvenida y recuperación de contraseña, en segundo plano con cola, y añade verificación de webhooks."*

---

## ⚡ Descubrimiento dinámico

Sendrix evoluciona. El agente puede consultar la especificación en vivo en:

- `{SENDRIX_BASE_URL}/skill.md`
- `{SENDRIX_BASE_URL}/llms.txt`
- `{SENDRIX_BASE_URL}/llms-full.txt`

---

## Licencia

MIT License © 2026 Sendrix / devcabeza
