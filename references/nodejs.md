# Sendrix — Guía de Integración para Node.js y TypeScript

Esta guía proporciona un cliente ligero con reintentos y soporte TypeScript para Node.js, Express y Next.js.

---

## 1. Instalación y Variables de Entorno

No requiere librerías externas pesadas; funciona de forma nativa con `fetch` (Node 18+).

### `.env`
```dotenv
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
```

---

## 2. Cliente TypeScript (`sendrix.ts`)

```typescript
export interface SendEmailPayload {
  to: string;
  subject: string;
  html: string;
  text?: string;
  from_name?: string;
  reply_to?: string;
  cc?: string[];
  bcc?: string[];
  async?: boolean;
  sandbox?: boolean;
}

export interface SendrixResponse {
  success?: boolean;
  id?: string;
  log_id?: string;
  status?: string;
  duplicated?: boolean;
  message?: string;
}

export class SendrixClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(apiKey?: string, baseUrl?: string) {
    this.apiKey = apiKey || process.env.SENDRIX_KEY || '';
    this.baseUrl = baseUrl || process.env.SENDRIX_BASE_URL || 'https://sendrix.alejandrocabeza.dev';

    if (!this.apiKey) {
      throw new Error('Sendrix API Key is missing. Define SENDRIX_KEY.');
    }
  }

  async send(payload: SendEmailPayload, retries = 3): Promise<SendrixResponse> {
    const url = `${this.baseUrl}/api/v1/send`;

    for (let attempt = 1; attempt <= retries; attempt++) {
      const res = await fetch(url, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        body: JSON.stringify(payload),
      });

      if (res.status === 429) {
        const errorData = await res.json().catch(() => ({}));
        const retryAfter = errorData.retry_after_seconds || 60;
        console.warn(`[Sendrix] Rate limit reached. Waiting ${retryAfter}s before retry...`);
        await new Promise((resolve) => setTimeout(resolve, (retryAfter + 2) * 1000));
        continue;
      }

      if (!res.ok) {
        const errorText = await res.text();
        throw new Error(`Sendrix Error (${res.status}): ${errorText}`);
      }

      return res.json() as Promise<SendrixResponse>;
    }

    throw new Error('Sendrix: Max retry attempts reached due to rate limiting.');
  }

  async getStatus(id: string): Promise<any> {
    const res = await fetch(`${this.baseUrl}/api/v1/emails/${id}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Accept': 'application/json',
      },
    });

    if (!res.ok) {
      throw new Error(`Sendrix Status Check Error: ${res.statusText}`);
    }

    return res.json();
  }
}
```

---

## 3. Ejemplo de Uso en Next.js (App Router / Server Action)

```typescript
import { SendrixClient } from '@/lib/sendrix';

const sendrix = new SendrixClient();

export async function sendWelcomeEmail(userEmail: string, userName: string) {
  try {
    const response = await sendrix.send({
      to: userEmail,
      subject: '¡Bienvenido a nuestra plataforma!',
      html: `<h1>Hola ${userName}</h1><p>Tu cuenta ha sido creada con éxito.</p>`,
      from_name: 'Equipo Sendrix',
    });

    return { success: true, id: response.id };
  } catch (error) {
    console.error('Error al enviar email:', error);
    return { success: false, error: (error as Error).message };
  }
}
```
