# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint

* Status: Aceito
* Decisores: Sofia, Bruno, Larissa, Diego
* Data: [09:19]–[09:23] na transcrição

## Contexto

Webhooks enviam dados sensíveis de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa poder verificar que a requisição realmente vem da nossa plataforma e que o payload não foi alterado em trânsito.

## Decisão

Usar **HMAC-SHA256** assinando o corpo JSON do request. A assinatura é enviada no header `X-Signature`. Cada endpoint de webhook cadastrado possui uma **secret única**, gerada automaticamente pelo sistema na criação do webhook. O sistema oferece endpoint de rotação de secret; a secret anterior permanece válida por **24 horas** (grace period) após a rotação.

### Requisitos adicionais decorrentes
- URLs de webhook devem ser `https://`; requisições para `http://` são rejeitadas na validação do schema Zod ([09:23] Sofia).
- Limite de payload de 64 KB; eventos que extrapolam são rejeitados antes do disparo ([09:23]–[09:24]).

## Alternativas Consideradas

1. **Secret global da plataforma**
   - *Prós:* implementação mais simples, um único segredo para validar todas as requisições.
   - *Contras:* se uma secret vaza, impacta 100 % dos clientes. Sofríamos um incidente anterior em que cliente vazou secret em log próprio.
   - *Decisão:* descartado em [09:21] Sofia.

2. **mTLS (mutual TLS)**
   - *Prós:* autenticação forte no nível de transporte, sem necessidade de assinatura manual.
   - *Contras:* exige gestão de certificados para cada cliente B2B; aumenta muito a fricção de onboarding e a complexidade operacional.
   - *Decisão:* não foi sequer discutida formalmente na reunião, pois Sofia propôs HMAC como padrão de mercado imediatamente.

## Consequências

### Positivas
- Compatível com a maioria das plataformas de recebimento de webhooks (Stripe, GitHub, etc.).
- Isolamento de blast radius: vazamento de secret afeta apenas um endpoint.
- Grace period de 24 h permite que clientes atualizem seus sistemas sem downtime.

### Negativas
- Cada endpoint adiciona um segredo a ser armazenado e protegido no banco (criptografia em repouso recomendada no futuro).
- Implementação do grace period exige verificar duas secrets durante a janela de transição, aumentando ligeiramente a complexidade do worker.
- Clientes precisam entender como verificar HMAC; exige documentação clara no portal de desenvolvedores.

## Trade-off Explícito

Trocamos simplicidade de implementação interna por segurança do cliente. Isolamento de secret é preferido a facilidade de gerenciamento centralizado.
