# ADR-003: Política de Retry com Backoff Exponencial e DLQ

* Status: Aceito
* Decisores: Diego, Larissa, Bruno, Marcos
* Data: [09:14]–[09:19] na transcrição

## Contexto

Clientes B2B podem ficar indisponíveis por manutenções planejadas ou falhas de rede. O sistema precisa retentar a entrega antes de considerar um evento como falho permanente, mas também não pode acumular tentativas infinitas.

## Decisão

Implementar **retry com backoff exponencial fixo** e uma **Dead Letter Queue (DLQ)** persistente.

| # tentativa | Delay desde a tentativa anterior |
|-------------|----------------------------------|
| 1           | 1 minuto                         |
| 2           | 5 minutos                        |
| 3           | 30 minutos                       |
| 4           | 2 horas                          |
| 5           | 12 horas                         |

Após a 5ª tentativa fracassada, o evento é movido para a tabela `webhook_dead_letter`. O reprocessamento é manual via endpoint administrativo.

## Alternativas Consideradas

1. **Retry indefinido com backoff exponencial**
   - *Prós:* nunca perde um evento por exaustão de retry.
   - *Contras:* evento de um cliente que sumiu permanece pendurado para sempre, poluindo a outbox e desperdiçando recursos de worker.
   - *Decisão:* descartado em [09:15]–[09:16]; Diego argumentou que uma indisponibilidade de 2 h já foi observada com clientes reais.

2. **3 tentativas, mais agressivas (1 min, 5 min, 15 min)**
   - *Prós:* descarta eventos rapidamente, mantém a outbox enxuta.
   - *Contras:* janela de cobertura de apenas ~21 minutos. Uma manutenção planejada de 2 horas seria suficiente para matar todos os eventos.
   - *Decisão:* descartado em [09:16] Bruno propôs, mas Diego convenceu a manter 5 tentativas.

## Consequências

### Positivas
- Cobertura de até ~15 horas de indisponibilidade do cliente, atendendo a casos reais de manutenção.
- DLQ separada mantém a outbox principal performática e oferece registro de evidência para debugging.
- Reprocessamento manual permite recuperação controlada sem reenviar tudo automaticamente e possivelmente causar mais dano.

### Negativas
- Delay de 12 horas na última tentativa significa que evento pode ficar “quase morto” na outbox por muito tempo antes de ir para a DLQ.
- Sem reprocessamento automático, operadores precisam agir manualmente; se ninguém monitorar a DLQ, eventos ficam perdidos.
- O cálculo de `next_retry_at` exige campo e lógica adicionais na outbox.

## Trade-off Explícito

Preferimos confiabilidade de entrega (cobrir manutenções longas) sobre a agressividade de descarte. Aceitamos a complexidade de gerenciar DLQs e reprocessamento manual como contrapartida.
