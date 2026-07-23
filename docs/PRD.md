# Product Requirement Document (PRD): Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature

Hoje, clientes B2B que integram com a nossa plataforma precisam fazer polling repetitivo na API (`GET /orders`) para descobrir mudanças de status em seus pedidos. Essa abordagem é lenta, cara em infraestrutura do cliente e gera fricção operacional. Três clientes estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — exigiram notificações push. A Atlas vinculou a continuidade do contrato à entrega da feature até o final de novembro.

A feature consiste em um Sistema de Webhooks de Notificação de Pedidos: quando o status de um pedido muda na nossa plataforma, o sistema entrega automaticamente um evento HTTP para um endpoint configurado pelo cliente, com baixa latência, garantia de entrega e autenticidade verificável.

## 2. Problema e Motivação

- **Para o cliente:** polling gera latência percebida alta, desperdício de requests e dificuldade de integração em tempo real.
- **Para o negócio:** risco de churn de clientes estratégicos B2B; barreira de adoção para novos parceiros que esperam webhooks como commodity.
- **Para a operação:** requests desnecessários de polling aumentam carga na API de pedidos, sem agregar valor.

## 3. Público-Alvo e Cenários de Uso

**Público-alvo primário:** Clientes B2B (integradores ERP, WMS, plataformas logísticas) que gerenciam dezenas a milhares de pedidos diariamente.

**Cenários de uso:**
1. **Atlas Comercial:** deseja iniciar coleta/logística assim que status muda para `SHIPPED`, sem precisar consultar a API.
2. **MaxDistribuição:** atualiza seu ERP interno quando o pedido é marcado como `PAID` ou `CANCELLED`, automatizando fluxo de faturamento.
3. **Nova Cargo:** aguarda sinal de `DELIVERED` para fechar ciclo de entrega e liberar pagamento ao transportador.

## 4. Objetivos e Métricas de Sucesso

| # | Objetivo | Métrica | Meta | Como medir |
|---|----------|---------|------|------------|
| 1 | Reduzir latência de notificação | Tempo médio entre status change e webhook entregue | ≤ 5 s (p95 ≤ 10 s) | Métrica `webhook_delivery_duration_seconds` no worker |
| 2 | Diminuir carga de polling | Redução no volume de `GET /orders` vindos de clientes B2B | ≥ 60 % em 30 dias | Logs de acesso ou rate-limiter por API key (futuro) |
| 3 | Prevenir retenção de clientes | Número de clientes B2B reportando satisfação com integração | 100 % dos 3 clientes alvo validam integração até 15/dez | Pesquisa / acceptance test comerciais |

## 5. Escopo

### Dentro do Escopo

Inclusão de mecanismo transacional de outbox em MySQL para eventos de mudança de status de pedidos.
Worker separado para processamento e entrega HTTP dos eventos.
CRUD completo de configuração de webhooks por cliente (cadastro, edição, remoção, listagem, filtro de eventos).
Histórico de entregas por endpoint (`GET /webhooks/:id/deliveries`).
Retry automático com backoff exponencial e Dead Letter Queue (DLQ).
Reprocessamento manual de DLQ via endpoint administrativo.
Autenticação HMAC-SHA256 por endpoint, com rotação de secret.
Garantia at-least-once com header `X-Event-Id` para deduplicação pelo cliente.
Reaproveitamento integral dos padrões, bibliotecas e estrutura de módulos da aplicação existente.

### Fora do Escopo

- **Notificação por email como fallback para falha:** foi proposto pelo Marcos ([09:37]) e explicitamente descartado pela Larissa. Fica para uma fase futura, depois que houver métricas de impacto.
- **Dashboard visual / frontend para gestão de webhooks:** mencionado pelo Marcos ([09:39]) e descartado pela Larissa. O cliente interage via API; painel visual é projeto do time de frontend, separado.
- **Arquivamento automático de eventos entregues após 30 dias:** mencionado por Diego ([09:08]) como necessidade operacional futura, mas fora do escopo da feature atual.

### Adiado / Observar

- **Rate limiting / throttling de envio por cliente:** levantado por Diego ([09:38]). O time decidiu observar comportamento em produção antes de adicionar complexidade de throttling ou batching. Não entra no MVP, mas pode ser reavaliado com base em métricas de volume.

## 6. Requisitos Funcionais

| ID | Requisito | Fonte na conversa |
|----|-----------|-------------------|
| FR-01 | O cliente deve poder cadastrar um webhook via API, informando URL, lista de status de interesse e customerId. A secret HMAC é gerada automaticamente pelo sistema. | Marcos [09:31] |
| FR-02 | O sistema deve permitir editar (PATCH) e remover (DELETE) um webhook previamente cadastrado. | Bruno [09:33] |
| FR-03 | O cliente deve poder listar todos os webhooks de um customer específico. | Bruno [09:33] |
| FR-04 | O sistema deve filtrar eventos por status na **inserção** da outbox: se nenhum webhook ativo do customer quer aquele status, nenhuma linha de outbox é criada. | Bruno [09:34], Diego [09:34] |
| FR-05 | O cliente deve poder consultar o histórico de entregas de um webhook (`GET /webhooks/:id/deliveries`), contendo status de sucesso/falha, payload, resposta e tempo. | Marcos [09:34] |
| FR-06 | O sistema deve entregar eventos com latência máxima de 10 segundos em condições normais, via worker em polling. | Marcos [09:02], Larissa [09:10] |
| FR-07 | Em caso de falha de entrega, o sistema deve retentar automaticamente até 5 vezes com backoff exponencial de 1min / 5min / 30min / 2h / 12h. | Diego [09:15], Larissa [09:17] |
| FR-08 | Após esgotar as 5 tentativas, o evento deve ser movido para uma Dead Letter Queue (DLQ) persistente e um administrador deve poder reprocessá-lo manualmente via endpoint. | Diego [09:18], Larissa [09:35] |
| FR-09 | O endpoint de reprocessamento de DLQ deve exigir role ADMIN e auditar quem fez o replay. | Sofia [09:36], Larissa [09:36] |
| FR-10 | O payload do webhook deve conter informações essenciais do pedido (event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents) sem incluir os itens do pedido. | Diego [09:43] |
| FR-11 | O cliente deve poder rotacionar a secret de um webhook; a secret anterior deve permanecer válida por 24 horas. | Sofia [09:21] |

## 7. Requisitos Não Funcionais

| ID | Requisito | Fonte |
|----|-----------|-------|
| NFR-01 | **Atomicidade transacional:** a inserção do evento na outbox e a mudança de status do pedido devem ocorrer dentro da mesma transação SQL (commit/rollback unificado). | Diego [09:06] |
| NFR-02 | **Worker separado:** o processo de entrega deve rodar em entry-point diferente da API HTTP (`src/worker.ts`), conectando-se ao mesmo banco. | Diego [09:11] |
| NFR-03 | **Ordering parcial:** eventos de um mesmo pedido devem ser entregues na ordem cronológica de criação, garantida pelo single-worker e índice `created_at` na outbox. | Diego [09:12] |
| NFR-04 | **TLS obrigatório:** URLs de webhook devem ser `https://`; URLs `http://` devem ser rejeitadas na criação/edição. | Sofia [09:23] |
| NFR-05 | **Limite de payload:** eventos com payload excedendo 64 KB devem ser rejeitados. | Sofia [09:23] |
| NFR-06 | **Timeout:** requisição HTTP para o cliente deve usar timeout de 10 segundos. | Sofia [09:42] |
| NFR-07 | **Observabilidade:** todos os envios devem gerar logs estruturados (Pino) com correlation ID (`event_id`). | Reunião consenso geral. |
| NFR-08 | **Compatibilidade:** o módulo de webhooks deve seguir a estrutura de módulos existente (`src/modules/<domain>/`, com `controller`, `service`, `repository`, `routes`, `schemas`). | Bruno [09:27] |

## 8. Decisões e Trade-offs Principais

- **Outbox no MySQL vs. fila externa:** optou-se por outbox no banco já existente para evitar nova infraestrutura e manter atomicidade. Custo é latência de polling e carga de I/O compartilhada.
- **At-least-once vs. exactly-once:** exactly-once exigiria coordenação bilateral com o cliente (código que não controlamos). O padrão at-least-once do mercado (Stripe, GitHub) foi adotado, transferindo deduplicação para o consumidor.
- **Snapshot do payload vs. lazy render:** o payload é serializado na inserção da outbox para imutabilidade histórica, mesmo que ocupe mais espaço.
- **Retry exponencial vs. retry agressivo:** 5 tentativas em ~15 h cobrem janelas de manutenção do cliente, ao contrário de 3 tentativas que falhariam em poucos minutos.

## 9. Dependências

- Entrega da migração Prisma com tabelas de outbox, DLQ, configuração e deliveries.
- Implementação do entry-point `src/worker.ts` e script `npm run worker`.
- Revisão de segurança da Sofia antes do deploy (2 dias úteis).

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| Worker para de funcionar e eventos não saem (downtime silencioso) | Média | Alto | Healthcheck do processo; métrica de pending events; alerta se `webhook_outbox_pending_total` cresce monotonicamente por mais de 2 min. |
| Cliente B2B recebe duplicatas e processa ordem duas vezes | Baixa | Alto | Documentar claramente garantia at-least-once e header `X-Event-Id` no portal do desenvolvedor; fornecer exemplos de deduplicação. |
| Performance de `changeStatus` degrada por causa do insert adicional na transação | Média | Médio | Manter índice composto em `webhook_outbox(status, nextRetryAt)`; monitorar latência p95 da transação antes e depois; se necessário, inserir outbox como última operação da transação. |
| Vazamento de secret em log do cliente ou no nosso | Baixa | Alto | Redação de secrets nos nossos logs via Pino; validação de URL HTTPS obrigatória; documentar ao cliente para não logar headers de assinatura. |

## 11. Critérios de Aceitação

- [ ] Cliente consegue cadastrar webhook e receber evento < 10 s após mudança de status.
- [ ] Evento gerado por `changeStatus` é inserido na outbox atomically (mesma transação).
- [ ] Falha simulada no endpoint do cliente dispara retry 1min / 5min / 30min / 2h / 12h e move para DLQ na 5ª falha.
- [ ] Admin consegue reprocessar evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`.
- [ ] Requisição para URL `http://` é rejeitada com `WEBHOOK_INVALID_URL`.
- [ ] Signature HMAC-SHA256 pode ser validada pelo cliente usando a secret fornecida.
- [ ] Rotacionar secret gera nova secret válida imediatamente e mantém antiga por 24h.
- [ ] Em teste de carga com 100 eventos pendentes, 95 % são entregues em < 5 s (single worker).

## 12. Estratégia de Testes e Validação

- **Unitários:** service e repository de webhook (CRUD, filtro, snapshot, geração de secret).
- **Integração:** transação atômica de `OrderService.changeStatus` + outbox insert (mock de banco ou container MySQL via `docker-compose`).
- **Worker isolado:** simular fila de outbox e endpoint HTTP mock (nock / MSW), verificar retry, backoff, DLQ.
- **Segurança:** fuzzing de HMAC com payload variado; validação de grace period de secret.
- **Ponta a ponta (opcional):** subir API + worker + MySQL via Docker e validar todo o ciclo de cadastro de webhook → mudança de status → entrega → retry.

## 13. Notas e Referências

- Transcrição completa da reunião: `TRANSCRICAO.md`
- Decisões arquiteturais detalhadas: `docs/adrs/`
- Proposta técnica e revisão: `docs/RFC.md`
- Especificação de implementação: `docs/FDD.md`
