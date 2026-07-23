# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo         | Valor                                                              |
|---------------|--------------------------------------------------------------------|
| **Autor**     | Larissa (Tech Lead) — documento técnico elaborado por Diego e Bruno  |
| **Data**      | Quinta-feira, 09:00 (data da reunião de consenso)                  |
| **Status**    | Proposto / Em revisão                                              |
| **Revisores** | Marcos (PM), Sofia (Segurança), Diego (Plataforma), Bruno (Pedidos)  |

---

## 1. TL;DR (Resumo Executivo)

Substituir o modelo de polling manual dos clientes B2B por webhooks push com garantia de entrega at-least-once, latência máxima de 10 segundos e assinatura HMAC-SHA256. A entrega é feita por um **worker em processo separado** consumindo uma **tabela outbox no MySQL**. Retry com backoff exponencial até 5 tentativas (~15 h de janela), depois DLQ com reprocessamento manual por admins. Todo o módulo reusa padrões, estruturas e bibliotecas existentes no projeto (AppError, Pino, Zod, Prisma, Express).

---

## 2. Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) exigem notificação em tempo real quando o status de um pedido muda. Hoje eles consomem `GET /orders` de forma progressiva, gerando carga desnecessária e fricção de integração. A Atlas ameaçou churn caso a feature não esteja disponível até o fim de novembro (prazo: 3 sprints). A aplicação atual (Node.js + TypeScript + MySQL via Prisma) não possui qualquer mecanismo de notificação externa, filas ou eventos.

---

## 3. Proposta Técnica

### 3.1. Visão Geral da Solução

Introduzimos um novo domínio `webhooks` na aplicação, estruturado como um módulo igual aos existentes (`src/modules/webhooks` com controller, service, repository, routes, schemas). Além do módulo HTTP, criamos um entry-point separado `src/worker.ts` para o processo de entrega.

A alteração sensível é no método `changeStatus` de `OrderService`. Dentro da mesma transação Prisma que atualiza a ordem, decrementa estoque e grava histórico, inserimos um evento na tabela `webhook_outbox` (snapshot do payload). Isso garante atomicidade entre mudança de pedido e registro do evento.

Um worker stateless faz **polling** na outbox a cada 2 segundos, lê o lote de eventos pendentes, dispara requisições HTTP assinadas e marca como entregue ou falho. Regras de retry, DLQ, autenticação e observabilidade são governadas pelas decisões arquiteturais já fechadas (ver ADRs).

### 3.2. Macro Componentes

```
┌─────────────────┐
│   OrderService  │──┐
│  changeStatus() │  │  transação Prisma
└─────────────────┘  │  (order + history + stock + outbox)
                     ▼
         ┌──────────────────────┐
         │   webhook_outbox   │  (MySQL, índice status + created_at)
         └──────────────────────┘
                     │
                     │ polling 2s
                     ▼
         ┌──────────────────────┐
         │   src/worker.ts      │  (processo separado, PrismaClient próprio)
         │   webhook.processor  │
         └──────────────────────┘
                     │
                     ▼ HTTPS + HMAC-SHA256
         ┌──────────────────────┐
         │   Endpoint do Cliente │
         └──────────────────────┘
```

### 3.3. Decisões Arquiteturais Relacionadas

| Decisão | ADR |
|---|---|
| Outbox no MySQL com snapshot do payload | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s | [ADR-002](adrs/ADR-002-worker-polling-em-processo-separado.md) |
| Retry exponencial 1m/5m/30m/2h/12h + DLQ | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação 24h | [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com X-Event-Id | [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) |
| Reuso dos padrões existentes | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Snapshot do payload na inserção | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao-outbox.md) |

---

## 4. Alternativas Consideradas

### 4.1. Redilvery Síncrono no Fluxo de Pedido

Trigger direto de HTTP dentro de `changeStatus` antes de dar commit na transação.

- **Trade-off:** cliente lento ou offline bloquearia a transação de modificação de status de todos os pedidos. Falha no webhook exigiria rollback da mudança de status — comportamento inaceitável para um sistema primário de pedidos.
- **Descarte:** [09:04] Bruno e [09:06] Diego.

### 4.2. Filas Externas (Redis Streams / RabbitMQ / SQS)

Serviço de mensageria dedicado para publicar eventos e consumir por workers.

- **Trade-off:** latência menor, throughput e desacoplamento superiores. Por outro lado, demanda provisionar, monitorar, fazer backup e pagar por infra extra. O volume atual de eventos e o tamanho do time não justificam a sobrecarga operacional.
- **Descarte:** [09:07] Diego argumentou que “subir Redis Cluster pra isso é overengineering”.

---

## 5. Questões em Aberto

| # | Questão | Quem levantou | Por que permanece aberto |
|---|---------|---------------|--------------------------|
| 1 | **Rate limiting de saída por cliente:** se 50 pedidos mudarem de status em 1 min, o cliente recebe 50 chamadas. Devemos implementar throttling/batch? | Diego ([09:38]) | O time decidiu observar em produção antes de adicionar complexidade. Não está no escopo do MVP. |
| 2 | **Escala para múltiplos workers e ordering global:** hoje um único worker garante ordenação por `created_at`. Se o volume exigir múltiplos workers, a ordenação de eventos por pedido pode se perder. | Diego ([09:12], [09:13]) | O time decidiu que single-worker é suficiente para o MVP; particionamento por `order_id` ou lock pessimista fica como problema do futuro. Propositalmente adiado. |

---

## 6. Impacto e Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| Worker falha silenciosamente e eventos param de sair | Média | Alto | Healthcheck do processo worker; monitorar métrica `webhook_outbox_pending_total`; alerta se crescer monotonicamente. |
| Cliente fica 15h+ fora e eventos vão para DLQ, sem que ninguém reprocesse | Baixa | Médio | Dashboard de DLQ e endpoint de replay (`POST /admin/webhooks/dead-letter/:id/replay`) com log de auditoria. |
| Vazamento de secret de webhook | Baixa | Alto | Secret por endpoint (blast radius limitado); rotação com grace period; redação de `*.secret` nos logs. |
| Poluição da outbox por eventos filtrados desnecessariamente | Baixa | Médio | Filtrar eventos na inserção: só cria outbox se houver ao menos um webhook ativo e interessado naquele `toStatus`. |

---

## 7. Planejamento e Prazo

- **Estimativa:** 3 sprints.
- **Alocação:** Sprint 1 (modelagem outbox + DLQ + Prisma schema); Sprint 2 (worker + retry + backoff + HMAC); Sprint 3 (CRUD de config + deliveries + integração `OrderService` + testes ponta a ponta).
- **Gate de segurança:** 2 dias úteis de revisão de código por Sofia antes do deploy.

---

## 8. Checklist de Aprovação

- [ ] Marcos valida que requisitos funcionais cobrem expectativas dos clientes B2B.
- [ ] Sofia revisa implementação de HMAC, rotação e validação TLS.
- [ ] Diego aprova abordagem de polling e estratégia de concorrência no worker.
- [ ] Bruno confirma que a transação em `changeStatus` não sofre regressão de performance com o insert adicional na outbox.

---

*Este documento está aberto para comentários até aprovação. Após aprovação, as decisões são formalizadas nos ADRs vinculados.*
