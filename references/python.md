# Sendrix — Guía de Integración para Python

Esta guía proporciona ejemplos con `requests` o `httpx` para proyectos Python, Django y FastAPI.

---

## 1. Configuración de Entorno

```bash
pip install requests
```

### `.env`
```dotenv
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
```

---

## 2. Cliente de Envío con Manejo de Rate Limit

```python
import os
import time
import requests

SENDRIX_BASE_URL = os.getenv("SENDRIX_BASE_URL", "https://sendrix.alejandrocabeza.dev")
SENDRIX_KEY = os.getenv("SENDRIX_KEY", "")

def send_email(to: str, subject: str, html: str, from_name: str = None, retries: int = 3):
    url = f"{SENDRIX_BASE_URL}/api/v1/send"
    headers = {
        "Authorization": f"Bearer {SENDRIX_KEY}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }
    payload = {
        "to": to,
        "subject": subject,
        "html": html,
    }
    if from_name:
        payload["from_name"] = from_name

    for attempt in range(retries):
        response = requests.post(url, json=payload, headers=headers)

        if response.status_code == 429:
            retry_after = response.json().get("retry_after_seconds", 60)
            print(f"[Sendrix] Rate limit alcanzado. Esperando {retry_after}s...")
            time.sleep(retry_after + 2)
            continue

        response.raise_for_status()
        return response.json()

    raise RuntimeError("Sendrix: Máximo de reintentos alcanzado.")
```
