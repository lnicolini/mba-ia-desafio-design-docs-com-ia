# ADR-001: Padrão Outbox no MySQL

* Status: Aceito
* Decisores: Larissa, Diego, Bruno
* Data: [09:06]–[09:08] na transcrição

## Contexto

O Webhook de Notificação de Pedidos precisa ser disparado toda vez que o status de um pedido muda, sem comprometer a transação principal de alteração de status. A reunião levantou três possibilidades: (1) chamada síncrona dentro do `OrderService`, (2) uma fila dedicada externa (Redis Streams), ou (3) padrão Outbox usando o próprio MySQL existente.

A chamada síncrona foi descartada imediatamente porque qualquer latência ou falha no cliente travaria o commit da transação de pedido. Redis Streams exigiria provisionar e operar nova infraestrutura (cluster Redis), o que não é viável para o tamanho atual do time.

## Decisão

Adotar o padrão **Outbox** com uma tabela `webhook_outbox` no banco MySQL já existente.

Quando o status de um pedido muda dentro de uma transação Prisma (`$transaction`), a mesma transação insere uma linha na `webhook_outbox` contendo o evento serializado. Se a transação commita, o evento existe; se rollback, o evento some junto. Um worker independente faz polling na tabela e dispara os HTTP requests.

## Alternativas Consideradas

1. **Chamada síncrona no `OrderService`**
   - *Prós:* implementação trivial, latência zero.
   - *Contras:* falha no cliente gera rollback da mudança de status; cliente lento bloqueia outros pedidos; impossível separar preocupações.
   - *Decisão:* descartado em [09:04] Bruno e confirmado por Diego em [09:06].

2. **Redis Streams / fila externa**
   - *Prós:* maior throughput, baixa latência, desacoplamento total.
   - *Contras:* operar Redis Cluster é overengineering para o volume atual; aumenta superfície de infra, custo e complexidade de deploy.
   - *Decisão:* descartado em [09:07] Diego.

## Consequências

### Positivas
- Consistência transacional garantida entre mudança de status e emissão do evento, sem coordenação distribuída adicional.
- Zero infraestrutura nova; reuse do pool de conexões e backups já existentes.
- Worker pode ser desenvolvido e testado incrementalmente sem alterar a API principal.

### Negativas
- O worker compete pelo mesmo recurso de I/O do banco que a API; carga de polling contínuo pode afetar consultas da aplicação se mal dimensionado.
- Latência mínima do evento é limitada pelo intervalo de polling (2 s nesta fase), não sendo verdadeiramente instantâneo.
- Necessidade de estratégia de arquivamento de eventos entregues para evitar crescimento ilimitado da tabela.

## Trade-off Explícito

Trocamos latência máxima e throughput ilimitado pela simplicidade operacional e pela garantia de atomicidade. Optamos por resolver o problema de entrega confiável antes de otimizar a velocidade.
