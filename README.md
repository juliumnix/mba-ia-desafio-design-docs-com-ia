# Da reunião ao documento: webhooks de notificação de pedidos

## Sobre o desafio

Este repositório entrega o pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição literal da reunião técnica (`TRANSCRICAO.md`) e do código do OMS (Node.js + TypeScript + Prisma/MySQL). Nada da aplicação foi implementado: `src/`, `prisma/`, `tests/` e configurações permanecem intactos. O código serviu só como contexto — máquina de estados, transação de `changeStatus`, `AppError`, Pino, `requireRole`, layout dos módulos.

A regra que guiou o pacote: **não inventar**. Cada requisito, decisão, restrição ou path de arquivo nos docs precisa aparecer no [tracker](docs/TRACKER.md) com origem na call ou no fonte. O que a reunião descartou (e-mail, dashboard, Redis, HTTP síncrono, exactly-once) entra como fora de escopo, não como feature.

Enunciado original do desafio: [mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Cursor Cloud Agent (Grok 4.6)** | Ferramenta principal: leitura do repositório, extração da transcrição, redação dos Markdowns, checagem de paths reais e montagem do tracker. |
| **Explore subagent (Cursor)** | Varredura do OMS (módulos, Prisma, erros, auth, logger) para a seção de integração do FDD e o ADR-006, sem depender de memória genérica de “como um OMS costuma ser”. |
| **Git + GitHub (`gh` somente leitura; PR via fluxo do agente)** | Versionar o pacote documental e abrir o pull request de entrega. |

Não houve Copilot Chat, Gemini nem plugins externos de curso neste ciclo: o agente leu o repo inteiro e a transcrição no workspace.

## Workflow adotado

Ordem alinhada ao enunciado (decisões primeiro, produto por último):

1. **Contextualização** — ler `TRANSCRICAO.md` e o código (`order.service.ts`, `order.status.ts`, `http-errors.ts`, `auth.middleware.ts`, `schema.prisma`, `app.ts`). Separar: decisões fechadas, FRs, NFRs, descartes, adiamentos, ganchos de código.
2. **ADRs** — sete arquivos MADR em `docs/adrs/`, um por decisão. ADR-006 cita paths reais.
3. **RFC** — proposta curta para revisão (Larissa autora; Bruno, Diego, Sofia, Marcos revisores). Alternativas e abertos; links para ADRs; sem copiar payloads do FDD.
4. **FDD** — fluxos, contratos HTTP, matriz `WEBHOOK_*`, resiliência, observabilidade, integração com o OMS.
5. **PRD** — problema, público (Atlas / MaxDistribuição / Nova Cargo), métrica &lt; 10 s, FRs, fora de escopo, riscos.
6. **Tracker** — varredura dos docs prontos; toda linha com `TRANSCRICAO` ou `CODIGO`.
7. **README** — este arquivo, escrito no fim, quando o processo já tinha iterações reais para relatar.

Fronteira entre documentos, aplicada na revisão:

- PRD responde *por que / o quê*.
- RFC responde *o que propomos e o que está aberto*.
- ADR responde *por que desta forma*.
- FDD responde *como construir*.
- Tracker responde *de onde veio*.

## Prompts customizados

Os prompts abaixo foram os que de fato direcionaram a extração e a redação. Não é “gere um PRD a partir da transcrição”.

### Prompt 1 — filtrar o que entra e o que não entra

```text
Leia TRANSCRICAO.md no formato [hh:mm] Nome: fala.

Produza três listas, cada item com timestamp e falante:
1) Decisões FECHADAS (a reunião concluiu).
2) Requisitos funcionais e não funcionais explícitos.
3) Itens DESCARTADOS ou ADIADOS (e-mail, dashboard, Redis, síncrono,
   trigger MySQL, exactly-once, rate limit, arquivo 30 dias, multi-worker).

Regra: se não houver timestamp, o item não existe. Não “complete”
com boas práticas de webhook (Stripe, exactly-once, Prometheus, etc.)
a menos que alguém na call tenha dito.
```

### Prompt 2 — FDD ancorado no código real

```text
Com base só em TRANSCRICAO.md e nestes arquivos:
- src/modules/orders/order.service.ts (método changeStatus, $transaction)
- src/modules/orders/order.status.ts
- src/shared/errors/http-errors.ts
- src/middlewares/auth.middleware.ts
- src/middlewares/error.middleware.ts
- src/app.ts, src/routes/index.ts, src/server.ts
- prisma/schema.prisma

Escreva docs/FDD.md acionável. Na seção "Integração com o sistema existente"
cite pelo menos 4 caminhos que EXISTEM no git. Descreva como
publishWebhookEvent(tx, ...) entra na transação atual.

Contratos: prefixo /api/v1 como em app.ts. customer_id NÃO vem do JWT
([09:32] Larissa). Não invente SLA 99,9%. Payload SEM items ([09:43] Diego).

Se precisar de um path que a call não deu (ex.: rotate-secret), marque
como inferência do padrão REST do OMS e registre no TRACKER com Fonte=CODIGO.
```

Um terceiro prompt, usado na passagem RFC vs FDD, foi: *“o RFC não pode ter exemplo de body HTTP nem matriz WEBHOOK_*; isso é FDD. O RFC tem no máximo 4 páginas e fala em decisão.”*

## Iterações e ajustes

Houve mais de uma passagem. Os cortes concretos:

**Iteração 1 — alucinação de escopo.** O primeiro rascunho de PRD/RFC tentava “completar” a feature com e-mail de alerta, dashboard, Redis “se o volume crescer” e SLA de 99,9% de entrega. Nada disso é meta da call. Correção: e-mail e dashboard foram para *Fora de escopo*; Redis ficou só como alternativa *descartada*; a única métrica quantitativa de latência é **&lt; 10 s** ([09:02] Marcos). Rate limit virou questão em aberto, não NFR.

**Iteração 2 — RFC inchado.** A primeira RFC copiava payloads, tabela de erros e o passo a passo do worker (nível FDD). Ficou longa e repetia os ADRs. Correção: RFC reduzida a abordagem, alternativas (síncrono, Redis, trigger) e abertos (rate limit, multi-worker, papéis do CRUD); contratos e `WEBHOOK_*` saíram para o FDD.

**Iteração 3 — paths e derivações.** O FDD chegou a citar um `EventBus` e `src/events/` que **não existem**. Correção: só arquivos do tree atual. O tratamento “4xx = DLQ imediato” também não foi falado na reunião; em vez de vender como decisão, o FDD rotula como derivação operacional e o tracker aponta `FDD-DER-02`.

**Iteração 4 — JWT vs customer_id.** Um rascunho de contrato extraía `customerId` de `req.user`. A call diz o contrário ([09:32] Larissa). Correção: body/query; JWT só autentica o operador.

Ciclos principais até o pacote estável: **quatro** (filtro de escopo → fronteira RFC/FDD → âncora no código → auth do cadastro), mais a varredura final do tracker contra a checklist de aceite.

## Como navegar a entrega

Ordem sugerida de leitura (da decisão ao detalhe, depois a prova de origem):

| Ordem | Arquivo | Para quê |
| --- | ---: | --- |
| 1 | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária da reunião (não alterada) |
| 2 | [`docs/adrs/README.md`](docs/adrs/README.md) | Índice das 7 decisões |
| 3 | [`docs/RFC.md`](docs/RFC.md) | Proposta para revisão da equipe |
| 4 | [`docs/PRD.md`](docs/PRD.md) | Escopo de produto, FRs, métricas, riscos |
| 5 | [`docs/FDD.md`](docs/FDD.md) | Como implementar, contratos, integração |
| 6 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada linha |

ADRs individuais:

- [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry, backoff e DLQ](docs/adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 por endpoint](docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once e X-Event-Id](docs/adrs/ADR-004-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker separado em polling](docs/adrs/ADR-005-worker-separado-em-polling.md)
- [ADR-006 — Reuso dos padrões do OMS](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload](docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md)

```text
.
├── README.md                 ← você está aqui (processo)
├── TRANSCRICAO.md            ← não alterar
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── README.md
        └── ADR-00N-*.md
```
