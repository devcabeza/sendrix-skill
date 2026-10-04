# Sendrix — Guía de integración para Node.js y TypeScript

Cliente ligero (Node 18+ `fetch`) con tipado, reintentos con backoff y soporte de lotes, plantillas, adjuntos y verificación de webhooks. Compatible con Express, Next.js y funciones serverless.

---

## 1. Variables de entorno

```dotenv
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
SENDRIX_WEBHOOK_SECRET=tu_secreto_de_webhook
```

---

## 2. Cliente TypeScript (`sendrix.ts`)

```typescript
export interface Attachment {
  filename: string;
  content: string; // base64
  content_type?: string;
}

export interface EmailPayload {
  to: string;
  subject?: string;
  html?: string;
  text?: string;
  template?: string;
  variables?: Record<string, unknown>;
  from_name?: string;
  reply_to?: string;
  cc?: string[];
  bcc?: string[];
  attachments?: Attachment[];
  async?: boolean;
  sandbox?: boolean;
}

export interface SendResult {
  id: string;
  project_id: number;
  status: 'sent' | 'sandbox' | 'queued';
  resend_id: string | null;
  from: string;
  to: string;
  subject: string;
  sent_at: string | null;
}

export interface BatchResult {
  total: number;
  successful: number;
  failed: number;
  data: Array<Record<string, unknown>>;
}

export class SendrixError extends Error {
  constructor(message: string, public readonly status: number, public readonly body: unknown) {
    super(message);
  }
}

export class SendrixClient {
  private readonly baseUrl: string;
  private readonly apiKey: string;

  constructor(apiKey = process.env.SENDRIX_KEY ?? '', baseUrl = process.env.SENDRIX_BASE_URL ?? '') {
    this.apiKey = apiKey;
    this.baseUrl = (baseUrl || 'https://sendrix.alejandrocabeza.dev').replace(/\/$/, '');
    if (!this.apiKey) throw new Error('Falta SENDRIX_KEY.');
  }

  private headers(): Record<string, string> {
    return {
      Authorization: `Bearer ${this.apiKey}`,
      'Content-Type': 'application/json',
      Accept: 'application/json',
    };
  }

  private async request<T>(path: string, init: RequestInit, retries = 3): Promise<T> {
    for (let attempt = 1; attempt <= retries; attempt++) {
      const res = await fetch(`${this.baseUrl}${path}`, { ...init, headers: this.headers() });
      const body = await res.json().catch(() => ({}));

      if (res.status === 429 || res.status >= 500) {
        const delayMs = Math.min(60_000, 2 ** attempt * 1000);
        if (attempt === retries) throw new SendrixError(`Sendrix ${res.status}`, res.status, body);
        await new Promise((r) => setTimeout(r, delayMs));
        continue;
      }

      if (!res.ok) throw new SendrixError(`Sendrix ${res.status}`, res.status, body);
      return body as T;
    }
    throw new SendrixError('Sendrix: máximo de reintentos alcanzado', 0, null);
  }

  send(payload: EmailPayload): Promise<SendResult> {
    return this.request<SendResult>('/api/v1/send', {
      method: 'POST',
      body: JSON.stringify(payload),
    });
  }

  sendBatch(emails: EmailPayload[], async = false): Promise<BatchResult> {
    return this.request<BatchResult>('/api/v1/batch', {
      method: 'POST',
      body: JSON.stringify({ batch: emails, async }),
    });
  }

  getStatus(id: string): Promise<Record<string, unknown>> {
    return this.request(`/api/v1/emails/${encodeURIComponent(id)}`, { method: 'GET' });
  }
}
```

---

## 3. Ejemplos de uso

### Next.js (App Router / Server Action)

```typescript
'use server';

import { SendrixClient } from '@/lib/sendrix';

const sendrix = new SendrixClient();

export async function sendWelcomeEmail(email: string, name: string) {
  const result = await sendrix.send({
    to: email,
    template: 'welcome',
    variables: { name, company: 'Acme' },
    from_name: 'Equipo Acme',
  });

  return result.id; // log id para consultar el estado
}
```

### Express

```typescript
import express from 'express';
import { SendrixClient } from './sendrix';

const app = express();
app.use(express.json());
const sendrix = new SendrixClient();

app.post('/api/notify', async (req, res) => {
  const result = await sendrix.send({
    to: req.body.email,
    subject: 'Nueva notificación',
    html: '<p>Tienes una novedad.</p>',
  });
  res.json({ id: result.id, status: result.status });
});
```

### Adjuntos

```typescript
import { readFileSync } from 'node:fs';

await sendrix.send({
  to: 'cliente@example.com',
  subject: 'Tu factura',
  html: '<p>Adjuntamos tu factura.</p>',
  attachments: [
    {
      filename: 'factura.pdf',
      content: readFileSync('./factura.pdf').toString('base64'),
      content_type: 'application/pdf',
    },
  ],
});
```

---

## 4. Verificación de webhooks

Ver el ejemplo completo en [webhooks.md](webhooks.md). Recuerda usar el **cuerpo crudo** (`express.raw`).

---

## 5. Buenas prácticas

- Usa `async: true` para envíos masivos y no bloquear la respuesta HTTP.
- Guarda el `id` devuelto para consultar el estado (`GET /api/v1/emails/{id}`).
- No reintentes `401`, `404` ni `422`: son errores permanentes.
- En desarrollo, usa una clave `sndx_test_*` o `sandbox: true` para no enviar correos reales.
