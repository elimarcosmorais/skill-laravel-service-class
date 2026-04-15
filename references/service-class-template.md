# Service Class Template — Exemplo Completo

Exemplo real de um service completo com todas as classes auxiliares,
registro no AppServiceProvider e testes Pest.

---

## Sumário

- Exemplo completo: NotificationService
- Estrutura de diretórios
- Cada arquivo do service
- Registro no AppServiceProvider
- Testes Pest

---

## Exemplo: NotificationService

Service que envia notificações por múltiplos canais (email, SMS, push)
com retry automático e logging de falhas.

### Estrutura

```
app/Services/Notification/
├── DTOs/
│   └── NotificationPayload.php
├── Enums/
│   └── ChannelType.php
├── Exceptions/
│   └── NotificationDeliveryException.php
├── Support/
│   └── NotificationRetry.php
├── NotificationService.php
└── NotificationService.md       ← instruções AI
```

---

### Enums/ChannelType.php

```php
<?php

declare(strict_types=1);

namespace App\Services\Notification\Enums;

enum ChannelType: string
{
    case Email = 'email';
    case Sms = 'sms';
    case Push = 'push';

    public function label(): string
    {
        return match ($this) {
            self::Email => 'E-mail',
            self::Sms   => 'SMS',
            self::Push  => 'Push Notification',
        };
    }
}
```

---

### DTOs/NotificationPayload.php

```php
<?php

declare(strict_types=1);

namespace App\Services\Notification\DTOs;

use App\Services\Notification\Enums\ChannelType;

final readonly class NotificationPayload
{
    /**
     * @param  array<string, mixed>  $metadata
     */
    public function __construct(
        public string $recipient,
        public string $subject,
        public string $body,
        public ChannelType $channel,
        public array $metadata = [],
    ) {}

    /**
     * @param  array<string, mixed>  $data
     */
    public static function fromArray(array $data): self
    {
        return new self(
            recipient: $data['recipient'],
            subject: $data['subject'],
            body: $data['body'],
            channel: ChannelType::from($data['channel']),
            metadata: $data['metadata'] ?? [],
        );
    }
}
```

---

### Exceptions/NotificationDeliveryException.php

```php
<?php

declare(strict_types=1);

namespace App\Services\Notification\Exceptions;

use App\Services\Notification\Enums\ChannelType;
use RuntimeException;

final class NotificationDeliveryException extends RuntimeException
{
    public static function channelFailed(
        ChannelType $channel,
        string $recipient,
        string $reason,
    ): self {
        return new self(
            "Failed to deliver via {$channel->value} to '{$recipient}': {$reason}"
        );
    }

    public static function allRetriesExhausted(
        ChannelType $channel,
        int $attempts,
    ): self {
        return new self(
            "All {$attempts} retries exhausted for {$channel->value} channel"
        );
    }
}
```

---

### Support/NotificationRetry.php

```php
<?php

declare(strict_types=1);

namespace App\Services\Notification\Support;

use Throwable;

final readonly class NotificationRetry
{
    public function __construct(
        private int $maxAttempts = 3,
        private int $delayMs = 100,
    ) {}

    /**
     * Executa callback com retry automático.
     *
     * @template T
     * @param  callable(): T  $callback
     * @return T
     * @throws Throwable  Última exceção após esgotar tentativas
     */
    public function attempt(callable $callback): mixed
    {
        $lastException = null;

        for ($i = 1; $i <= $this->maxAttempts; $i++) {
            try {
                return $callback();
            } catch (Throwable $e) {
                $lastException = $e;

                if ($i < $this->maxAttempts) {
                    usleep($this->delayMs * 1000 * $i);
                }
            }
        }

        throw $lastException;
    }
}
```

---

### NotificationService.php

```php
<?php

declare(strict_types=1);

namespace App\Services\Notification;

use App\Services\Notification\DTOs\NotificationPayload;
use App\Services\Notification\Enums\ChannelType;
use App\Services\Notification\Exceptions\NotificationDeliveryException;
use App\Services\Notification\Support\NotificationRetry;
use Illuminate\Mail\Mailer;
use Psr\Log\LoggerInterface;

final readonly class NotificationService
{
    public function __construct(
        private Mailer $mailer,
        private NotificationRetry $retry,
        private LoggerInterface $logger,
    ) {}

    /**
     * Envia notificação pelo canal especificado no payload.
     *
     * @throws NotificationDeliveryException
     */
    public function send(NotificationPayload $payload): bool
    {
        try {
            return $this->retry->attempt(
                fn () => $this->dispatch($payload),
            );
        } catch (\Throwable $e) {
            $this->logger->error('Notification delivery failed', [
                'channel'   => $payload->channel->value,
                'recipient' => $payload->recipient,
                'error'     => $e->getMessage(),
            ]);

            throw NotificationDeliveryException::channelFailed(
                channel: $payload->channel,
                recipient: $payload->recipient,
                reason: $e->getMessage(),
            );
        }
    }

    private function dispatch(NotificationPayload $payload): bool
    {
        return match ($payload->channel) {
            ChannelType::Email => $this->sendEmail($payload),
            ChannelType::Sms   => $this->sendSms($payload),
            ChannelType::Push  => $this->sendPush($payload),
        };
    }

    private function sendEmail(NotificationPayload $payload): bool
    {
        $this->mailer->raw($payload->body, function ($message) use ($payload) {
            $message->to($payload->recipient)->subject($payload->subject);
        });

        return true;
    }

    private function sendSms(NotificationPayload $payload): bool
    {
        // Integração com serviço SMS (Twilio, AWS SNS, etc.)
        // Implementar conforme provider usado no projeto
        return true;
    }

    private function sendPush(NotificationPayload $payload): bool
    {
        // Integração com serviço Push (Firebase, OneSignal, etc.)
        // Implementar conforme provider usado no projeto
        return true;
    }
}
```

---

### Registro no AppServiceProvider

```php
// AppServiceProvider.php

use App\Services\Notification\NotificationService;
use App\Services\Notification\Support\NotificationRetry;

public function register(): void
{
    // Singleton — stateless, config imutável, usado em muitos pontos
    $this->app->singleton(
        NotificationService::class,
        fn () => new NotificationService(
            mailer: app('mailer'),
            retry: new NotificationRetry(
                maxAttempts: (int) config('notifications.retry_attempts', 3),
                delayMs: (int) config('notifications.retry_delay_ms', 100),
            ),
            logger: app(\Psr\Log\LoggerInterface::class),
        ),
    );
}
```

**Justificativa: `singleton`**
- Service é stateless (não acumula dados entre chamadas)
- Configuração é imutável (retry config, mailer)
- Usado em controllers, jobs e listeners — reutilização justifica singleton

---

### Testes Pest

```php
<?php

declare(strict_types=1);

use App\Services\Notification\DTOs\NotificationPayload;
use App\Services\Notification\Enums\ChannelType;
use App\Services\Notification\Exceptions\NotificationDeliveryException;
use App\Services\Notification\NotificationService;
use App\Services\Notification\Support\NotificationRetry;
use Illuminate\Mail\Mailer;
use Psr\Log\NullLogger;

describe('NotificationService', function () {

    beforeEach(function () {
        $this->mailer = Mockery::mock(Mailer::class);
        $this->retry = new NotificationRetry(maxAttempts: 2, delayMs: 0);
        $this->logger = new NullLogger();
    });

    describe('send', function () {

        it('delivers email notification successfully', function () {
            $this->mailer->shouldReceive('raw')->once();

            $service = new NotificationService(
                mailer: $this->mailer,
                retry: $this->retry,
                logger: $this->logger,
            );

            $payload = new NotificationPayload(
                recipient: 'user@example.com',
                subject: 'Test',
                body: 'Hello',
                channel: ChannelType::Email,
            );

            expect($service->send($payload))->toBeTrue();
        });

        it('retries on transient failure then succeeds', function () {
            $this->mailer->shouldReceive('raw')
                ->once()
                ->andThrow(new RuntimeException('Timeout'));
            $this->mailer->shouldReceive('raw')
                ->once();

            $service = new NotificationService(
                mailer: $this->mailer,
                retry: $this->retry,
                logger: $this->logger,
            );

            $payload = new NotificationPayload(
                recipient: 'user@example.com',
                subject: 'Test',
                body: 'Hello',
                channel: ChannelType::Email,
            );

            expect($service->send($payload))->toBeTrue();
        });

        it('throws after all retries exhausted', function () {
            $this->mailer->shouldReceive('raw')
                ->twice()
                ->andThrow(new RuntimeException('Timeout'));

            $service = new NotificationService(
                mailer: $this->mailer,
                retry: $this->retry,
                logger: $this->logger,
            );

            $payload = new NotificationPayload(
                recipient: 'user@example.com',
                subject: 'Test',
                body: 'Hello',
                channel: ChannelType::Email,
            );

            expect(fn () => $service->send($payload))
                ->toThrow(NotificationDeliveryException::class);
        });
    });
});
```
