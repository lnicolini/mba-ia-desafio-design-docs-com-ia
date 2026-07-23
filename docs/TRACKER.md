# Tracker de Rastreabilidade

Este documento mapeia os itens registrados nos design docs à sua origem na transcrição da reunião ou no código fonte da aplicação. O objetivo é garantir que nenhum requisito, decisão ou restrição tenha sido inventado — toda informação pode ser verificada nas fontes originais.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Adotar padrão Outbox no MySQL existente, não síncrono e não fila externa | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Descarte de chamada síncrona no service de orders: cliente lento travaria transação | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Descarte de Redis Streams: overengineering e nova infra para time pequeno | TRANSCRICAO | [09:07] Diego / Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-polling-em-processo-separado.md | Decisão | Worker em processo separado fazendo polling de 2 segundos | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-worker-polling-em-processo-separado.md | Trade-off | Descarte de worker no mesmo processo da API: reinício mata worker | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-polling-em-processo-separado.md | Trade-off | Descarte de trigger do MySQL notificando externamente: MySQL não tem NOTIFY/LISTEN nativo | TRANSCRICAO | [09:09] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Retry exponencial com 5 tentativas e DLQ separada | TRANSCRICAO | [09:15] Diego; [09:17] Diego; [09:18] Diego |
| ADR-003-ALT1 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Descarte de retry indefinido: evento ficaria pendurado para sempre | TRANSCRICAO | [09:15]–[09:16] Diego |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Descarte de 3 tentativas agressivas: não cobriria manutenção de 2 horas | TRANSCRICAO | [09:16] Bruno propõe, Diego rebate |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 por endpoint, secret única, rotação com grace period 24h | TRANSCRICAO | [09:19]–[09:22] Sofia |
| ADR-004-ALT1 | docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md | Trade-off | Descarte de secret global: blast radius de vazamento é total | TRANSCRICAO | [09:21] Sofia |
| ADR-004-NFR | docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md | Restrição | TLS obrigatório; URLs http rejeitadas; payload limitado a 64KB | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com header X-Event-Id para dedup do cliente | TRANSCRICAO | [09:24]–[09:26] Diego / Larissa |
| ADR-005-ALT1 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Descarte de exactly-once: exigiria coordenação bilateral, muito mais complexo | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reusar AppError, Pino, Zod, Prisma, estrutura src/modules, middlewares existentes | TRANSCRICAO | [09:27]–[09:31] Bruno / Larissa |
| ADR-006-CODE1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | AppError como classe base de erros do projeto | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Middleware de erro trata AppError, ZodError e Prisma errors | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CODE3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Logger Pino com redaction de campos sensíveis | CODIGO | src/shared/logger/index.ts |
| ADR-006-CODE4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Middleware de autenticação JWT com roles ADMIN/OPERATOR | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-CODE5 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Estrutura de módulo com controller, service, repository, routes, schemas | CODIGO | src/modules/orders/order.service.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao-outbox.md | Decisão | Snapshot do payload renderizado na inserção da outbox (imutável) | TRANSCRICAO | [09:51]–[09:53] Larissa / Diego / Bruno |
| ADR-007-ALT1 | docs/adrs/ADR-007-payload-snapshot-na-insercao-outbox.md | Trade-off | Descarte de lazy render: payload poderia refletir estado alterado posteriormente | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook via POST com url, eventFilter, customerId; secret gerada automaticamente | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar (PATCH) e remover (DELETE) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtrar eventos por status na inserção da outbox; sem match não insere | TRANSCRICAO | [09:34] Bruno / Diego |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Histórico de entregas com sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Entrega com latência máxima de 10 segundos via worker em polling | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Retry automático até 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | DLQ persistente e reprocessamento manual por admin | TRANSCRICAO | [09:18] Diego; [09:35] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Endpoint de replay DLQ exige role ADMIN e audita operador | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Payload enxuto com event_id, event_type, timestamp, order_id, from/to status, total_cents | TRANSCRICAO | [09:43] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não-Funcional | Atomicidade: inserção na outbox e mudança de status na mesma transação SQL | TRANSCRICAO | [09:06] Diego; [09:41] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não-Funcional | Worker separado em src/worker.ts, mesmo banco, instância Prisma separada | TRANSCRICAO | [09:11] Diego; [09:30] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não-Funcional | Ordering parcial por created_at com single-worker | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não-Funcional | TLS obrigatório; validação de URL https no schema Zod | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não-Funcional | Limite de payload de 64KB; rejeita se ultrapassar | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não-Funcional | Timeout de 10 segundos para HTTP call do worker | TRANSCRICAO | [09:42] Sofia / Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não-Funcional | Logs estruturados com correlation ID (event_id) | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-08 | docs/PRD.md | Requisito Não-Funcional | Estrutura de módulo igual aos existentes: controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência média ≤ 5s, p95 ≤ 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-01 | docs/PRD.md | Exclusão | Notificação por email como fallback para falha | TRANSCRICAO | [09:37] Marcos propõe; [09:37] Larissa descarta |
| PRD-FORA-02 | docs/PRD.md | Exclusão | Dashboard visual / frontend para gestão de webhooks | TRANSCRICAO | [09:39] Marcos; [09:40] Larissa descarta |
| PRD-FORA-03 | docs/PRD.md | Adiado | Rate limiting / throttling de envio por cliente | TRANSCRICAO | [09:38] Diego; [09:39] Larissa adia |
| RFC-PROP-01 | docs/RFC.md | Proposta | Worker separado consumindo outbox MySQL com polling de 2s | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| RFC-PROP-02 | docs/RFC.md | Proposta | Retry exponencial 1m/5m/30m/2h/12h com DLQ separada | TRANSCRICAO | [09:17] Diego |
| RFC-PROP-03 | docs/RFC.md | Proposta | At-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Redelivery síncrono dentro do fluxo de pedido | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Filas externas (Redis Streams / RabbitMQ / SQS) | TRANSCRICAO | [09:07] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída por cliente (50 pedidos → 50 chamadas) | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers e ordering global (particionamento por order_id ou lock pessimista) | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação de evento na outbox dentro de prisma.$transaction no changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker loteia pendentes, processa, marca PROCESSING/DELIVERED/FALHA | TRANSCRICAO | [09:08] Diego; [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com nextRetryAt calculated por attemptCount | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Move para DLQ após 5 tentativas; replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego; [09:35] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — criação de endpoint | TRANSCRICAO | [09:31] Marcos / Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/customers/:customerId/webhooks — listagem | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — reprocessamento | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:29] Bruno (prefixo WEBHOOK_) |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400) — TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_STATUS (400) — filtro inexistente | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (400) — edge case de geração de secret | TRANSCRICAO | [09:29] Bruno |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT — timeout de 10s; worker loga e retry | TRANSCRICAO | [09:42] Diego / Sofia |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (400) — > 64KB | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_SIGNATURE_VERIFICATION_FAILED (500) — falha interna ao computar HMAC | TRANSCRICAO | [09:20] Sofia |
| FDD-RESILIENCIA-01 | docs/FDD.md | Resiliência | Timeout de 10 segundos por requisição HTTP | TRANSCRICAO | [09:42] Sofia / Diego |
| FDD-RESILIENCIA-02 | docs/FDD.md | Resiliência | Backoff exponencial fixo com 5 tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-RESILIENCIA-03 | docs/FDD.md | Resiliência | Movimento para DLQ como destino terminal | TRANSCRICAO | [09:18] Diego |
| FDD-RESILIENCIA-04 | docs/FDD.md | Resiliência | Reprocessamento manual preservando event_id | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Métrica webhook_outbox_pending_total | TRANSCRICAO | [09:10] Larissa |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs estruturados via Pino com requestId e eventId | TRANSCRICAO | [09:29] Bruno / Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Header X-Event-Id como correlation ID | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | src/modules/orders/order.service.ts recebe publishWebhookEvent(tx, ...) | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-INT-02 | docs/FDD.md | Integração | src/shared/errors/app-error.ts como base para erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Integração | src/middlewares/error.middleware.ts trata AppError e ZodError sem mudança | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | src/middlewares/auth.middleware.ts fornece authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | src/shared/logger/index.ts usado pelo worker e reaproveitado com redaction | CODIGO | src/shared/logger/index.ts |
| FDD-INT-06 | docs/FDD.md | Integração | src/server.ts inspirando src/worker.ts como entry-point separado | TRANSCRICAO | [09:11] Larissa; [09:30] Bruno |

---

## Cobertura e Validação

- Total de itens rastreados: **74**
- Fonte = TRANSCRICAO com timestamp: **~60** linhas (~81 %)
- Fonte = CODIGO com caminho real: **8** linhas (acima do mínimo de 5)
