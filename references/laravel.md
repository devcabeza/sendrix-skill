# Sendrix — Guía de Integración para Laravel

Esta guía proporciona una integración completa de Sendrix como un transporte de correo nativo (`Mail::mailer('sendrix')`), permitiendo usar `Mail::to()->send()` o `Mail::to()->queue()` sin modificar tus Mailables ni lógica de negocio.

---

## 1. Configuración de Entorno

### `.env` y `.env.example`
```dotenv
MAIL_MAILER=sendrix
SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
SENDRIX_KEY=sndx_live_tu_api_key_aqui
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
    // ... otros mailers
],
```

---

## 2. Transporte Personalizado (`SendrixTransport`)

Crea `app/Mail/Transports/SendrixTransport.php`:

```php
<?php

declare(strict_types=1);

namespace App\Mail\Transports;

use Exception;
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

        $recipients = array_map(fn (Address $addr) => $addr->getAddress(), $email->getTo());
        $primaryTo = $recipients[0] ?? '';

        $fromList = $email->getFrom();
        $fromName = count($fromList) > 0 ? $fromList[0]->getName() : null;

        $replyToList = $email->getReplyTo();
        $replyTo = count($replyToList) > 0 ? $replyToList[0]->getAddress() : null;

        $cc = array_map(fn (Address $a) => $a->getAddress(), $email->getCc());
        $bcc = array_map(fn (Address $a) => $a->getAddress(), $email->getBcc());

        $payload = [
            'to' => $primaryTo,
            'subject' => $email->getSubject() ?? 'Sin Asunto',
            'html' => $email->getHtmlBody() ?? ($email->getTextBody() ?? ''),
            'text' => $email->getTextBody(),
            'from_name' => $fromName !== '' ? $fromName : null,
            'reply_to' => $replyTo,
            'cc' => count($cc) > 0 ? array_values($cc) : null,
            'bcc' => count($bcc) > 0 ? array_values($bcc) : null,
        ];

        $response = Http::withToken($this->key)
            ->baseUrl($this->baseUrl)
            ->asJson()
            ->acceptJson()
            ->post('/api/v1/send', array_filter($payload, fn ($v) => $v !== null));

        if ($response->status() === 429) {
            $retryAfter = (int) ($response->json('retry_after_seconds') ?? $response->header('Retry-After') ?? 60);
            throw new Exception("Sendrix Rate Limit Exceeded. Retry after {$retryAfter}s");
        }

        if (! $response->successful()) {
            throw new Exception('Sendrix Delivery Failed: '.$response->body());
        }
    }

    public function __toString(): string
    {
        return 'sendrix';
    }
}
```

---

## 3. Registro en `AppServiceProvider`

En `app/Providers/AppServiceProvider.php`:

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

## 4. Cola y Resiliencia ante Rate Limit (5 req/min)

Sendrix procesa hasta 5 solicitudes por minuto por proyecto. **Siempre** envía los correos a través de colas en segundo plano (`Mail::to()->queue()`) o un Job dedicado:

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
use Throwable;

final class SendSendrixEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 5;
    public int $backoff = 65; // Superior al minuto de ventana del rate-limit

    public function __construct(
        public readonly string $to,
        public readonly string $subject,
        public readonly string $html,
    ) {
        $this->onQueue('emails');
    }

    public function handle(): void
    {
        $response = Http::withToken(config('services.sendrix.key'))
            ->baseUrl(config('services.sendrix.base_url'))
            ->post('/api/v1/send', [
                'to' => $this->to,
                'subject' => $this->subject,
                'html' => $this->html,
            ]);

        if ($response->status() === 429) {
            $retryAfter = (int) ($response->json('retry_after_seconds') ?? 60);
            $this->release($retryAfter + 5);
            return;
        }

        if (! $response->successful()) {
            throw new \RuntimeException('Error de envío en Sendrix: '.$response->body());
        }
    }
}
```

---

## 5. Pruebas Automatizadas con Pest

Para simular Sendrix sin consumir la cuota de la API:

```php
use Illuminate\Support\Facades\Http;

it('envía email de bienvenida a través de Sendrix', function () {
    Http::fake([
        '*/api/v1/send' => Http::response([
            'success' => true,
            'id' => 'sndx_msg_fake123',
            'log_id' => '01JFAKE00000000000',
        ], 200),
    ]);

    // Ejecuta tu acción / mailable
    // ...

    Http::assertSent(function ($request) {
        return str_contains($request->url(), '/api/v1/send')
            && $request['to'] === 'usuario@ejemplo.com'
            && $request['subject'] === '¡Bienvenido!';
    });
});
```
