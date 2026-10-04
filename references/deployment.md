# Sendrix — Despliegue y operación (self-hosting)

Sendrix es una aplicación **Laravel 13 + Livewire 4** que se autoaloja con PostgreSQL y Redis, y usa **Laravel Horizon** para las colas. Esta guía cubre Docker local y producción (Coolify).

---

## 1. Arquitectura de servicios

| Servicio | Responsabilidad |
|---|---|
| **app** | Web (Laravel + Livewire), API v1, dashboard. |
| **worker** | `php artisan horizon` — procesa las colas `default`, `emails`, `webhooks`. |
| **scheduler** | `php artisan schedule:work` — tareas programadas. |
| **postgres** | Base de datos principal (PostgreSQL 17). |
| **redis** | Colas de Horizon, caché y sesiones. |
| **mailpit** (solo local) | Captura SMTP de desarrollo. |

En producción con Nginx + PHP-FPM, el contenedor web ejecuta migraciones y cachea configuración/rutas/vistas antes de arrancar. Un contenedor aparte con `CONTAINER_ROLE=worker` arranca Horizon.

---

## 2. Desarrollo local (Docker Compose)

```bash
docker compose up -d
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate --seed
```

- App: `http://localhost:8000`
- Mailpit (SMTP dev): `http://localhost:8026`
- Postgres: `localhost:5433`
- Redis: `localhost:6380`

Variables clave del compose: `DB_CONNECTION=pgsql`, `REDIS_HOST=redis`, `QUEUE_CONNECTION=redis`.

---

## 3. Producción (Coolify / Docker)

1. **Build pack:** Dockerfile; usa `docker/prod/Dockerfile.prod`.
2. **Servicio web:** contenedor principal (Nginx + PHP-FPM). Ejecuta migraciones y caches al arrancar.
3. **Servicio worker:** mismo Dockerfile con `CONTAINER_ROLE=worker` → `php artisan horizon`.
4. **Servicio scheduler:** mismo Dockerfile con `CONTAINER_ROLE=scheduler` (o `CUSTOM_COMMAND`) → `php artisan schedule:work`.
5. **PostgreSQL y Redis gestionados** o como servicios del stack.
6. Configura `APP_URL` con `https://` (el entrypoint corrige `http://` a `https://` detrás de proxy).

> Para evitar duplicar migraciones, solo el contenedor web debe ejecutarlas.

---

## 4. Variables de entorno (servidor Sendrix)

| Variable | Requerida | Ejemplo / notas |
|---|---|---|
| `APP_NAME` | Sí | `Sendrix` |
| `APP_ENV` | Sí | `production` |
| `APP_KEY` | Sí | `php artisan key:generate` |
| `APP_DEBUG` | Sí | `false` en producción |
| `APP_URL` | Sí | `https://sendrix.alejandrocabeza.dev` |
| `APP_PORT` | No | `8000` |
| `DB_CONNECTION` | Sí | `pgsql` |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | Sí | o `DB_URL` |
| `REDIS_CLIENT` | Sí | `phpredis` |
| `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD` | Sí | o `REDIS_URL` |
| `QUEUE_CONNECTION` | Sí | `redis` |
| `CACHE_STORE` | No | `redis` |
| `SESSION_DRIVER` | No | `redis` |
| `RESEND_WEBHOOK_SECRET` | No | secreto Svix (`whsec_...`) para el webhook entrante de Resend |
| `HORIZON_*` | No | nombre/dominio/prefix de Horizon |
| `CONTAINER_ROLE` | Prod | `worker` para el contenedor Horizon |

> Las **API keys de Resend/Postmark** no van en `.env`: se guardan por usuario en la base de datos desde el dashboard. `RESEND_WEBHOOK_SECRET` sí es de entorno, porque valida el webhook entrante.

### Donaciones (Binance) — opcional

```dotenv
DONATION_BINANCE_PAY_ID="849201842"
DONATION_BINANCE_USDT_ADDRESS="TXk9..."
DONATION_BINANCE_NETWORK="USDT (TRC20 / BEP20)"
```

---

## 5. Colas y programador

- Horizon supervisa las colas `default`, `emails` y `webhooks` (balance `auto`).
- `emails`: envíos asíncronos (`SendProxiedEmailJob`), 5 intentos, backoff `10/30/60/120` s, timeout 60 s.
- `webhooks`: entregas de webhooks (`DeliverWebhookJob`), 3 intentos, backoff `10/60/300` s, timeout 15 s.
- El scheduler (`schedule:work`) ejecuta las tareas programadas (snapshots de Horizon, mantenimiento).

Comandos útiles:

```bash
php artisan horizon            # worker de colas
php artisan horizon:status
php artisan schedule:work       # programador
php artisan queue:failed
```

---

## 6. Health checks y observabilidad

- `GET /up` — Laravel health (`bootstrap/app.php` → `health: '/up'`).
- `GET /health` — web: comprueba conexión a base de datos.
- `GET /api/v1/health` — API v1: `{"status":"ok","version":"v1"}`.
- Horizon UI: `/{HORIZON_PATH}` (por defecto `/horizon`).
- Laravel Telescope disponible en entornos no productivos.

---

## 7. Configurar el webhook entrante de Resend

1. En Resend → *Webhooks*, añade:
   `https://<tu-dominio>/api/v1/webhooks/resend`
2. Suscríbete a `email.sent`, `email.delivered`, `email.bounced`, `email.complained` (y opcionalmente `email.opened`, `email.clicked`).
3. Copia el signing secret (`whsec_...`) a `RESEND_WEBHOOK_SECRET` y reinicia el contenedor.

---

## 8. Checklist de despliegue

- [ ] `APP_KEY`, `APP_URL` (https), `APP_DEBUG=false`.
- [ ] PostgreSQL y Redis accesibles.
- [ ] Migraciones ejecutadas (solo contenedor web).
- [ ] Contenedor worker con Horizon activo.
- [ ] Contenedor scheduler activo.
- [ ] `RESEND_WEBHOOK_SECRET` configurado.
- [ ] Proveedor Resend conectado y dominio verificado desde el dashboard.
- [ ] Backup de la base de datos configurado.
