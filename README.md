# Desafio: Da Reunião ao Documento — Design Docs Gerados por IA

Este repositório contém a entrega do desafio de transformar a transcrição de uma reunião técnica em um pacote completo de design docs para uma feature de Sistema de Webhooks de Notificação de Pedidos. O objetivo central foi usar IA como ferramenta de produção, assumindo o papel de maestro — definindo o que precisava ser feito, formulando prompts direcionados, revisando criticamente o conteúdo gerado e refinando até atingir consistência e rastreabilidade totais.

A documentação produzida cobre todas as camadas necessárias para o time de engenharia iniciar a implementação: decisões arquiteturais (ADRs), proposta técnica para revisão (RFC), especificação detalhada de implementação (FDD), requisitos do produto (PRD) e rastreabilidade completa de cada item à transcrição ou ao código (TRACKER). Todo o conteúdo é estritamente ancorado na transcrição da reunião (`TRANSCRICAO.md`) e na estrutura real da aplicação Node.js + TypeScript existente no repositório.

**Link para o repositório base (enunciado original):** <https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia>

---

## Ferramentas de IA Utilizadas

- **Claude Code:** agente principal e orquestrador do processo, executando com o modelo **Claude Opus 4.7**. Realizou a leitura do código, análise da transcrição, geração e refinamento dos documentos em Markdown, e escrita efetiva dos arquivos no filesystem.
- **Claude Opus 4.7 (modelo base):** forneceu o raciocínio estratégico para estruturar ADRs, decidir fronteiras entre RFC e FDD, e validar rastreabilidade. Nenhum conteúdo foi copiado sem revisão crítica.

---

## Workflow Adotado

1. **Contextualização:** leitura completa de `TRANSCRICAO.md` e exploração da estrutura de `src/`, `prisma/`, tests e configs para entender padrões (AppError, Pino, Zod, Prisma, Express modules).
2. **Inventário de decisões:** varredura da transcrição identificando decisões fechadas, alternativas descartadas, pontos em aberto e requisitos funcionais e não-funcionais. Isso foi feito manualmente (por mim, o agente) antes de qualquer geração de documento.
3. **ADRs primeiro:** produzi 7 ADRs isolando cada decisão arquitetural principal. Isso formou o esqueleto do “como implementar” e evitou que o RFC e o FDD se tornassem confusos.
4. **RFC em seguida:** consolidei a visão arquitetural geral, referenciando ADRs já escritos e mantendo o documento conciso (arquitetura, não implementação).
5. **FDD depois:** com decisões formalizadas, detalhei contratos, fluxos, matriz de erros, resiliência, observabilidade e a seção obrigatória de integração com 6 caminhos de arquivo reais da base de código.
6. **PRD por último entre os grandes:** como é mais alto nível, tornou-se uma consolidação natural do que estava nos ADRs, RFC e FDD, alinhado ao discurso do PM (Marcos) na transcrição.
7. **Tracker em paralelo / final:** após cada documento, mapeei os itens principais para a tabela de rastreabilidade, garantindo que não houvesse alucinação.
8. **README (processo):** documentação final do workflow, escrita com o processo completo em mente.

---

## Prompts Customizados

Abaixo estão dois prompts que foram centrais para guiar a produção de conteúdo com qualidade e sem alucinação.

### Prompt 1 — Extração estruturada da transcrição

```
Leia a transcrição fornecida (TRANSCRICAO.md) e produza um JSON estruturado contendo:
1. Decisões fechadas (com timestamp e quem decidiu)
2. Alternativas discutidas e descartadas (com quem argumentou contra e o trade-off)
3. Requisitos funcionais explícitos (com timestamp)
4. Requisitos não-funcionais / restrições
5. Pontos em aberto ou adiados para fases futuras
6. Itens explicitamente fora de escopo

Não invente nada que não esteja na transcrição. Se algo for implícito mas não confirmado, marque como "implícito".
```

### Prompt 2 — Geração de ADR com formato MADR e rastreabilidade

```
Com base na decisão: "[descrição da decisão]", extraída da transcrição em [timestamp] [falante],
redija um ADR no formato MADR com as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 com trade-off explícito), Consequências Positivas e Negativas.

Cada alternativa descartada deve citar o timestamp da transcrição onde foi discutida.
As consequências devem incluir um trade-off explícito.
Não adicione tecnologias ou requisitos que não apareçam na transcrição.
```

---

## Iterações e Ajustes

### Ajuste 1 — Fronteira entre RFC e FDD

Na primeira iteração, o conteúdo gerado para o RFC continha pseudo-código detalhado do worker e payloads completos de endpoints. Isso violou a regra do desafio: "O RFC é conciso (2 a 4 páginas) e fala em decisão; o FDD é profundo e fala em implementação". Revisei o documento, removi pseudo-código e payloads, transferindo-os para o FDD. O RFC foi reescrito para focar em visão geral, alternativas, questões em aberto e riscos.

### Ajuste 2 — Tracker incompleto e timestamps genéricos

Na primeira versão do Tracker, algumas linhas usavam descrições vagas como "discutido na reunião" em vez de `[hh:mm] Nome`. A correção exigiu revarrer a transcrição e atualizar todas as fontes para o formato exato `[hh:mm] Participante`, validando que pelo menos 70 % das entradas tivessem essa forma. Também adicionei as 5+ linhas com `Fonte = CODIGO` e caminhos reais, que inicialmente estavam sub-representadas.

### Ajuste 3 — Erros inventados no payload e headers

Em um momento inicial da elaboração do FDD, a IA sugeriu headers extras (como `X-Retry-Count`, `X-Payload-Version`) que não constavam da transcrição. Após confrontar com o tracker, identifiquei que esses itens não tinham origem identificável. Foram removidos, mantendo apenas os headers discutidos: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`.

---

## Como Navegar a Entrega

A ordem sugerida de leitura reflete o princípio de que "as decisões formam o esqueleto":

1. **docs/adrs/** — Comece pelos ADRs para entender cada decisão arquitetural isolada, suas alternativas e trade-offs.
2. **docs/RFC.md** — Leia a proposta técnica consolidada, com visão geral, alternativas descartadas e questões em aberto.
3. **docs/FDD.md** — Aprofunde no nível de implementação: fluxos, contratos HTTP, matriz de erros, resiliência, observabilidade e integração com código existente.
4. **docs/PRD.md** — Veja a visão de produto: requisitos, escopo, métricas de sucesso, riscos e critérios de aceitação.
5. **docs/TRACKER.md** — Use como referência cruzada para verificar, para qualquer item dos documentos acima, de onde ele veio na transcrição ou no código.
6. **TRANSCRICAO.md** — Fonte primária da reunião (não alterada).

**Arquivos entregues:**

```
.
├── README.md                              (este arquivo)
├── TRANSCRICAO.md                         (fonte da reunião)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-polling-em-processo-separado.md
│       ├── ADR-003-retry-backoff-exponencial-e-dlq.md
│       ├── ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-garantia-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
│       └── ADR-007-payload-snapshot-na-insercao-outbox.md
├── src/                                   (código base — referência apenas)
├── prisma/                                (schema base — referência apenas)
└── tests/                                 (testes base — referência apenas)
```

---

## Checklist de Critérios de Aceite (Auto-avaliação)

- [x] PRD existe com todas as seções obrigatórias; ≥ 8 requisitos funcionais; ≥ 1 meta quantitativa; ≥ 2 itens fora de escopo; ≥ 2 riscos com prob/impacto/mitigação.
- [x] RFC existe com metadados, TL;DR, contexto, proposta, alternativas (≥ 2 descartadas com trade-offs), questões em aberto (≥ 2), links para ADRs.
- [x] FDD existe com fluxos, contratos (≥ 4 endpoints), matriz de erros WEBHOOK_*, integração com ≥ 4 arquivos reais, observabilidade (métricas/logs/tracing).
- [x] 7 ADRs em `docs/adrs/` no formato correto, cobrindo as 6 decisões principais + payload snapshot; pelo menos 1 referencia arquivos do código existente.
- [x] TRACKER com tabela padronizada; ≥ 80 % de cobertura; ≥ 70 % com timestamp `[hh:mm] Nome`; ≥ 5 linhas Fonte = CODIGO.
- [x] README com workflow, ferramentas, ≥ 2 prompts em blocos de código, ≥ 2 iterações/ajustes, e mapa de navegação.
- [x] Nenhum item contradiz a transcrição ou o código; nenhum arquivo de código citado é inexistente.
