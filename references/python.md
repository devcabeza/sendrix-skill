# Sendrix — Guía de integración para Python

Cliente con `requests` (o `httpx`) con reintentos, soporte de plantillas, lotes, adjuntos y verificación de webhooks. Compatible con Django y FastAPI.

---

## 1. Configuración

```bash
pip install requests
```

```dotenv
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
SENDRIX_WEBHOOK_SECRET=tu_secreto_de_webhook
```

---

## 2. Cliente (`sendrix.py`)

```python
import os
import time
import base64
from typing import Any

import requests

SENDRIX_BASE_URL = os.getenv("SENDRIX_BASE_URL", "https://sendrix.alejandrocabeza.dev").rstrip("/")
SENDRIX_KEY = os.getenv("SENDRIX_KEY", "")


class SendrixError(RuntimeError):
    def __init__(self, message: str, status: int, body: Any = None):
        super().__init__(message)
        self.status = status
        self.body = body


def _headers() -> dict[str, str]:
    return {
        "Authorization": f"Bearer {SENDRIX_KEY}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }


def _post(path: str, payload: dict, retries: int = 3) -> dict:
    url = f"{SENDRIX_BASE_URL}{path}"

    for attempt in range(1, retries + 1):
        response = requests.post(url, json=payload, headers=_headers(), timeout=15)

        # Permanentes: no reintentar
        if response.status_code in (401, 404, 422):
            raise SendrixError(f"Sendrix {response.status_code}: {response.text}", response.status_code, response.json())

        # Temporales: cuota diaria (429) o error de proveedor/servidor (5xx)
        if response.status_code == 429 or response.status_code >= 500:
            if attempt == retries:
                raise SendrixError(f"Sendrix {response.status_code}", response.status_code, response.json())
            time.sleep(min(60, 2 ** attempt))
            continue

        response.raise_for_status()
        return response.json()

    raise SendrixError("Sendrix: máximo de reintentos alcanzado", 0)


def send_email(
    to: str,
    *,
    subject: str | None = None,
    html: str | None = None,
    template: str | None = None,
    variables: dict | None = None,
    from_name: str | None = None,
    attachments: list[dict] | None = None,
    is_async: bool = False,
) -> dict:
    payload = {
        k: v
        for k, v in {
            "to": to,
            "subject": subject,
            "html": html,
            "template": template,
            "variables": variables,
            "from_name": from_name,
            "attachments": attachments,
            "async": is_async,
        }.items()
        if v is not None
    }
    return _post("/api/v1/send", payload)


def send_batch(emails: list[dict], is_async: bool = False) -> dict:
    return _post("/api/v1/batch", {"batch": emails, "async": is_async})


def get_status(log_id: str) -> dict:
    response = requests.get(
        f"{SENDRIX_BASE_URL}/api/v1/emails/{log_id}", headers=_headers(), timeout=15
    )
    if response.status_code == 404:
        raise SendrixError("Email no encontrado", 404, response.json())
    response.raise_for_status()
    return response.json()
```

### Ejemplo

```python
# Envío directo
result = send_email(
    "cliente@example.com",
    subject="Tu pedido #1042",
    html="<p>Gracias por tu compra.</p>",
)
print(result["id"], result["status"])

# Con plantilla y variables
send_email(
    "cliente@example.com",
    template="welcome",
    variables={"name": "Ana", "company": "Acme"},
)

# Con adjunto (base64)
with open("factura.pdf", "rb") as fh:
    content = base64.b64encode(fh.read()).decode()

send_email(
    "cliente@example.com",
    subject="Tu factura",
    html="<p>Adjunta encontrarás tu factura.</p>",
    attachments=[{"filename": "factura.pdf", "content": content, "content_type": "application/pdf"}],
)

# Lote
send_batch(
    [
        {"to": "a@example.com", "subject": "Hola A", "html": "<p>A</p>"},
        {"to": "b@example.com", "template": "welcome", "variables": {"name": "B"}},
    ]
)
```

---

## 3. Verificación de webhooks

Implementación completa en [webhooks.md](webhooks.md) (HMAC-SHA256 sobre `timestamp.raw_body`).

---

## 4. Buenas prácticas

- Usa `async=True` para envíos masivos y no bloquear el request.
- Persiste el `id` devuelto y consulta el estado con `get_status()`.
- En desarrollo usa una clave `sndx_test_*` o `sandbox=True` para capturar correos sin enviarlos.
