# Sendrix — Guía de integración para Laravel

Integra Sendrix como un **transporte de correo nativo** (`Mail::mailer('sendrix')`) para usar `Mail::to()->send()` / `queue()` sin cambiar tus Mailables, o como **cliente HTTP** para usar plantillas, lotes y estados.

---

## 1. Configuración

### `.env` / `.env.example`

```dotenv
MAIL_MAILER=sendrix
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
SENDRIX_WEBHOOK_SECRET=tu_secreto_de_webhook
```

### `config/services.php`

```php
'sendrix' => [
    'key' => env('SENDRIX_KEY'),
    'base_url' => env('SENDRIX_BASE_URL', 'https://sendrix.alejandrocabeza.dev'),
    'webhook_secret' => env('SENDRIX_WEBHOOK_SECRET'),
],
```

### `config/mail.php`

```php
'mailers' => [
    'sendrix' => [
        'transport' => 'sendrix',
    ],
    // ...
],
```

---

## 2. Transporte personalizado (`SendrixTransport`)

Crea `app/Mail/Transports/SendrixTransport.php`:

```php
<?php

declare(strict_types=1);

namespace App\Mail\Transports;

use RuntimeException;
use Illuminate\Support\Facades\Http;
use Symfony\Component\Mailer\SentMessage;
use Symfony\Component\Mailer\Transport\AbstractTransport;
use Symfony\Component\Mime\Address;
use Symfony\Component\Mime\Email;
use Symfony\Component\Mime\MessageConverter;

final class SendrixTransport extends AbstractTransport
{
    public function __construct(
        private readonly string $key,
        private readonly string $baseUrl = 'https://sendrix.alejandrocabeza.dev',
    ) {
        parent::__construct();
    }

    protected function doSend(SentMessage $message): void
    {
        $email = MessageConverter::toEmail($message->getOriginalMessage());

        $to = array_map(fn (Address $a) => $a->getAddress(), $email->getTo());
        $cc = array_map(fn (Address $a) => $a->getAddress(), $email->getCc());
        $bcc = array_map(fn (Address $a) => $a->getAddress(), $email->getBcc());
        $from = $email->getFrom()[0] ?? null;
        $replyTo = $email->getReplyTo()[0] ?? null;

        $payload = array_filter([
            'to' => $to[0] ?? '',
            'subject' => $email->getSubject() ?? 'Sin Asunto',
            'html' => $email->getHtmlBody() ?? ($email->getTextBody() ?? ''),
            'text' => $email->getTextBody(),
            'from_name' => $from?->getName(),
            'reply_to' => $replyTo?->getAddress(),
            'cc' => $cc ?: null,
            'bcc' => $bcc ?: null,
            'attachments' => $this->attachments($email),
        ], fn ($v) => $v !== null && $v !== '' && $v !== []);

        $response = Http::withToken($this->key)
            ->baseUrl($this->baseUrl)
            ->asJson()
            ->acceptJson()
            ->timeout(15)
            ->post('/api/v1/send', $payload);

        // Errores permanentes: no reintentar
        if (in_array($response->status(), [401, 404, 422], true)) {
            throw new RuntimeException('Sendrix permanente: '.$response->body());
        }

        // 429 (cuota diaria) o 5xx: deja que el job reintente
        if (! $response->successful()) {
            throw new RuntimeException('Sendrix temporal ('.$response->status().'): '.$response->body());
        }

        $message->setMessageId((string) ($response->json('resend_id') ?? $response->json('id')));
    }

    /**
     * @return array<int, array{filename: string, content: string, content_type?: string}>
     */
    private function attachments(Email $email): array
    {
        $items = [];

        foreach ($email->getAttachments() as $attachment) {
            $items[] = array_filter([
                'filename' => $attachment->getFilename() ?? 'adjunto',
                'content' => base64_encode($attachment->getBody()),
                'content_type' => $attachment->getContentType(),
            ]);
        }

        return $items;
    }

    public function __toString(): string
    {
        return 'sendrix';
    }
}
```

---

## 3. Registro en `AppServiceProvider`

```php
use App\Mail\Transports\SendrixTransport;
use Illuminate\Support\Facades\Mail;

public function boot(): void
{
    Mail::extend('sendrix', function () {
        return new SendrixTransport(
            key: (string) config('services.sendrix.key', ''),
            baseUrl: (string) config('services.sendrix.base_url', 'https://sendrix.alejandrocabeza.dev'),
        );
    });
}
```

---

## 4. Colas y resiliencia

- Envía siempre en segundo plano: `Mail::to($user)->queue(new WelcomeMail(...))` o un Job dedicado con `onQueue('emails')` monitorizado por Horizon.
- Ante `429`/`5xx`, deja que el job reintente con backoff **≥ 60 s**. No hagas busy-loop.
- Persiste el `id` (log id) para consultar el estado después con `GET /api/v1/emails/{id}`.

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Http;

final class SendSendrixEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 5;

    /** @var array<int> */
    public array $backoff = [10, 30, 60, 120];

    public function __construct(
        public readonly string $to,
        public readonly string $subject,
        public readonly string $html,
    ) {
        $this->onQueue('emails'); // cola 'emails' de Horizon
    }

    public function handle(): void
    {
        $response = Http::withToken((string) config('services.sendrix.key'))
            ->baseUrl((string) config('services.sendrix.base_url'))
            ->acceptJson()
            ->timeout(15)
            ->post('/api/v1/send', [
                'to' => $this->to,
                'subject' => $this->subject,
                'html' => $this->html,
            ]);

        if ($response->status() === 429) {
            // Cuota diaria superada: reintenta más tarde (mañana)
            $this->release(3600);
            return;
        }

        if (! $response->successful()) {
            throw new \RuntimeException('Error de envío en Sendrix: '.$response->body());
        }

        // Guarda $response->json('id') para trazabilidad.
    }
}
```

Para envíos asíncronos dentro de Sendrix (sin Horizon propio), usa `async: true` y Sendrix devolverá `202`:

```php
Http::withToken(config('services.sendrix.key'))
    ->baseUrl(config('services.sendrix.base_url'))
    ->post('/api/v1/send', [
        'to' => $user->email,
        'template' => 'welcome',
        'variables' => ['name' => $user->name, 'company' => 'Acme'],
        'async' => true,
    ]);
```

---

## 5. Webhooks firmados

Registra una ruta sin CSRF y verifica `X-Sendrix-Signature`. Ver el código completo en [webhooks.md](webhooks.md).

```php
// routes/api.php
Route::post('/webhooks/sendrix', SendrixWebhookController::class)
    ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class]);
```

---

## 6. Pruebas con Pest

```php
use Illuminate\Support\Facades\Http;

it('envía un email de bienvenida a través de Sendrix', function () {
    Http::fake([
        '*/api/v1/send' => Http::response([
            'id' => '01923e77-0000-7000-8000-000000000001',
            'project_id' => 1,
            'status' => 'sent',
            'resend_id' => 'resend-msg-fake123',
            'from' => 'Mi App <notify@test.dev>',
            'to' => 'usuario@ejemplo.com',
            'subject' => '¡Bienvenido!',
            'sent_at' => now()->toIso8601String(),
        ], 200),
    ]);

    // Ejecuta tu acción / mailable…
    // Mail::to('usuario@ejemplo.com')->send(new WelcomeMail());

    Http::assertSent(fn ($request) => str_contains($request->url(), '/api/v1/send')
        && $request['to'] === 'usuario@ejemplo.com'
        && $request['subject'] === '¡Bienvenido!');
});
```

> Usa `sndx_test_*` y/o `sandbox: true` en entornos no productivos: Sendrix captura el correo sin enviarlo.
