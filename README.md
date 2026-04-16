# laravel-service-class (v1.0.2)

Skill para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que padroniza a criação, revisão e refatoração de classes de serviço (Services) em Laravel, seguindo padrões enterprise de organização, tipagem e testabilidade.

## O que faz

Recebe requisições para criar ou revisar Service classes e entrega código padronizado aplicando:

- **Service Class:** `final readonly class` com `strict_types`, injeção via construtor com `readonly`, tipagem completa, métodos com verbos de ação em inglês
- **Classes Auxiliares:** DTOs, Enums (`BackedEnum`), Exceptions de domínio, ValueObjects imutáveis, Repositories e Support — criados sob demanda no diretório do serviço
- **Container Binding:** decisão guiada entre `singleton`, `scoped` e `bind` com registro no `AppServiceProvider`
- **Testes:** geração com Pest seguindo padrões de teste para services
- **Documentação:** gera `.md` da service class otimizado para agentes AI

> **Stack:** PHP 8.4 · Laravel 13.x · Pest

---

## Instalação

Requisito: [Node.js](https://nodejs.org/) instalado (para `npx`).

```bash
npx degit elimarcosmorais/skill-laravel-service-class .claude/skills/laravel-service-class
```

Isso baixa a skill diretamente para `.claude/skills/laravel-service-class` sem histórico git.

### Verificar a instalação

```bash
ls .claude/skills/laravel-service-class/
# SKILL.md  references/
```

---

## Atualização

Para atualizar para a versão mais recente, rode o mesmo comando com a flag `--force`:

```bash
npx degit elimarcosmorais/skill-laravel-service-class .claude/skills/laravel-service-class --force
```

A flag `--force` sobrescreve os arquivos existentes.

### Instalar uma versão específica

Se o repositório usar tags de versão:

```bash
npx degit elimarcosmorais/skill-laravel-service-class#v1.0.0 .claude/skills/laravel-service-class --force
```

---

## Estrutura

```
laravel-service-class/
├── SKILL.md                                  # Instruções para o Claude Code
└── references/
    ├── service-class-template.md             # Template completo com exemplo real
    ├── container-binding-guide.md            # Decisão singleton vs scoped vs bind
    └── pest-testing-guide.md                 # Padrões de teste com Pest para services
```

---

## Como usar

Com a skill instalada, basta pedir ao Claude Code para criar ou revisar um service:

```
Crie um service de pagamento seguindo a skill laravel-service-class
```

A skill será acionada automaticamente quando você mencionar termos como _criar service_, _refatorar service_, _testar service_, _registrar service_, _service provider_, _singleton vs scoped_, _extrair lógica para service_, entre outros.

---

## Desinstalação

```bash
rm -rf .claude/skills/laravel-service-class
```

---

## Licença

MIT
