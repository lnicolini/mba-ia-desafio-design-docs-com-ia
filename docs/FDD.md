# Feature Design Document (FDD): Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e Motivação Técnica

A aplicação atual não possui mecanismo de notificação externa, filas ou eventos. Clientes B2B dependem de polling no `GET /orders`, gerando latência, custo de integração e risco de churn. O objetivo é introduzir um mecanismo de push confiável, atômico com a transação de pedido, sem alterar o comportamento dos módulos existentes.

## 2. Objetivos Técnicos

- Entrega de eventos de mudança de status com latência máxima de 10 segundos.
- Garantia at-least-once com deduplicação possível pelo consumidor.
- Atomicidade entre mudança de status e registro do evento (mesma transação SQL).
- Worker stateless, recuperável e observável.
- Zero dependências de infraestrutura nova (MySQL e stack Node.js existentes).

## 3. Escopo e Exclusões

### Incluso
- Tabela `webhook_outbox` e `webhook_dead_letter`.
- Worker em entry-point separado (`src/worker.ts`).
- CRUD de configuração de webhooks (`/webhooks`).
- Endpoint de entregas (`GET /webhooks/:id/deliveries`).
- Endpoint de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`).
- Integração no `OrderService.changeStatus` para inserção na outbox.
- HMAC-SHA256, retry exponencial, DLQ, headers de rastreabilidade.

### Excluso
- Rate limiting / throttling por cliente (ponto em aberto).
- Batching de eventos (ponto em aberto).
- Notificação por email de falha (adiado para próxima fase).
- Dashboard frontend (fora de escopo, projeto separado).
- Arquivamento automático de eventos entregues após 30 dias (fora de escopo).

## 4. Fluxos Detalhados

### 4.1. Criação do Evento na Outbox (OrderService)

1. Usuário autenticado chama `PATCH /api/v1/orders/:id/status`.
2. `OrderController.changeStatus` chama `OrderService.changeStatus(id, input, userId)`.
3. Dentro de `prisma.$transaction(async (tx) => { ... })`:
   a. Busca e valida transição de status.
   b. Ajusta estoque via `debitStock` / `replenishStock`.
   c. Atualiza `order.status` e insere `orderStatusHistory`.
   d. **Nova operação:** `publishWebhookEvent(tx, order, from, to)`:
      - Consulta `webhookConfig` do `customerId` com `active = true` e `eventFilter` incluindo `to`.
      - Se houver matches, serializa payload snapshot JSON.
      - Insere na `webhook_outbox` (status = `PENDING`, `event_id` UUID gerado, `created_at` = now, `payload` = JSON).
4. `tx` commita. Se falhar em qualquer ponto, outbox também rollback.

### 4.2. Processamento pelo Worker

Pseudo-código do loop principal em `src/worker.ts`:

```ts
while (running) {
  const events = await prisma.webhookOutbox.findMany({
    where: { status: 'PENDING', nextRetryAt: { lte: new Date() } },
    orderBy: { createdAt: 'asc' },
    take: 10,
  });

  for (const event of events) {
    await prisma.webhookOutbox.update({
      where: { id: event.id },
      data: { status: 'PROCESSING' },
    });

    try {
      const response = await fetch(event.url, {
        method: 'POST',
        headers: buildHeaders(event),
        body: event.payload,
        signal: AbortSignal.timeout(10000),
      });

      if (response.ok) {
        await markDelivered(event.id, response.status, response.headers);
      } else {
        await scheduleRetryOrDLQ(event, response.status);
      }
    } catch (networkError) {
      await scheduleRetryOrDLQ(event, null);
    }
  }

  await sleep(2000);
}
```

**Observação:** worker é single-thread e single-instance nesta fase. Ordem está garantida por `created_at` dentro de um único worker.

### 4.3. Retry e Backoff

| Tentativa | Delay desde a falha anterior |
|-----------|------------------------------|
| 1         | 1 minuto                     |
| 2         | 5 minutos                    |
| 3         | 30 minutos                   |
| 4         | 2 horas                      |
| 5         | 12 horas                     |

- `nextRetryAt` é calculado como `now + delay`.
- Se tentativa 5 falha: move para `webhook_dead_letter`.
- Colunas de `webhook_outbox` para retry: `attemptCount`, `lastAttemptAt`, `lastError`, `nextRetryAt`.

### 4.4. DLQ e Reprocessamento

- Tabela `webhook_dead_letter` armazena `payload`, `url`, `event_id`, `lastHttpStatus` (nullable), `lastError`, `failedAt`, `webhookId`.
- Replay: `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN, usa `requireRole('ADMIN')` e loga `userId`).
- Replay recria a linha na `webhook_outbox` com status `PENDING`, `attemptCount = 0`, novo `nextRetryAt = now`.

## 5. Contratos Públicos

### 5.1. Criar Webhook

```http
POST /api/v1/webhooks
Authorization: Bearer <token>
Content-Type: application/json
```

**Request:**
```json
{
  "url": "https://atlas.com.br/webhooks/orders",
  "eventFilter": ["SHIPPED", "DELIVERED"],
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Response 201:**
```json
{
  "id": "webhook-uuid-1",
  "url": "https://atlas.com.br/webhooks/orders",
  "secret": "whsec_xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "eventFilter": ["SHIPPED", "DELIVERED"],
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "active": true,
  "createdAt": "2024-01-15T09:30:00Z"
}
```

**Response 400:**
```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must use HTTPS"
  }
}
```

### 5.2. Listar Webhooks por Cliente

```http
GET /api/v1/customers/:customerId/webhooks
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "items": [
    {
      "id": "webhook-uuid-1",
      "url": "https://atlas.com.br/webhooks/orders",
      "eventFilter": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

### 5.3. Atualizar Webhook

```http
PATCH /api/v1/webhooks/:id
Authorization: Bearer <token>
Content-Type: application/json
```

**Request:**
```json
{
  "url": "https://atlas.com.br/webhooks/orders-v2",
  "eventFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response 200:** objeto atualizado.

**Response 404:**
```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

### 5.4. Rotacionar Secret de Webhook

```http
POST /api/v1/webhooks/:id/rotate-secret
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "id": "webhook-uuid-1",
  "url": "https://atlas.com.br/webhooks/orders",
  "secret": "whsec_new_secret_xxxxxxxxxxxxxxxxxxxxxxxx",
  "secretRotatedAt": "2024-01-15T10:00:00Z",
  "secretExpiresAt": "2024-01-16T10:00:00Z",
  "oldSecretValidUntil": "2024-01-16T10:00:00Z",
  "eventFilter": ["SHIPPED", "DELIVERED"],
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "active": true
}
```

**Response 404:**
```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

### 5.5. Deletar Webhook

```http
DELETE /api/v1/webhooks/:id
Authorization: Bearer <token>
```

**Response 204** (No Content).

### 5.6. Listar Deliveries

```http
GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "items": [
    {
      "id": "delivery-uuid-1",
      "eventId": "evt-uuid-1",
      "webhookId": "webhook-uuid-1",
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 245,
      "createdAt": "2024-01-15T10:00:00Z",
      "status": "DELIVERED"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

### 5.7. Replay de DLQ (Admin)

```http
POST /api/v1/admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <admin-token>
```

**Response 200:**
```json
{
  "replayId": "evt-uuid-1",
  "queuedAt": "2024-01-15T12:00:00Z"
}
```

**Response 403:**
```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Insufficient permissions"
  }
}
```

## 6. Matriz de Erros (Prefixo WEBHOOK_*)

| Código | HTTP | Mensagem (padrão) | Quando ocorre |
|--------|------|-------------------|---------------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook not found | GET/PATCH/DELETE/POST rotate-secret com ID inexistente. |
| `WEBHOOK_INVALID_URL` | 400 | Webhook URL must use HTTPS | Schema Zod rejeita URL não HTTPS. |
| `WEBHOOK_INVALID_STATUS` | 400 | Invalid status filter | `eventFilter` contém status inexistente. |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret is required for creation | Falha interna de geração (edge case). |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | — (log/worker) | Não exposto via API; worker loga e retry. |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 400 | Payload exceeds 64KB limit | Worker rejeita evento > 64KB antes do envio. |
| `WEBHOOK_SIGNATURE_VERIFICATION_FAILED` | 500 | Internal error (log) | Falha interna ao computar HMAC. |

## 7. Estratégias de Resiliência

- **Timeout:** 10 segundos por request HTTP (AbortController / `AbortSignal.timeout`).
- **Retry:** Backoff exponencial fixo com 5 tentativas; após a quinta, move para DLQ.
- **Circuit breaker:** *Não implementado nesta fase.* Considerar em evolução se um cliente ficar cronicamente offline.
- **Fallback:** Não há fallback para entrega síncrona; o DLQ é o destino terminal. Reprocessamento manual via endpoint admin.
- **Idempotência:** Replays de DLQ preservam o mesmo `event_id`, permitindo ao cliente deduplicar corretamente.

## 8. Observabilidade

### 8.1. Métricas (código hooks para futuro Prometheus ou similar)

- `webhook_outbox_pending_total` — gauge de eventos pendentes.
- `webhook_delivered_total` — counter com labels `status` (2xx, 4xx, 5xx, network_error).
- `webhook_delivery_duration_seconds` — histogram com buckets [0.1, 0.5, 1, 5, 10].
- `webhook_dead_letter_total` — counter de eventos movidos para DLQ.

### 8.2. Logs

- Worker: loga início de lote, resultado de cada entrega (success/failure) com `event_id`, `webhook_id`, `url`, `status_code`, `duration_ms`.
- Replay DLQ: loga `userId`, `dead_letter_id`, `event_id` em nível `info`.
- Todos via Pino (`src/shared/logger/index.ts`), com `service: 'webhook-worker'` ou `webhook-api`.

### 8.3. Tracing

- Header `X-Event-Id` propagado como correlation ID em todos os requests HTTP de saída.
- Logs do worker e da API devem incluir `eventId` no binding do log para correlação.

## 9. Dependências e Compatibilidade

- **Node.js >=20** (já definido no `package.json`).
- **Prisma 5.22.0** — nova migração adiciona tabelas; Prisma Client existente suporta.
- **MySQL 8.x** — já requerido pelo datasource; nenhuma feature nova do engine necessária.
- **Express 4.21.1** — rotas seguem mesmos middlewares de auth e validate.
- **Zod 3.23.8** — schemas de entrada e validação de URLs HTTPS.
- **Pino 9.5.0 / pino-http 10.3.0** — logging sem mudança de biblioteca.

## 10. Critérios de Aceite Técnicos

- [ ] Worker entrega evento em até 10 segundos após mudança de status em condições normais (single-worker, fila < 100 eventos pendentes).
- [ ] Falha no cliente após 5 tentativas move o evento para `webhook_dead_letter` e decremente `webhook_outbox_pending_total`.
- [ ] Mesma transação de `changeStatus` commita order + outbox (teste de integração com mock de banco verificando atomicidade).
- [ ] Endpoint de replay DLQ só aceita token com role `ADMIN`.
- [ ] Requisição de webhook para URL não HTTPS retorna `WEBHOOK_INVALID_URL` antes de persistir configuração.
- [ ] Payload de evento possui `event_id`, `event_type`, `timestamp`, `order_id`, `from_status`, `to_status`, `customer_id`, `total_cents` e é imutável após inserção na outbox.

## 11. Riscos e Mitigação (Técnica)

| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Morte do worker sem restart automático | Média | Alto | Orquestrador (systemd/Docker) com `restart=always`; healthcheck lendo pending events. |
| Crescimento da outbox por falha em massa de um cliente | Baixa | Alto | DLQ separada limita tamanho da outbox. Monitoração de DLQ size e alerta. |
| Deadlock no banco com Prisma transaction interna + worker lendo outbox concorrente | Baixa | Médio | Worker faz `UPDATE` de status para PROCESSING com `SKIP LOCKED` (ou equivalente Prisma) antes de processar. |
| Fuga de secrets em logs | Baixa | Alto | Secret é redigido no logger (`*.secret` no `redactPaths` do Pino). Signature (header) também não é logada como corpo. |

## 12. Integração com o Sistema Existente

Esta seção descreve como o novo módulo se integra a caminhos de arquivo reais já presentes na base de código.

### 12.1. `src/modules/orders/order.service.ts`

- **O que muda:** o método `changeStatus` (linhas 126–179 do arquivo atual) ganha uma chamada para `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro do bloco `prisma.$transaction`.
- **Como integra:** a função de publicação recebe o `TransactionClient` (`TxClient`) do Prisma, garantindo que o insert na `webhook_outbox` compartilha o mesmo commit/rollback da atualização da ordem, do histórico e do estoque.
- **Por que assim:** evita inconsistência entre status persistido e evento emitido.

### 12.2. `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`

- **O que muda:** novas classes de erro específicas de webhook (ex: `WebhookNotFoundError`, `WebhookInvalidUrlError`) estenderão `AppError` com `errorCode` prefixado por `WEBHOOK_`.
- **Como integra:** o `errorMiddleware` (ver abaixo) já trata instâncias de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError`. Nenhuma mudança no middleware é necessária para os novos erros serem serializados corretamente.
- **Por que assim:** consistência com `InsufficientStockError`, `InvalidStatusTransitionError` e outros erros de domínio já existentes.

### 12.3. `src/middlewares/error.middleware.ts`

- **O que muda:** nenhuma mudança no código do middleware.
- **Como integra:** o middleware captura instâncias de `AppError` (linhas 14–23) e ZodError (linhas 26–35). Os novos erros de webhook, por herdarem de `AppError`, terão seu `statusCode` e `errorCode` expostos automaticamente na resposta JSON.
- **Por que assim:** reutilização do tratamento centralizado, evitando handlers ad-hoc por módulo e manutenção de um único ponto de formatação.

### 12.4. `src/middlewares/auth.middleware.ts`

- **O que muda:** nenhuma mudança no código do middleware em si.
- **Como integra:** as rotas de CRUD de webhook usarão `authenticate` (linhas 27–47), e o endpoint de replay DLQ usará `authenticate` seguido de `requireRole('ADMIN')` (linhas 49–60). O tipo `AuthUser` (linhas 6–10) continua o contrato de `req.user`.
- **Por que assim:** reusa o mesmo esquema JWT, mesmo secret (`env.JWT_SECRET`) e mesma política de roles já implementada para usuários e pedidos. Não há necessidade de outro mecanismo de autenticação.

### 12.5. `src/shared/logger/index.ts`

- **O que muda:** adição de caminhos `*.secret`, `*.signature`, `*.secretOld` à lista `redactPaths` (linhas 4–11).
- **Como integra:** o logger Pino já é instanciado globalmente. Worker e rotas de webhook importam o mesmo `logger`, com `base: { service: 'webhook-worker' }` via child logger (`logger.child({ service: 'webhook-worker' })`).
- **Por que assim:** evita introduzir nova biblioteca de logging. Redação automática previne vazamento de secrets em logs estruturados.

### 12.6. `src/server.ts`

- **O que muda:** nenhuma mudança obrigatória no arquivo atual.
- **Como integra:** o worker vive em novo entry-point `src/worker.ts`, estruturado de forma similar a `server.ts` (bootstrap de Prisma, registro de sinais SIGINT/SIGTERM, disconnect limpo). O `package.json` recebe script `npm run worker`.
- **Por que assim:** separação de processo garante que reinício da API não aborte entregas em andamento.

## 13. Checklist de Deploy e Preparação

- [ ] Criar migração Prisma adicionando `WebhookConfig`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery`.
- [ ] Adicionar script `"worker": "node --env-file=.env dist/worker.js"` no `package.json`.
- [ ] Garantir que `DATABASE_URL` esteja acessível pelo processo worker (mesma variável).
- [ ] Configurar monitoramento (`webhook_outbox_pending_total` e healthcheck do worker).
- [ ] Executar revisão de segurança de Sofia (HMAC, rotação, redaction).
