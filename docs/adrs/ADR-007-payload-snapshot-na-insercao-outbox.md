# ADR-007: Payload Snapshot na Inserção do Outbox

* Status: Aceito
* Decisores: Larissa, Diego, Bruno
* Data: [09:51]–[09:53] na transcrição

## Contexto

Ao inserir um evento na outbox, o sistema pode armazenar apenas o `order_id` (ou referência mínima) e renderizar o payload completo no momento do envio pelo worker, ou já serializar o payload JSON no momento da inserção.

## Decisão

Armazenar o **payload renderizado (snapshot)** no campo `payload` da `webhook_outbox` no momento da criação do evento.

## Alternativas Consideradas

1. **Armazenar somente `order_id` e renderizar no envio**
   - *Prós:* menor uso de espaço na tabela, payload sempre “fresco” se o pedido sofrer alterações posteriores.
   - *Contras:* se o pedido mudar depois (ex: notas adicionadas), o evento emitido refletiria o estado atual, não o estado no momento exato da transição de status. Isso distorce a história. Também exige JOIN adicional no worker, aumentando o custo do polling.
   - *Decisão:* descartado em [09:52] Larissa e Diego.

## Consequências

### Positivas
- Eventual consistência temporal: o evento reflete fielmente o estado do pedido naquele instante, o que é esperado por sistemas de auditoria e integração.
- Worker mais simples e rápido: lê o payload pronto, sem precisar consultar a tabela `orders` novamente.
- Reprocessamento de DLQ é determinístico: o mesmo payload é reenviado sem regeneração.

### Negativas
- Maior ocupação de disco na `webhook_outbox`; payload pode duplicar dados já existentes na tabela `orders`.
- Se o formato do payload mudar em evoluções futuras, eventos antigos na outbox/DLQ terão o formato antigo. Isso é aceitável porque representam fatos ocorridos no passado.
- Alterações de schema do pedido não se refletem automaticamente em eventos pendentes antigos; migração de formato exige versão no payload.

## Trade-off Explícito

Preferimos exatidão histórica e simplicidade do worker sobre economia de espaço de banco. O armazenamento é barato; a inconsistência temporal de eventos é cara.
