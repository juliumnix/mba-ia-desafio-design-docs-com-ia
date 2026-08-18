# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **RFC** | 001 |
| **Título** | Outbox MySQL + worker de polling para webhooks outbound de status de pedido |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-08-18 |
| **Revisores** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (Product Manager) |

## 1. Resumo executivo (TL;DR)

Proposta: notificar clientes B2B quando o status de um pedido muda, **sem** chamada HTTP dentro da transação de `changeStatus`.

A mudança de status continua atômica no MySQL. Na **mesma transação**, gravamos um evento na tabela `webhook_outbox`. Um **worker em processo separado** faz polling a cada 2 s, assina o body com **HMAC-SHA256** (secret por endpoint) e entrega **at-least-once**, com `X-Event-Id` para o cliente deduplicar. Falhas seguem 5 tentativas com backoff; o teto vai para DLQ e replay manual de ADMIN.

Não introduzimos Redis, fila externa nem entrega exactly-once. E-mail de alerta, dashboard e rate limit de saída **não** entram nesta fase.

Decisões fechadas: [ADR-001](adrs/ADR-001-outbox-no-mysql.md) · [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) · [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) · [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) · [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) · [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md).

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação em “tempo real” de mudança de status. Hoje eles fazem polling em `GET /orders`, o que encarece a integração. Para eles, qualquer latência **abaixo de 10 segundos** já conta como tempo real. A Atlas condicionou a permanência na plataforma à entrega até o fim do trimestre (fim de novembro).

O OMS em produção (`src/modules/orders`) já tem máquina de estados, transação de estoque e auditoria em `order_status_history`. **Não há** notificação externa, eventos nem filas. A feature é **somente outbound**: nós enviamos; o cliente não envia webhooks para nós.

A transação de `OrderService.changeStatus` já atualiza pedido, histórico e estoque. Colocar HTTP síncrono no meio disso acoplaria disponibilidade do cliente à consistência do pedido — caminho rejeitado na reunião.

## 3. Proposta técnica

Visão geral (o “como construir” está no [FDD](FDD.md)):

```text
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus  ──$transaction──►  orders
                                             order_status_history
                                             stock (se aplicável)
                                             webhook_outbox (snapshot, se houver
                                               endpoint ativo para aquele status)
        │
        ▼  commit
worker (src/worker.ts, poll 2s)
        │
        ▼
HTTPS POST no URL do cliente
  HMAC-SHA256 · X-Event-Id · X-Signature · X-Timestamp · X-Webhook-Id
        │
        ├─ 2xx → marcar entregue
        ├─ falha/timeout 10s → backoff (5 tentativas)
        └─ teto → webhook_dead_letter → replay ADMIN
```

Pontos de desenho (não o detalhe de schemas/rotas):

- **Atomicidade via outbox no MySQL já existente**, sem broker. Índice em status + `created_at`; worker lê só pendentes em batch pequeno. ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))
- **Publicação** por função `publishWebhookEvent(tx, …)` chamada de dentro da transação de `changeStatus`, filtrando na inserção: se nenhum webhook do customer escuta aquele status, não grava linha. ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md))
- **Worker separado**, `npm run worker`, PrismaClient próprio, **um** processo nesta fase. Ordering implícita por `order_id` via `created_at`. ([ADR-005](adrs/ADR-005-worker-separado-em-polling.md))
- **Retry e DLQ** com teto de 5 e tabela `webhook_dead_letter`. ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md))
- **HMAC-SHA256**, secret gerada por nós, única por endpoint, rotação com grace 24 h, URL `https` obrigatória. ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md))
- **At-least-once** com UUID em `X-Event-Id`. ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md))
- **Módulo** `src/modules/webhooks` no padrão atual; erros `WEBHOOK_*`; Pino; `authenticate` no CRUD; `requireRole('ADMIN')` no replay. ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md))

API de configuração (autenticada, `customer_id` no body/query — **não** extraído do JWT): CRUD de endpoints, filtro de status, histórico das últimas 100 deliveries, rotação de secret, replay admin da DLQ. Contratos HTTP completos ficam no FDD.

Prazo interno: **três sprints**, com dois dias úteis de revisão de segurança da Sofia antes do deploy.

## 4. Alternativas consideradas

### 4.1 HTTP síncrono em `changeStatus`

Chamar o URL do cliente dentro da transação (ou logo após o commit, ainda no request do operador).

**Trade-off que levou ao descarte:** a transação já escreve pedido, histórico e estoque; HTTP no meio faz um cliente lento travar transições de outros pedidos. Se o cliente estiver fora, rollback da mudança de status é incoerente com o negócio. ([09:03–09:06] Larissa, Bruno, Diego)

### 4.2 Redis Streams (ou fila externa)

Publicar o evento num stream após o commit e consumir com um worker Redis.

**Trade-off que levou ao descarte:** resolve desacoplamento, mas quebra a atomicidade a menos que se combine com outbox de qualquer forma, e exige infra nova (Redis Cluster) para um time pequeno. Overengineering frente ao MySQL que já temos. ([09:07] Larissa, Diego)

### 4.3 Trigger MySQL para acordar o worker

Evitar polling usando trigger de banco.

**Trade-off que levou ao descarte:** MySQL não tem `LISTEN/NOTIFY`; trigger executa SQL e não notifica processo Node. Improvisar arquivo ou HTTP a partir do banco fica frágil. Polling de 2 s cabe no orçamento de &lt; 10 s. ([09:09] Bruno, Diego)

Exactly-once foi descartado como garantia (ver [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)): coordenação dos dois lados vs. padrão de mercado com `event_id`.

## 5. Questões em aberto

1. **Rate limiting de saída.** Se um customer tiver dezenas de mudanças de status no mesmo minuto, hoje vamos disparar uma chamada por evento. A reunião registrou: observar em produção e decidir depois — **não** implementar throttle nesta fase. ([09:38–09:39] Diego, Larissa)
2. **Escala para múltiplos workers.** Single-worker preserva ordem por `order_id`. Particionar por `order_id` ou lock pessimista fica para quando o volume exigir; até lá é limitação conhecida, não garantia global de ordering. ([09:12–09:13] Diego, Bruno, Larissa)
3. **Autorização do CRUD.** Qualquer role autenticada (`ADMIN` | `OPERATOR`) cadastra webhook nesta fase; endurecer depois. Replay permanece `ADMIN`. ([09:36–09:37] Marcos, Sofia)

Arquivamento da outbox após 30 dias, e-mail quando o webhook falha e dashboard visual foram **adiados/descartados** (escopo), não questões em aberto de arquitetura.

## 6. Impacto e riscos

| Impacto | O quê |
| --- | --- |
| Código | Extensão pontual de `OrderService.changeStatus` para publicar na outbox na mesma `$transaction`. Novo módulo, novas tabelas Prisma, novo processo `src/worker.ts`. Schema/código da aplicação **não** mudam neste RFC — só a proposta. |
| Operação | Deploy de dois processos; monitoramento de fila, retry e DLQ. |
| Cliente | Precisa expor HTTPS, verificar HMAC e deduplicar por `X-Event-Id`. Payload enxuto; detalhes via `GET /orders/:id`. |
| Prazo | Três sprints + janela de review da Sofia. Risco de produto: churn da Atlas se escorregar o trimestre. |

Riscos técnicos principais: cliente offline além da janela de ~15 h (mitigação: DLQ + replay); secret vazada no cliente (mitigação: secret por endpoint + rotação 24 h); payload &gt; 64 KB abortando a transação de status se o snapshot for in-transaction (mitigação: payload sem items, teto generoso segundo a reunião). Rate limit de saída permanece risco observado, sem controle nesta fase.

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Outbox transacional no MySQL |
| [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) | Retry, backoff e DLQ |
| [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 e secret por endpoint |
| [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) | At-least-once e `X-Event-Id` |
| [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) | Worker separado em polling |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso de AppError, Pino, módulo, Zod, `requireRole` |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do payload na inserção |

## 8. Pedido aos revisores

Confirmar que (a) outbox no MySQL é suficiente sem Redis nesta fase, (b) single-worker e ausência de rate limit de saída são limitações aceitáveis para o lançamento, (c) o recorte de API (CRUD + deliveries + rotate + replay ADMIN) cobre o compromisso com Atlas / MaxDistribuição / Nova Cargo. Comentários no FDD para contratos e erros; neste RFC, para abordagem e abertos.
