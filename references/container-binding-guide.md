# Container Binding Guide — singleton vs scoped vs bind

Guia definitivo para escolha do tipo de binding no AppServiceProvider.
A inconsistência (usar `singleton` num momento e `scoped` noutro para o mesmo service)
ocorre porque a decisão depende de características concretas do service, não de preferência.

---

## Sumário

- Árvore de decisão
- Definições com exemplos
- Quando NÃO registrar
- Padrões comuns do projeto
- Anti-patterns

---

## Árvore de Decisão

Seguir esta árvore na ordem. Parar na primeira condição verdadeira:

```
1. O service é uma classe concreta sem interface e sem config dinâmica?
   └─ SIM → NÃO registrar. Laravel auto-resolve.
   └─ NÃO → continuar

2. O service implementa uma interface?
   └─ SIM → Registrar (interface → implementação). Continuar para tipo.
   └─ NÃO → continuar

3. O service precisa de config dinâmica (API keys, valores de config())?
   └─ SIM → Registrar. Continuar para tipo.
   └─ NÃO → NÃO registrar. Laravel auto-resolve.

--- Escolha do tipo de binding ---

4. O service mantém estado mutável entre chamadas de método?
   └─ SIM → `scoped` (estado isolado por request)
   └─ NÃO → continuar

5. A criação do service é custosa (HTTP client, conexão externa, parsing pesado)?
   └─ SIM → `singleton` (criar uma vez, reutilizar)
   └─ NÃO → continuar

6. O service é usado em múltiplos pontos da mesma request?
   └─ SIM → `singleton` (evitar recriação)
   └─ NÃO → `bind` (criar sob demanda)
```

---

## Definições

### `singleton` — Uma instância para toda a aplicação

A mesma instância é compartilhada entre todas as requests e todos os pontos de
injeção. Vive até o processo encerrar (ou worker reiniciar em queue/Octane).

**Usar quando:**
- Service é stateless (sem estado mutável)
- Criação é custosa (conexão HTTP, parsing de config, warm-up)
- Reutilizado em vários pontos da aplicação
- Conecta-se a API externa com client reutilizável

**Exemplos reais:**

```php
// ✅ Singleton — client HTTP reutilizável, stateless, criação custosa
$this->app->singleton(
    PaymentGateway::class,
    fn () => new StripePaymentGateway(
        apiKey: config('services.stripe.key'),
        webhookSecret: config('services.stripe.webhook_secret'),
    ),
);

// ✅ Singleton — serviço de encriptação, stateless, config imutável
$this->app->singleton(
    EncryptionService::class,
    fn () => new EncryptionService(
        algorithm: config('encryption.algorithm'),
        key: config('encryption.key'),
    ),
);

// ✅ Singleton — cache/token service, stateless, usado em muitos pontos
$this->app->singleton(TokenGeneratorService::class);
```

**Cuidados:**
- Nunca armazenar dados de request no singleton (Auth::user(), request())
- Em Laravel Octane / queue workers, singletons persistem entre requests —
  qualquer estado mutável causará vazamento de dados entre requests

---

### `scoped` — Uma instância por request (lifecycle de request)

Nova instância criada a cada request HTTP ou job de queue. Dentro da mesma
request, todas as injeções recebem a mesma instância.

**Usar quando:**
- Service acumula estado durante a request (contadores, buffers, cache local)
- Service precisa de isolamento entre requests (dados de request-specific)
- Service faz tracking de operações dentro da request

**Exemplos reais:**

```php
// ✅ Scoped — acumula log de operações durante a request
$this->app->scoped(AuditTrailService::class);

// ✅ Scoped — cache de queries por request, estado mutável
$this->app->scoped(
    RequestCacheService::class,
    fn () => new RequestCacheService(
        ttl: (int) config('cache.request_ttl', 60),
    ),
);

// ✅ Scoped — estado mutável acumulado (mensagens coletadas na request)
$this->app->scoped(FlashMessageCollector::class);
```

**Quando NÃO usar scoped:**
- Service stateless sem estado mutável → usar `singleton`
- Service com interface que troca implementação → `singleton` se stateless

---

### `bind` — Nova instância a cada resolução

Cada vez que o container resolve a classe, cria uma nova instância.

**Usar quando:**
- Service é barato de criar
- Cada consumidor precisa de sua própria instância isolada
- Service é usado apenas uma vez por request
- Service carrega dados específicos da chamada (factory-like)

**Exemplos reais:**

```php
// ✅ Bind — cada uso precisa de instância nova com dados diferentes
$this->app->bind(
    ReportGenerator::class,
    fn () => new ReportGenerator(
        tempDir: storage_path('app/reports'),
        format: config('reports.default_format', 'pdf'),
    ),
);

// ✅ Bind — interface com implementação que varia por contexto
$this->app->bind(ExportDriver::class, CsvExportDriver::class);
```

---

### Sem registro — Auto-resolve do Laravel

O container resolve automaticamente qualquer classe concreta cujas dependências
também são resolvíveis. Não registrar quando não há necessidade.

**Usar quando:**
- Classe concreta (sem interface)
- Todas as dependências são tipadas e resolvíveis
- Não precisa de config dinâmica no construtor

**Exemplo:**

```php
// Service sem config dinâmica, dependências auto-resolvíveis
final readonly class OrderService
{
    public function __construct(
        private PaymentGateway $paymentGateway,  // ← já registrado
        private OrderRepository $orderRepository, // ← classe concreta
    ) {}
}

// NÃO precisa registrar. Laravel resolve automaticamente:
// - PaymentGateway: já tem binding registrado
// - OrderRepository: classe concreta, auto-resolve
```

---

## Tabela de Decisão Rápida

| Característica do Service                | Binding      |
|------------------------------------------|--------------|
| Stateless + config dinâmica              | `singleton`  |
| Stateless + interface                    | `singleton`  |
| Stateless + criação custosa              | `singleton`  |
| Stateless + usado em muitos pontos       | `singleton`  |
| Estado mutável por request               | `scoped`     |
| Acumula dados durante request            | `scoped`     |
| Tracking/audit por request               | `scoped`     |
| Nova instância necessária a cada uso     | `bind`       |
| Uso pontual + criação barata             | `bind`       |
| Classe concreta sem config dinâmica      | Não registrar|

---

## Padrões Comuns do Projeto

Baseado na stack do projeto (Laravel 13, Redis, Reverb, APIs externas):

| Service                  | Binding      | Justificativa                              |
|--------------------------|--------------|---------------------------------------------|
| `EncryptionService`      | `singleton`  | Stateless, config imutável                  |
| `TokenGeneratorService`  | `singleton`  | Stateless, sem config dinâmica              |
| `NavigationService`      | `scoped`     | Estado de navegação por request             |
| `ActivityLogService`     | `singleton`  | Stateless, registra sem acumular            |
| `ImageCropService`       | `bind`       | Cada uso é independente, criação barata     |
| `RedirectService`        | `scoped`     | Acumula redirecionamento por request        |

---

## Anti-patterns

### 1. Registrar sem necessidade

```php
// ❌ Desnecessário — Laravel resolve automaticamente
$this->app->singleton(OrderService::class);

// A menos que OrderService tenha construtor com config dinâmica,
// este registro é redundante e adiciona complexidade.
```

### 2. Singleton com estado mutável

```php
// ❌ Estado mutável em singleton causa vazamento entre requests
$this->app->singleton(CartService::class);

class CartService {
    private array $items = []; // ← compartilhado entre TODAS as requests!
}

// ✅ Corrigido — scoped isola por request
$this->app->scoped(CartService::class);
```

### 3. Scoped para service stateless

```php
// ❌ Scoped desnecessário — service não tem estado mutável
$this->app->scoped(
    EncryptionService::class,
    fn () => new EncryptionService(config('encryption.key')),
);

// ✅ Singleton é mais eficiente para stateless
$this->app->singleton(
    EncryptionService::class,
    fn () => new EncryptionService(config('encryption.key')),
);
```

### 4. Facade no construtor do service

```php
// ❌ Dificulta testes — dependência oculta
final readonly class ReportService
{
    public function generate(): void
    {
        $user = Auth::user();         // ← facade
        Cache::put('key', 'value');   // ← facade
    }
}

// ✅ Injetar contratos — testável e explícito
final readonly class ReportService
{
    public function __construct(
        private Guard $auth,
        private Repository $cache,
    ) {}

    public function generate(): void
    {
        $user = $this->auth->user();
        $this->cache->put('key', 'value');
    }
}
```
