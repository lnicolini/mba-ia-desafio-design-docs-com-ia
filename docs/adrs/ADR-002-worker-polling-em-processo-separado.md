# ADR-002: Worker em Processo Separado em Polling

* Status: Aceito
* Decisores: Diego, Larissa, Bruno
* Data: [09:05]–[09:10] na transcrição

## Contexto

Com o Outbox no MySQL definido, restava decidir como o worker consome os eventos pendentes. As opções incluíam: (a) processo interno à instância da API (mesmo processo Node), (b) trigger no banco de dados que notificasse um processo externo, e (c) processo separado fazendo polling.

## Decisão

O worker **roda como processo separado** da API HTTP (`src/worker.ts` como novo entry-point) e consome eventos por **polling a cada 2 segundos**.

## Alternativas Consideradas

1. **Worker no mesmo processo da API**
   - *Prós:* compartilha `PrismaClient`, menos deploys.
   - *Contras:* se a API reinicia (deploy, crash, SIGTERM), o worker morre junto. Perde-se a entrega de eventos durante a janela de downtime.
   - *Decisão:* descartado em [09:11] Diego.

2. **Trigger do MySQL notificando processo externo**
   - *Prós:* mais reativo, entregaria eventos imediatamente após o `INSERT`.
   - *Contras:* MySQL não possui mecanismo nativo tipo `NOTIFY/LISTEN` do PostgreSQL. Simular isso com triggers escrevendo em arquivo ou requisitando um endpoint é frágil e acoplado ao banco.
   - *Decisão:* descartado em [09:09] Diego.

## Consequências

### Positivas
- Maior resiliência: deploy da API não afeta processamento de eventos.
- Simplicidade: polling é stateless, fácil de implementar, debugar e escalar horizontalmente no futuro.
- Latência de 2 segundos atende o requisito de negócio (“abaixo de 10 segundos é tempo real") conforme [09:02] Marcos.

### Negativas
- Polling desperdiça ciclos de CPU e batidas no banco quando a outbox está vazia.
- Em cenário de alta escala futuro, a single-worker ficará gargalo. Ordem de entrega global também será limitada a ordenação por `order_id` e `created_at`.
- Requer infraestrutura adicional para monitorar e garantir que o processo worker esteja sempre rodando (ex: supervisord, systemd ou container separado).

## Trade-off Explícito

Aceitamos o overhead de polling e a limitação de throughput de single-worker em troca de confiabilidade operacional e simplicidade. Escalabilidade horizontal e reatividade são deixadas como problemas futuros.
