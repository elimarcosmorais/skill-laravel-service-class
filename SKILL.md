---
name: laravel-service-class
description: >
  Padroniza criação, revisão e refatoração de classes de serviço (Services) em
  Laravel 13.x com PHP 8.4. Cobre estrutura de diretórios, classes auxiliares
  (Enums, Exceptions, DTOs, ValueObjects, Repositories, Support), registro no
  AppServiceProvider (singleton vs scoped vs bind), e geração de testes com Pest.
  Acionar sempre que o usuário mencionar "service", "serviço", "criar service",
  "refatorar service", "testar service", "registrar service", "service provider",
  "singleton vs scoped", "bind service", ou qualquer criação/modificação de classe
  em app/Services/. Também acionar quando o usuário pedir para extrair lógica de
  negócio de controllers para services.
---

# Laravel Service Class

Padrões para criação e revisão de Service classes no Laravel 13.x com PHP 8.4.

---

## Estrutura de Diretórios

Cada domínio de serviço ocupa um diretório próprio em `app/Services/`:

```
app/Services/
├── Payment/
│   ├── DTOs/
│   │   └── PaymentResult.php
│   ├── Enums/
│   │   └── PaymentStatus.php
│   ├── Exceptions/
│   │   └── PaymentFailedException.php
│   ├── Repositories/
│   │   └── PaymentRepository.php
│   ├── Support/
│   │   └── PaymentCalculator.php
│   ├── ValueObjects/
│   │   └── Money.php
│   └── PaymentService.php
```

Subdiretórios são criados sob demanda — incluir apenas os que o serviço precisa.
Tipos de classes auxiliares permitidos:

| Subdiretório     | Conteúdo                                   | Exemplo                   |
|------------------|--------------------------------------------|---------------------------|
| `DTOs/`          | Data Transfer Objects (input/output)       | `PaymentResult.php`       |
| `Enums/`         | Enums tipados com BackedEnum               | `PaymentStatus.php`       |
| `Exceptions/`    | Exceções específicas do domínio            | `PaymentFailedException`  |
| `Repositories/`  | Abstração de acesso a dados                | `PaymentRepository.php`   |
| `Support/`       | Helpers, calculadores, formatadores        | `PaymentCalculator.php`   |
| `ValueObjects/`  | Objetos imutáveis de valor                 | `Money.php`               |
| `Constants/`     | Constantes agrupadas do domínio            | `PaymentLimits.php`       |

---

## Anatomia da Service Class

### Regras Obrigatórias

1. `declare(strict_types=1)` no topo
2. `final class` — services não são projetados para herança
3. Dependências injetadas via construtor com `readonly`
4. Métodos públicos com tipagem completa (parâmetros e retorno)
5. Nomes de métodos em inglês, camelCase, com verbo de ação

### Quando Usar `final readonly class`

- **`final readonly class`**: quando o serviço não possui estado mutável (todas as
  propriedades são definidas no construtor e nunca mudam). Caso mais comum.
- **`final class`**: quando o serviço precisa de estado interno mutável (cache em
  memória, contadores, buffers).

### Template Base

```php
<?php

declare(strict_types=1);

namespace App\Services\DomainName;

final readonly class DomainNameService
{
    public function __construct(
        private DependencyA $dependencyA,
        private DependencyB $dependencyB,
    ) {}

    /**
     * Descrição breve da operação.
     *
     * DocBlock só quando agrega valor além da tipagem nativa.
     *
     * @param  array<int, Item>  $items  ← tipar arrays internos
     * @throws DomainException           ← declarar exceções lançadas
     */
    public function execute(array $items): ResultDTO
    {
        // Early return para pré-condições
        // Lógica principal
        // Retorno tipado
    }
}
```

### Padrão para Métodos Internos

Extrair lógica complexa em métodos `private` com nomes descritivos:

```php
public function processOrder(Order $order): OrderResult
{
    $this->ensureOrderIsProcessable($order);
    $total = $this->calculateTotal($order);
    $payment = $this->chargePayment($order, $total);

    return new OrderResult(order: $order, payment: $payment);
}

private function ensureOrderIsProcessable(Order $order): void
{
    if ($order->isCompleted()) {
        throw new OrderAlreadyCompletedException($order->id);
    }

    if (! $order->hasItems()) {
        throw new EmptyOrderException($order->id);
    }
}

private function calculateTotal(Order $order): Money
{
    return $order->items
        ->reduce(
            fn (Money $carry, OrderItem $item) => $carry->add($item->subtotal()),
            Money::zero(),
        );
}
```

---

## Classes Auxiliares

### DTOs (Data Transfer Objects)

Objetos para transportar dados entre camadas. Sempre `final readonly class`:

```php
<?php

declare(strict_types=1);

namespace App\Services\Payment\DTOs;

final readonly class PaymentResult
{
    public function __construct(
        public string $transactionId,
        public PaymentStatus $status,
        public Money $amount,
        public ?string $errorMessage = null,
    ) {}

    /**
     * Factory method para construção a partir de resposta externa.
     *
     * @param  array<string, mixed>  $response
     */
    public static function fromGatewayResponse(array $response): self
    {
        return new self(
            transactionId: $response['id'],
            status: PaymentStatus::from($response['status']),
            amount: new Money($response['amount'], $response['currency']),
            errorMessage: $response['error'] ?? null,
        );
    }
}
```

### Enums

Sempre `BackedEnum` com `string` ou `int`. Incluir métodos utilitários no enum:

```php
<?php

declare(strict_types=1);

namespace App\Services\Payment\Enums;

enum PaymentStatus: string
{
    case Pending = 'pending';
    case Approved = 'approved';
    case Declined = 'declined';
    case Refunded = 'refunded';

    public function label(): string
    {
        return match ($this) {
            self::Pending  => 'Pendente',
            self::Approved => 'Aprovado',
            self::Declined => 'Recusado',
            self::Refunded => 'Reembolsado',
        };
    }

    public function isFinal(): bool
    {
        return in_array($this, [self::Approved, self::Declined, self::Refunded]);
    }
}
```

### Exceptions

Exceções específicas do domínio, estendendo `RuntimeException` ou `DomainException`:

```php
<?php

declare(strict_types=1);

namespace App\Services\Payment\Exceptions;

use RuntimeException;

final class PaymentFailedException extends RuntimeException
{
    public static function declined(string $transactionId, string $reason): self
    {
        return new self(
            "Payment {$transactionId} declined: {$reason}"
        );
    }

    public static function gatewayUnavailable(string $gateway): self
    {
        return new self(
            "Payment gateway '{$gateway}' is unavailable"
        );
    }
}
```

### ValueObjects

Objetos imutáveis que representam um valor do domínio:

```php
<?php

declare(strict_types=1);

namespace App\Services\Payment\ValueObjects;

use InvalidArgumentException;

final readonly class Money
{
    public function __construct(
        public int $amount,
        public string $currency = 'BRL',
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
    }

    public static function zero(string $currency = 'BRL'): self
    {
        return new self(0, $currency);
    }

    public function add(self $other): self
    {
        $this->ensureSameCurrency($other);

        return new self($this->amount + $other->amount, $this->currency);
    }

    private function ensureSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Cannot operate on different currencies: {$this->currency} vs {$other->currency}"
            );
        }
    }
}
```

---

## Registro no AppServiceProvider

Guia completo de decisão em **`references/container-binding-guide.md`**.

### Resumo Rápido

| Binding     | Quando usar                                         |
|-------------|-----------------------------------------------------|
| `singleton` | Sem estado mutável E reutilizado em vários pontos   |
| `scoped`    | Estado mutável por request OU isolamento por request |
| `bind`      | Sem estado, criação barata, uso pontual             |
| Nenhum      | Sem interface, sem config dinâmica — auto-resolve    |

### Regra de Ouro

A maioria dos services no projeto **não precisa de registro manual**. O container
do Laravel resolve automaticamente classes concretas com dependências tipadas.
Registrar manualmente apenas quando:

1. O service precisa de configuração dinâmica no construtor (API keys, configs)
2. O service precisa garantir instância única (`singleton`) ou por request (`scoped`)
3. O service usa uma interface e precisa de binding explícito (opcional, não obrigatório)

```php
// AppServiceProvider.php — boot() ou register()

// Singleton — config dinâmica justifica o registro
$this->app->singleton(
    NotificationService::class,
    fn () => new NotificationService(
        channel: config('notifications.default_channel'),
        retryAttempts: (int) config('notifications.retry_attempts', 3),
    ),
);
```

---

## Testabilidade

Services projetados para teste devem:

1. Receber todas as dependências pelo construtor (permite mocking)
2. Ter métodos com entrada e saída claras (testáveis isoladamente)
3. Lançar exceções tipadas (verificáveis em testes)

Guia completo de testes com Pest em **`references/pest-testing-guide.md`**.

### Exemplo Rápido

```php
it('processes a valid payment', function () {
    $gateway = Mockery::mock(PaymentGateway::class);
    $gateway->shouldReceive('charge')
        ->once()
        ->andReturn(['id' => 'txn_123', 'status' => 'approved', 'amount' => 5000, 'currency' => 'BRL']);

    $service = new PaymentService(gateway: $gateway);
    $result = $service->processPayment(new Money(5000));

    expect($result)
        ->toBeInstanceOf(PaymentResult::class)
        ->status->toBe(PaymentStatus::Approved)
        ->transactionId->toBe('txn_123');
});
```

---

## Referências

| Arquivo                               | Conteúdo                                    | Carregar quando                        |
|---------------------------------------|---------------------------------------------|----------------------------------------|
| `references/container-binding-guide.md` | Decisão singleton vs scoped vs bind         | Registrando service no container       |
| `references/pest-testing-guide.md`      | Padrões de teste com Pest para services     | Criando testes para services           |
| `references/service-class-template.md`  | Template completo com exemplo real          | Criando service do zero                |

---

## Documentação de Uso para AI (Obrigatório)

Toda service class DEVE ter UM ÚNICO arquivo `.md` em `.claude/docs/services/`. O nome do
arquivo é a conversão do nome da classe de PascalCase para kebab-case.

**Regra de nomenclatura**: cada letra maiúscula (exceto a primeira) é precedida por
hífen, tudo minúsculo.

- `PaymentService.php` → `payment-service.md`
- `RedirectService.php` → `redirect-service.md`
- `S3FileUploader.php` → `s3-file-uploader.md`
- `UserAuthenticationHandler.php` → `user-authentication-handler.md`

**Propósito**: instruções para agentes AI (Claude Code) consumirem a service.
Não é documentação para desenvolvedores — é referência compacta, sem didática,
otimizada para economia de tokens.

### Regra Fundamental: Um Arquivo, Uma Documentação

NUNCA gerar documentos separados para classes auxiliares. NUNCA criar seções
independentes que tratem a classe auxiliar como autônoma.

A documentação é sobre a classe principal. Classes auxiliares são documentadas
DENTRO do contexto da classe principal, como extensão da sua API.

### Localização

```
.claude/docs/services/
└── payment-service.md     ← arquivo único de instruções AI

app/Services/Payment/
├── PaymentService.php
├── DTOs/
│   └── PaymentResult.php
├── Support/
│   └── PaymentQueryBuilder.php
└── ...
```

### Como Identificar Classes Auxiliares

Uma classe auxiliar é qualquer classe cujos métodos públicos são acessíveis a
partir da classe principal. Isso acontece quando:

- Um método público da classe principal retorna uma instância da classe auxiliar
- Uma propriedade pública da classe principal é uma instância da classe auxiliar
- A classe principal usa composição e delega operações para a auxiliar

### Estrutura Obrigatória do Arquivo

Seguir EXATAMENTE esta estrutura. Não alterar a ordem, não adicionar seções, não
remover seções.

```markdown
# {ServiceName}

Namespace: `App\Services\DomainName`

Dependências: `TipoA`, `TipoB` (via construtor)

## Métodos

### metodoSimples(TipoParam $param): TipoRetorno

Descrição direta em uma ou duas frases.

- `$param` — descrição funcional
- Returns: descrição do retorno
- Throws: `SpecificException` quando [condição]

### metodoQueRetornaAuxiliar(): ClasseAuxiliar

Descrição direta do método.

- Returns: instância de `ClasseAuxiliar` com os métodos abaixo

#### Métodos acessíveis via ClasseAuxiliar

#### metodoAuxiliarA(Tipo $param): TipoRetorno

Descrição.

- `$param` — descrição
- Returns: descrição

#### metodoAuxiliarB(): TipoRetorno

Descrição.

- Returns: descrição

## Uso Rápido

A variável que recebe a instância do service DEVE usar o nome da classe sem o
sufixo `Service`, em camelCase. Exemplos:
- `UserMenuService` → `$userMenu`
- `PaymentService` → `$payment`
- `S3FileUploaderService` → `$s3FileUploader`
- `OrderProcessingService` → `$orderProcessing`

\```php
$domainName = app(DomainNameService::class);

// Método simples
$resultado = $domainName->metodoSimples($valor);

// Método com classe auxiliar (encadeamento)
$resultado = $domainName->metodoQueRetornaAuxiliar()
    ->metodoAuxiliarA($param)
    ->metodoAuxiliarB();
\```
```

### Regras de Escrita

1. **Um único arquivo** — NUNCA gerar múltiplos `.md` para o mesmo service
2. **Sem explicações** — apenas assinaturas, tipos e comportamento
3. **Uma linha por conceito** — sem parágrafos
4. **Somente métodos públicos** — métodos `private`/`protected` são internos, NUNCA documentá-los
5. **Métodos mágicos** — não documentar (`__construct`, `__toString`) exceto se tiverem comportamento relevante para uso externo
6. **Getters/setters triviais** — não documentar se apenas retornam/atribuem uma propriedade
7. **Classes auxiliares como submétodos** — documentar métodos públicos de classes auxiliares DENTRO do método da classe principal que as expõe, usando `####` e a seção "Métodos acessíveis via {ClasseAuxiliar}"
8. **Seção "Uso Rápido"** — snippet mínimo cobrindo instanciação e todos os padrões de uso. A variável DEVE usar o nome da classe sem `Service`, em camelCase (ex: `$userMenu = app(UserMenuService::class)`)
9. **Resolver via container** — sempre mostrar `app(Class::class)`, nunca `new`
10. **Exceções** — listar quais e quando são lançadas (uma linha cada)
11. **Sem formatação decorativa** — sem badges, emojis, seções "Introdução" ou "Visão Geral"

### Formatação de Parâmetros e Retorno

- Parâmetros: liste com travessão, um por linha: `- \`$param\` — descrição funcional`
- Retorno: sempre inclua: `- Returns: descrição`
- Exceções: `- Throws: \`ExceptionType\` quando [condição]`
- Blocos de código: sempre com \```php

### Exemplo Real: payment-service.md

```markdown
# PaymentService

Namespace: `App\Services\Payment`

Dependências: `PaymentGateway`, `PaymentRepository` (via construtor)

## Métodos

### processPayment(Money $amount): PaymentResult

Processa pagamento via gateway configurado. Retry automático.

- `$amount` — valor a cobrar
- Returns: `PaymentResult` com `transactionId`, `status`, `amount`
- Throws: `PaymentFailedException` quando gateway recusa após retries

### find(string $transactionId): ?PaymentResult

Busca resultado de pagamento pelo ID da transação.

- `$transactionId` — identificador da transação
- Returns: `PaymentResult` ou `null` se não encontrado

### query(): PaymentQueryBuilder

Cria um query builder para busca filtrada de pagamentos.

- Returns: instância de `PaymentQueryBuilder` com os métodos abaixo

#### Métodos acessíveis via PaymentQueryBuilder

#### whereStatus(PaymentStatus $status): self

Filtra por status do pagamento.

- `$status` — status para filtrar
- Returns: `self` (encadeável)

#### whereDateBetween(Carbon $start, Carbon $end): self

Filtra por intervalo de datas.

- `$start` — data inicial
- `$end` — data final
- Returns: `self` (encadeável)

#### get(): Collection

Executa a query e retorna os resultados.

- Returns: `Collection` de `PaymentResult`

## Uso Rápido

\```php
$payment = app(PaymentService::class);

// Pagamento direto
$result = $payment->processPayment(new Money(5000));

// Busca por ID
$found = $payment->find('txn_123');

// Query filtrada
$payments = $payment->query()
    ->whereStatus(PaymentStatus::Approved)
    ->whereDateBetween(now()->subDays(7), now())
    ->get();
\```
```

---

## Revisão de Service Class Existente

Quando o usuário pedir para **revisar, melhorar ou refatorar** uma service class já
existente, seguir OBRIGATORIAMENTE este fluxo:

### Fluxo de Revisão

1. **Analisar** — ler o código atual completo da classe e suas auxiliares
2. **Apresentar diagnóstico** — listar para o usuário TODAS as melhorias e correções
   identificadas, organizadas por categoria:
   - 🔴 **Correções** — problemas que violam as regras desta skill (falta de
     `strict_types`, classe não `final`, tipagem incompleta, etc.)
   - 🟡 **Melhorias** — refatorações que elevam a qualidade (extração de métodos,
     DTOs faltantes, exceções genéricas que deveriam ser específicas, etc.)
   - 🟢 **Sugestões** — ajustes opcionais de estilo ou organização
3. **Aguardar autorização** — o usuário DEVE aprovar antes de qualquer alteração no
   código. Perguntar explicitamente: _"Posso prosseguir com essas alterações?"_
4. **Aplicar** — somente após aprovação, gerar o código atualizado

### Regra Fundamental

**NUNCA alterar código de service class existente sem apresentar o diagnóstico
primeiro e receber autorização explícita do usuário.** Mesmo que as correções sejam
óbvias ou triviais, o usuário precisa saber o que será alterado.

---

## Checklist Pré-Entrega

Verificar antes de entregar qualquer service:

- [ ] `declare(strict_types=1)` presente
- [ ] Classe marcada como `final` (ou `final readonly`)
- [ ] Dependências injetadas via construtor com `readonly`
- [ ] Métodos públicos com tipagem completa
- [ ] Nomes em inglês, camelCase, verbos de ação
- [ ] Exceções específicas do domínio (não genéricas)
- [ ] DTOs para entrada/saída complexa
- [ ] Enums para valores finitos
- [ ] Diretório do service criado em `app/Services/DomainName/`
- [ ] Registro no AppServiceProvider justificado (ou auto-resolve documentado)
- [ ] Arquivo `{service-name}.md` (kebab-case) com instruções AI criado no diretório do service
- [ ] Testes com Pest quando solicitados
