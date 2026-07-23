# ADR-006: Reuso dos Padrões Existentes do Projeto

* Status: Aceito
* Decisores: Larissa, Bruno, Diego
* Data: [09:27]–[09:31] na transcrição

## Contexto

A codebase atual já possui estrutura, bibliotecas e convenções consolidadas. Em vez de introduzir novos frameworks ou padrões exclusivos para o módulo de webhooks, a reunião decidiu manter consistência arquitetural.

## Decisão

O módulo de webhooks deve **reutilizar integralmente** os padrões, bibliotecas e estruturas já existentes no projeto.

## Especificações de Reuso

| Padrão / Artefato | Origem no Código Base | Uso no Webhook |
|---|---|---|
| Tratamento de erro baseado em `AppError` | `src/shared/errors/app-error.ts` | Novos erros de webhook estendem `AppError` com prefixo `WEBHOOK_`. |
| Classes específicas de erro HTTP | `src/shared/errors/http-errors.ts` | Reutilizar `ValidationError`, `NotFoundError`, `ConflictError`, etc. quando aplicável. |
| Logger Pino com redaction | `src/shared/logger/index.ts` | Worker e rotas HTTP de webhook usam o mesmo logger, com redação de `*.secret` adicionada. |
| Middleware de erro centralizado | `src/middlewares/error.middleware.ts` | Trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` sem alteração. |
| Middleware de autenticação JWT | `src/middlewares/auth.middleware.ts` | Rotas de CRUD usam `authenticate`; endpoint de replay DLQ usa `authenticate` + `requireRole('ADMIN')`. |
| Estrutura de módulo | `src/modules/orders/` (exemplo) | Webhook vive em `src/modules/webhooks/` com `controller`, `service`, `repository`, `routes`, `schemas`. |
| Schemas de validação Zod | `src/modules/orders/order.schemas.ts` | Schemas de entrada do webhook seguem mesmo padrão de tipagem e coerção. |
| Prisma como ORM e transações | `src/modules/orders/order.service.ts` (uso de `$transaction`) | Inserção na outbox usa `tx` (transaction client) dentro do mesmo bloco de `changeStatus`. |

## Alternativas Consideradas

1. **Introduzir BullMQ / Agenda / outra biblioteca de jobs**
   - *Prós:* abstrações prontas para retry, delay e scheduling.
   - *Contras:* adiciona dependência nova, aumenta bundle, quebra o padrão de módulos existente. Não justifica para a fase atual.
   - *Decisão:* descartado implicitamente; consenso foi manter a stack mínima.

2. **Criar camada de eventos separada (EventBus interno)**
   - *Prós:* mais desacoplamento entre orders e webhooks.
   - *Contras:* camada extra que não existe hoje; `changeStatus` já está centralizado no `OrderService`. Inserção direta via `tx` é mais simples e atômica.
   - *Decisão:* descartado; Bruno propôs função pura de `publishWebhookEvent(tx, ...)` diretamente.

## Consequências

### Positivas
- Curva de aprendizado zero para desenvolvedores do time.
- Menor superfície de ataque em dependências.
- Middleware e handlers de erro já testados nos demais módulos; menos código para validar.

### Negativas
- O logger Pino precisa ser revisitado para garantir que `*.secret` e `*.signature` sejam redigidos em logs de webhook.
- A estrutura de módulo atual não prevê workers; exige criação de novo entry-point (`src/worker.ts`) e adição de script no `package.json`.
- Se no futuro o volume de eventos exigir uma fila dedicada, a migração exigirá retirar o polling e adaptar o padrão outbox para publicar em fila externa.

## Trade-off Explícito

Trocamos potencial de otimização futura por consistência e velocidade de entrega agora. O time conhece o padrão, o que reduz risco de bugs e retrabalho.
