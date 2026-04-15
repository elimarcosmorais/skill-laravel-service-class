# Pest Testing Guide — Service Classes

Padrões para testar services do projeto usando Pest PHP.

---

## Sumário

- Estrutura de diretórios de teste
- Anatomia de um teste de service
- Mocking de dependências
- Testando exceções
- Testando DTOs e ValueObjects
- Testando Enums
- Padrões comuns

---

## Estrutura de Diretórios

Espelhar a estrutura de `app/Services/` em `tests/Unit/Services/`:

```
tests/
├── Unit/
│   └── Services/
│       └── Payment/
│           ├── PaymentServiceTest.php
│           ├── DTOs/
│           │   └── PaymentResultTest.php
│           ├── Enums/
│           │   └── PaymentStatusTest.php
│           └── ValueObjects/
│               └── MoneyTest.php
```

---

## Anatomia de um Teste de Service

### Estrutura Básica

```php
<?php

declare(strict_types=1);

use App\Services\Payment\PaymentService;
use App\Services\Payment\Contracts\PaymentGateway;
use App\Services\Payment\DTOs\PaymentResult;
use App\Services\Payment\Enums\PaymentStatus;
use App\Services\Payment\Exceptions\PaymentFailedException;
use App\Services\Payment\ValueObjects\Money;

describe('PaymentService', function () {

    describe('processPayment', function () {

        it('processes a valid payment successfully', function () {
            // Arrange
            $gateway = Mockery::mock(PaymentGateway::class);
            $gateway->shouldReceive('charge')
                ->once()
                ->with(Mockery::on(fn (Money $m) => $m->amount === 5000))
                ->andReturn([
                    'id'       => 'txn_123',
                    'status'   => 'approved',
                    'amount'   => 5000,
                    'currency' => 'BRL',
                ]);

            $service = new PaymentService(gateway: $gateway);

            // Act
            $result = $service->processPayment(new Money(5000));

            // Assert
            expect($result)
                ->toBeInstanceOf(PaymentResult::class)
                ->transactionId->toBe('txn_123')
                ->status->toBe(PaymentStatus::Approved)
                ->amount->amount->toBe(5000);
        });

        it('throws exception when payment is declined', function () {
            $gateway = Mockery::mock(PaymentGateway::class);
            $gateway->shouldReceive('charge')
                ->once()
                ->andReturn([
                    'id'     => 'txn_456',
                    'status' => 'declined',
                    'amount' => 5000,
                    'currency' => 'BRL',
                    'error'  => 'Insufficient funds',
                ]);

            $service = new PaymentService(gateway: $gateway);

            expect(fn () => $service->processPayment(new Money(5000)))
                ->toThrow(PaymentFailedException::class);
        });

        it('throws exception for zero amount', function () {
            $gateway = Mockery::mock(PaymentGateway::class);
            $service = new PaymentService(gateway: $gateway);

            expect(fn () => $service->processPayment(Money::zero()))
                ->toThrow(InvalidArgumentException::class);
        });
    });
});
```

### Convenções de Nomeação

- Arquivo: `{ServiceName}Test.php`
- Describe blocks: agrupar por método público
- It blocks: descrever o comportamento esperado em inglês

```php
describe('ServiceName', function () {
    describe('methodName', function () {
        it('does X when Y', function () { ... });
        it('throws Z when W', function () { ... });
        it('returns null when not found', function () { ... });
    });
});
```

---

## Mocking de Dependências

### Com Mockery (padrão do Pest)

```php
// Mock simples
$repo = Mockery::mock(UserRepository::class);
$repo->shouldReceive('findById')
    ->with(1)
    ->once()
    ->andReturn(new User(['id' => 1, 'name' => 'John']));

// Mock com callback de validação
$repo->shouldReceive('save')
    ->once()
    ->with(Mockery::on(function (User $user) {
        return $user->name === 'John' && $user->email === 'john@example.com';
    }))
    ->andReturn(true);

// Mock que lança exceção
$gateway->shouldReceive('charge')
    ->once()
    ->andThrow(new ConnectionException('Timeout'));
```

### Instanciar o Service nos Testes

Criar o service manualmente injetando mocks — não usar o container:

```php
// ✅ Correto — controle total das dependências
$service = new PaymentService(
    gateway: $mockGateway,
    logger: $mockLogger,
);

// ❌ Evitar — perde controle, pode usar implementações reais
$service = app(PaymentService::class);
```

### BeforeEach para Setup Compartilhado

```php
describe('OrderService', function () {

    beforeEach(function () {
        $this->repository = Mockery::mock(OrderRepository::class);
        $this->paymentGateway = Mockery::mock(PaymentGateway::class);
        $this->service = new OrderService(
            repository: $this->repository,
            paymentGateway: $this->paymentGateway,
        );
    });

    it('creates an order', function () {
        $this->repository->shouldReceive('save')->once()->andReturn(true);
        // ...
    });
});
```

---

## Testando Exceções

```php
// Verificar tipo da exceção
it('throws when order is empty', function () {
    expect(fn () => $this->service->process(new Order()))
        ->toThrow(EmptyOrderException::class);
});

// Verificar mensagem da exceção
it('throws with descriptive message', function () {
    expect(fn () => $this->service->process(new Order()))
        ->toThrow(EmptyOrderException::class, 'Order has no items');
});

// Verificar exceção com factory method
it('throws declined exception with transaction id', function () {
    $exception = PaymentFailedException::declined('txn_789', 'Card expired');

    expect($exception)
        ->toBeInstanceOf(PaymentFailedException::class)
        ->getMessage()->toContain('txn_789')
        ->getMessage()->toContain('Card expired');
});
```

---

## Testando DTOs

```php
describe('PaymentResult', function () {

    it('creates from gateway response', function () {
        $response = [
            'id'       => 'txn_123',
            'status'   => 'approved',
            'amount'   => 5000,
            'currency' => 'BRL',
        ];

        $result = PaymentResult::fromGatewayResponse($response);

        expect($result)
            ->transactionId->toBe('txn_123')
            ->status->toBe(PaymentStatus::Approved)
            ->amount->amount->toBe(5000)
            ->amount->currency->toBe('BRL')
            ->errorMessage->toBeNull();
    });

    it('includes error message when present', function () {
        $response = [
            'id'       => 'txn_456',
            'status'   => 'declined',
            'amount'   => 5000,
            'currency' => 'BRL',
            'error'    => 'Insufficient funds',
        ];

        $result = PaymentResult::fromGatewayResponse($response);

        expect($result)
            ->status->toBe(PaymentStatus::Declined)
            ->errorMessage->toBe('Insufficient funds');
    });
});
```

---

## Testando ValueObjects

```php
describe('Money', function () {

    it('creates with amount and currency', function () {
        $money = new Money(5000, 'BRL');

        expect($money)
            ->amount->toBe(5000)
            ->currency->toBe('BRL');
    });

    it('defaults to BRL currency', function () {
        expect(new Money(1000))->currency->toBe('BRL');
    });

    it('creates zero value', function () {
        expect(Money::zero())->amount->toBe(0);
    });

    it('adds two money objects', function () {
        $a = new Money(3000);
        $b = new Money(2000);

        expect($a->add($b))->amount->toBe(5000);
    });

    it('rejects negative amounts', function () {
        expect(fn () => new Money(-100))
            ->toThrow(InvalidArgumentException::class);
    });

    it('rejects adding different currencies', function () {
        $brl = new Money(1000, 'BRL');
        $usd = new Money(1000, 'USD');

        expect(fn () => $brl->add($usd))
            ->toThrow(InvalidArgumentException::class, 'different currencies');
    });
});
```

---

## Testando Enums

```php
describe('PaymentStatus', function () {

    it('has correct backing values', function () {
        expect(PaymentStatus::Pending->value)->toBe('pending');
        expect(PaymentStatus::Approved->value)->toBe('approved');
    });

    it('returns localized labels', function () {
        expect(PaymentStatus::Pending->label())->toBe('Pendente');
        expect(PaymentStatus::Approved->label())->toBe('Aprovado');
    });

    it('identifies final statuses', function () {
        expect(PaymentStatus::Approved->isFinal())->toBeTrue();
        expect(PaymentStatus::Pending->isFinal())->toBeFalse();
    });

    it('creates from string value', function () {
        expect(PaymentStatus::from('approved'))->toBe(PaymentStatus::Approved);
    });

    it('returns null for invalid value with tryFrom', function () {
        expect(PaymentStatus::tryFrom('invalid'))->toBeNull();
    });
});
```

---

## Testes com Integração Laravel (quando necessário)

Para services que interagem com banco ou cache, usar testes de integração:

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

describe('UserService (integration)', function () {

    it('creates a user in the database', function () {
        $service = app(UserService::class);

        $user = $service->createUser(
            name: 'John Doe',
            email: 'john@example.com',
        );

        expect($user)->toBeInstanceOf(User::class);
        $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
    });
});
```

Preferir testes unitários com mocks. Usar integração apenas quando o
comportamento do banco faz parte da lógica sendo testada.
