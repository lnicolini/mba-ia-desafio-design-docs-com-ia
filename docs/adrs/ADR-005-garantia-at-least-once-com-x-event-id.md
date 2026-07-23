# ADR-005: Garantia At-Least-Once com X-Event-Id

* Status: Aceito
* Decisores: Diego, Larissa, Sofia, Marcos
* Data: [09:24]–[09:26] na transcrição

## Contexto

Em sistemas de entrega de eventos por HTTP, falhas de rede podem fazer com que o cliente receba o evento e nossa plataforma não receba a confirmação (ACK). Isso leva a duplicatas. A reunião precisava definir se o sistema garantiria exactly-once ou at-least-once.

## Decisão

O sistema oferece garantia **at-least-once**. Cada evento recebe um `event_id` (UUID) gerado na inserção da outbox. Esse ID é enviado no header HTTP `X-Event-Id` em todo request de webhook. A responsabilidade de deduplicação (`dedup`) fica com o cliente.

## Alternativas Consideradas

1. **Exactly-once**
   - *Prós:* elimina duplicatas, simplifica a vida do consumidor.
   - *Contras:* exige coordenação de dois lados (confirmação idempotente e estado de "já entregue" infalível). Sem controle do código do cliente, não é possível garantir de forma robusta. Stripe, GitHub e outros adotam at-least-once por esse mesmo motivo.
   - *Decisão:* descartado em [09:25] Diego.

2. **At-least-once sem identificador único**
   - *Prós:* ainda mais simples de implementar.
   - *Contras:* cliente teria que deduplicar por conteúdo do payload, que pode variar levemente (timestamps, por exemplo). Não é confiável.
   - *Decisão:* implicitamente descartado; todos concordaram com `X-Event-Id`.

## Consequências

### Positivas
- Implementação simples e robusta do lado do emissor.
- Alinhamento com padrão de mercado; clientes já esperam comportamento similar de outras plataformas.
- O `X-Event-Id` também serve como chave de rastreabilidade nos logs e no painel de deliveries.

### Negativas
- Contrato com o cliente exige documentação explícita: “implemente dedup pelo X-Event-Id”.
- Em casos de reprocessamento de DLQ, o mesmo `event_id` será reutilizado; se o cliente já processou e descartou a primeira entrega, a segunda será corretamente ignorada, mas se o cliente não deduplicou, verá duplicata.
- Eventos podem ser entregues fora de ordem se surgirem múltiplos workers no futuro, exigindo que o cliente ordene por `order_id` + timestamp interno se a ordem importar.

## Trade-off Explícito

Preferimos simplicidade de implementação e robustez da entrega sobre a conveniência do consumidor. O custo da deduplicação é transferido para o cliente, o que é padrão no ecossistema de webhooks.
