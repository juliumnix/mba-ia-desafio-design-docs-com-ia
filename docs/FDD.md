# FDD — Sistema de Webhooks de Notificação de Pedidos

Documento de implementação. Decisões fechadas: [RFC](RFC.md) e [ADRs](adrs/README.md). Este texto descreve **como construir**.

## 1. Contexto e motivação técnica

O OMS persiste mudança de status em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) dentro de `prisma.$transaction`: valida transição (`src/modules/orders/order.status.ts`), atualiza `orders`, grava `order_status_history` e ajusta estoque. Não existe hoje publicação externa.

A feature precisa (1) não bloquear essa transação com HTTP, (2) não perder o evento se o commit ocorreu, (3) entregar com HMAC e retries sem exactly-once. A abordagem é outbox no MySQL + worker separado — ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-005](adrs/ADR-005-worker-separado-em-polling.md).

## 2. Objetivos técnicos

- Inserir evento na outbox **na mesma transação** que a mudança de status; rollback conjunto se a inserção falhar.
- Worker em `src/worker.ts` (`npm run worker`), polling 2 s, timeout HTTP 10 s, 5 tentativas com backoff 1m / 5m / 30m / 2h / 12h, depois `webhook_dead_letter`.
- Módulo `src/modules/webhooks` no padrão existente; erros `WEBHOOK_*`; Pino; Zod; `authenticate` / `requireRole('ADMIN')`.
- Payload snapshotado na inserção ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)), ≤ 64 KB, sem `items`.
- Latência de ponta a ponta compatível com &lt; 10 s em caminho feliz (poll 2 s + HTTP).

## 3. Escopo e exclusões

**No escopo:** tabelas Prisma (configuração, outbox, deliveries, DLQ), CRUD autenticado, filtro de status na inserção, HMAC + rotação 24 h, histórico das últimas 100 deliveries, replay ADMIN, worker + processor, extensão pontual de `changeStatus`.

**Fora:** e-mail de alerta, dashboard, rate limit de saída, Redis, trigger MySQL, exactly-once, arquivamento 30 dias, múltiplos workers, inbound webhooks.

## 4. Modelagem persistida (Prisma)

Novos models em `prisma/schema.prisma`, IDs `@default(uuid()) @db.Char(36)` como o restante do schema. Enums sugeridos:

```text
WebhookOutboxStatus   PENDING | PROCESSING | FAILED | DELIVERED
WebhookDeliveryResult SUCCESS | FAILURE
```

| Tabela | Papel |
| --- | --- |
| `webhook_endpoints` | `customerId`, `url` (https), `secret` atual, `previousSecret` + `previousSecretExpiresAt` (grace 24 h), `subscribedStatuses` (JSON array de `OrderStatus`), `active`, timestamps |
| `webhook_outbox` | `eventId` (UUID = `X-Event-Id`), `webhookEndpointId`, `orderId`, `customerId`, `status`, `payload` (JSON snapshot), `attemptCount`, `nextAttemptAt`, `lastError`, `createdAt` |
| `webhook_deliveries` | histórico por envio: `outboxId`, `webhookEndpointId`, HTTP status, body de response truncado, `durationMs`, `result`, `attemptNumber`, `createdAt` |
| `webhook_dead_letter` | cópia do payload + motivo + `failedAt` + referência ao endpoint/evento; origem para replay |

Índices mínimos (reunião): outbox em `status` + `createdAt`; worker seleciona `PENDING` com `nextAttemptAt <= now()`, `ORDER BY created_at`, batch pequeno (ex.: 10–50; o tamanho exato do batch não foi fechado na call — usar constante no processor, ajustar com métricas).

Secret em `webhook_endpoints` nunca deve aparecer em logs (estender redact em `src/shared/logger/index.ts` com `*.secret`, `*.previousSecret`, analogia a `*.password` / `*.token`).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Disparo **somente** em `changeStatus` (não em `create` com `fromStatus: null`). Máquina de estados inalterada em `src/modules/orders/order.status.ts`.

```text
OrderService.changeStatus
  prisma.$transaction(tx =>
    1. load order + items
    2. validar transição (já existente)
    3. debit/replenish stock (já existente)
    4. tx.order.update status
    5. tx.orderStatusHistory.create
    6. await publishWebhookEvent(tx, order, from, to)   // NOVO
    7. reload and return
  )
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)`:

1. Buscar endpoints ativos do `order.customerId` cujo `subscribedStatuses` contém `toStatus`.
2. Se nenhum, return (não grava outbox).
3. Para cada endpoint: montar snapshot JSON (seção 6.8); se `Buffer.byteLength(json, 'utf8') > 64 * 1024`, lançar erro `WEBHOOK_PAYLOAD_TOO_LARGE` (a transação dá rollback — alinhado a “erra, não trunca”).
4. `tx.webhookOutbox.create` com `eventId` UUID, `status: PENDING`, `attemptCount: 0`, `nextAttemptAt: now()`, `payload` = snapshot.

Não injetar `WebhookRepository` em `OrderService`; a função recebe `Prisma.TransactionClient`.

### 5.2 Processamento pelo worker

`src/worker.ts`: bootstrap (PrismaClient próprio via o mesmo padrão de `src/config/database.ts`, logger Pino), loop:

1. Sleep 2 s (ou sleep no fim do ciclo para não sobrepor se o batch passar de 2 s).
2. Selecionar lote `PENDING` com `nextAttemptAt <= now()`, mais antigos primeiro.
3. Marcar `PROCESSING` (optimistic: `UPDATE … WHERE status = PENDING`) para o single-worker não reprocessar a mesma linha no próximo tick se o HTTP atrasar.
4. Para cada linha, `WebhookProcessor.process(row)`.
5. SIGINT/SIGTERM: parar o loop, `prisma.$disconnect()` — espelhar shutdown de `src/server.ts`.

`WebhookProcessor.process`:

1. Carregar endpoint; se inativo, tratar como falha permanente → DLQ (não retentar URL desativado).
2. POST HTTPS, timeout 10 s, headers da seção 6.8, body = `payload` snapshotado.
3. Assinatura: HMAC-SHA256 do **raw body** com a secret atual; header `X-Signature` (hex ou `sha256=<hex>` — o FDD fixa `sha256=<hex>` em minúsculas para o cliente).
4. Durante grace de rotação, o cliente pode verificar com secret nova ou antiga; **nós enviamos só a secret atual**. A antiga serve para o cliente que ainda não migrou verificar o que **já estava em trânsito** e o que enviarmos se a implementação optar por assinar com as duas em headers distintos — a reunião especificou apenas que a antiga permanece **válida 24 h no cliente**. Implementação: enviar `X-Signature` com a secret **atual**; documentar no portal que, na janela, o cliente deve aceitar assinatura da nova **ou** da antiga (se ainda tiver a antiga). Não enviar duas assinaturas a menos que o portal peça depois.
5. Resposta 2xx → outbox `DELIVERED`; gravar `webhook_deliveries` SUCCESS.
6. Timeout, 5xx, rede, 429, 408 → falha transitória (seção 5.3).
7. 4xx (exceto 408/429) → falha permanente → DLQ (URL recusou de forma não recuperável). A reunião não distinguiu 4xx/5xx; esta regra é a interpretação operacional mínima para não gastar 15 h em 401/404. Registrar no Tracker como derivação da política de DLQ ([09:17] Diego), não como fala literal.

### 5.3 Retry

Falha transitória:

- `attemptCount += 1`
- Se `attemptCount < 5`: `status = PENDING`, `nextAttemptAt = now() + backoff[attemptCount-1]` onde backoff = `[60s, 300s, 1800s, 7200s, 43200s]`, gravar delivery FAILURE, `lastError`.
- Se `attemptCount >= 5`: seguir 5.4.

A 1ª tentativa é o poll inicial (`attemptCount` 0 → 1). As cinco janelas de espera aplicam-se **entre** tentativas após falha.

### 5.4 DLQ e replay

Mover para `webhook_dead_letter` (payload, motivo, timestamp, ids), marcar outbox `FAILED` (ou deletar a linha pendente — preferir manter `FAILED` para auditoria local, DLQ como evidência operacional).

Replay `POST /api/v1/admin/webhooks/dead-letter/:id/replay`:

- `authenticate` + `requireRole('ADMIN')`
- Log Pino obrigatório: `webhook_dlq_replay` com `adminUserId`, `deadLetterId`, `eventId`
- Recriar (ou reabrir) linha na outbox `PENDING`, `attemptCount = 0`, `nextAttemptAt = now()`
- Não apagar a DLQ (histórico); marcar `replayedAt` se o model tiver o campo

## 6. Contratos públicos

Prefixo: `/api/v1` (`src/app.ts`). Auth: `Authorization: Bearer <jwt>` (`src/middlewares/auth.middleware.ts`). Validação: `validate()` + Zod (`src/middlewares/validate.middleware.ts`). Erros: `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`).

`customerId` vai no **body ou query**, nunca extraído do JWT.

Respostas de criação seguem o estilo dos controllers atuais (`res.status(201).json(recurso)`). Listagens podem usar `paginated()` de `src/shared/http/response.ts` — para deliveries a reunião pediu **últimos 100**, então o GET de deliveries devolve array limitado a 100, sem paginação obrigatória.

Secret **só** aparece na criação e na rotação. GET/PATCH/LIST nunca retornam o valor da secret (apenas `secretLastFour` se útil; não foi pedido — omitir o campo).

### 6.1 `POST /api/v1/webhooks` — cadastrar

**Auth:** `authenticate` (ADMIN ou OPERATOR).

Request:

```http
POST /api/v1/webhooks HTTP/1.1
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://hooks.atlas.example/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201`:

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://hooks.atlas.example/oms",
  "secret": "whsec_8f3a1c0e2b9d4a76c5e1f0a2b3c4d5e6",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-08-18T12:00:00.000Z",
  "updatedAt": "2026-08-18T12:00:00.000Z"
}
```

A secret é gerada pelo servidor (não aceitar secret no body).

| Status | Quando |
| --- | --- |
| 201 | criado |
| 400 | URL http, status inválido, UUID inválido (`VALIDATION_ERROR` ou `WEBHOOK_INVALID_URL`) |
| 401 | sem JWT |
| 404 | `customerId` inexistente (`WEBHOOK_CUSTOMER_NOT_FOUND` ou `NotFoundError('Customer')` já existente) |

### 6.2 `GET /api/v1/webhooks` — listar por customer

```http
GET /api/v1/webhooks?customerId=3fa85f64-5717-4562-b3fc-2c963f66afa6 HTTP/1.1
Authorization: Bearer <jwt>
```

Response `200` (espelha `paginated()`):

```json
{
  "data": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "url": "https://hooks.atlas.example/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-18T12:00:00.000Z",
      "updatedAt": "2026-08-18T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| 200 | lista (pode ser vazia) |
| 400 | `customerId` ausente ou inválido |
| 401 | sem JWT |

### 6.3 `PATCH /api/v1/webhooks/:id` — editar

```http
PATCH /api/v1/webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7 HTTP/1.1
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "url": "https://hooks.atlas.example/oms/v2",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200`: mesmo shape do item em GET (sem secret).

| Status | Quando |
| --- | --- |
| 200 | atualizado |
| 400 | URL http / payload inválido |
| 401 | sem JWT |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.4 `DELETE /api/v1/webhooks/:id` — remover

```http
DELETE /api/v1/webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7 HTTP/1.1
Authorization: Bearer <jwt>
```

Response: `204` corpo vazio (igual `OrderController.delete`).

| Status | Quando |
| --- | --- |
| 204 | removido |
| 401 | sem JWT |
| 404 | `WEBHOOK_NOT_FOUND` |

Eventos já na outbox **não** são apagados pelo DELETE (a reunião não pediu cascade). Worker: se o endpoint não existir mais, falha permanente → DLQ.

### 6.5 `GET /api/v1/webhooks/:id/deliveries` — últimos 100 envios

```http
GET /api/v1/webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7/deliveries HTTP/1.1
Authorization: Bearer <jwt>
```

Response `200`:

```json
{
  "data": [
    {
      "id": "0f1e2d3c-4b5a-6978-90ab-cdef12345678",
      "webhookId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "eventId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "success": true,
      "httpStatus": 200,
      "payload": {
        "event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
        "event_type": "order.status_changed",
        "timestamp": "2026-08-18T12:01:03.000Z",
        "order_id": "11111111-1111-1111-1111-111111111111",
        "order_number": "ORD-000042",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "total_cents": 12990
      },
      "responseBody": "{\"ok\":true}",
      "durationMs": 142,
      "createdAt": "2026-08-18T12:01:05.210Z"
    }
  ]
}
```

Limite: 100 mais recentes (`ORDER BY createdAt DESC LIMIT 100`).

| Status | Quando |
| --- | --- |
| 200 | histórico |
| 401 | sem JWT |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.6 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

A reunião pediu endpoint de rotação; o path exato não foi falado. Segue o padrão REST do OMS (`/resource/:id/ação`, como `PATCH /orders/:id/status`).

```http
POST /api/v1/webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7/rotate-secret HTTP/1.1
Authorization: Bearer <jwt>
```

Response `200`:

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "secret": "whsec_novo_valor_gerado",
  "previousSecretExpiresAt": "2026-08-19T12:30:00.000Z"
}
```

A secret anterior permanece válida para verificação no cliente até `previousSecretExpiresAt` (now + 24 h).

| Status | Quando |
| --- | --- |
| 200 | nova secret |
| 401 | sem JWT |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

**Auth:** `authenticate` + `requireRole('ADMIN')` — mesmo encadeamento de `src/modules/users/user.routes.ts`.

```http
POST /api/v1/admin/webhooks/dead-letter/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/replay HTTP/1.1
Authorization: Bearer <jwt admin>
```

Response `202`:

```json
{
  "deadLetterId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "outboxId": "bbbbbbbb-cccc-dddd-eeee-ffffffffffff",
  "eventId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "PENDING"
}
```

| Status | Quando |
| --- | --- |
| 202 | reenfileirado |
| 401 | sem JWT |
| 403 | JWT não ADMIN (`FORBIDDEN`) |
| 404 | DLQ id inexistente (`WEBHOOK_DEAD_LETTER_NOT_FOUND`) |

### 6.8 Entrega outbound (não é rota nossa — contrato com o cliente)

```http
POST /oms HTTP/1.1
Host: hooks.atlas.example
Content-Type: application/json
X-Event-Id: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d
X-Webhook-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Timestamp: 2026-08-18T12:01:03.000Z
X-Signature: sha256=3f1c0a9b...

{
  "event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-18T12:01:03.000Z",
  "order_id": "11111111-1111-1111-1111-111111111111",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "total_cents": 12990
}
```

Sem `items`. Cliente que precisar de linhas do pedido chama `GET /api/v1/orders/:id`. `from_status` / `to_status` usam os valores do enum Prisma `OrderStatus`. HMAC sobre o body UTF-8 exatamente como enviado. Cliente 2xx = sucesso.

## 7. Matriz de erros `WEBHOOK_*`

Novas classes em `src/shared/errors/http-errors.ts` (ou arquivo `webhook-errors.ts` reexportado em `src/shared/errors/index.ts`), sempre estendendo `AppError`. Prefixos pedidos na reunião em [09:28–09:29].

| Código | HTTP | Quando | Classe sugerida |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint id inexistente | `NotFoundError` com code override **ou** subclasse |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, malformada ou não `https` | `BadRequestError` / `ValidationError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige secret internamente e ela não está persistida (estado corrompido) | `BadRequestError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot &gt; 64 KB | `UnprocessableEntityError` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` do cadastro não existe | `NotFoundError('Customer')` ou code dedicado |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de id inexistente | 404 |
| `WEBHOOK_ENDPOINT_INACTIVE` | 422 | Tentativa de operação que exige endpoint ativo (opcional no CRUD) | `UnprocessableEntityError` |

`NotFoundError` atual fixa o code `NOT_FOUND`. Para prefixo `WEBHOOK_`, preferir `new AppError(..., 404, 'WEBHOOK_NOT_FOUND')` ou estender `NotFoundError` permitindo code — **não** quebrar o construtor atual dos outros módulos; adicionar subclasses novas.

Zod continua devolvendo `VALIDATION_ERROR` via error middleware (URL http pode ser recusada no schema com `z.string().url().startsWith('https://')`, o que já gera 400 sem code `WEBHOOK_*`; usar refine e mapear para `WEBHOOK_INVALID_URL` no service se quisermos o code estável).

Códigos existentes reutilizados sem prefixo: `UNAUTHORIZED`, `FORBIDDEN`, `VALIDATION_ERROR`.

## 8. Estratégias de resiliência

| Mecanismo | Valor |
| --- | --- |
| Timeout HTTP | 10 s por tentativa |
| Polling | 2 s |
| Retry | 5 tentativas |
| Backoff | 1m, 5m, 30m, 2h, 12h |
| Fallback | DLQ + replay ADMIN (não há fallback de canal; e-mail fora de escopo) |
| TLS | recusar `http://` no cadastro |
| Tamanho | 64 KB, erro, sem truncate |
| Processo | worker ≠ API; restart da API não mata o worker |
| Assinatura | HMAC-SHA256; rotação 24 h |

Não há circuit breaker nem rate limit de saída nesta fase (questão em aberto do RFC).

## 9. Observabilidade

### Logs (Pino — `src/shared/logger/index.ts`)

Eventos sugeridos (message string estável, como `server_started` / `http_request`):

- `webhook_outbox_enqueued` — `eventId`, `orderId`, `webhookId`, `toStatus` (sem payload completo se passar de um tamanho seguro; secret nunca)
- `webhook_delivery_attempt` — `eventId`, `attempt`, `durationMs`, `httpStatus`
- `webhook_delivery_failed` — `eventId`, `attempt`, `error`
- `webhook_moved_to_dlq` — `eventId`, `reason`
- `webhook_dlq_replay` — `adminUserId`, `deadLetterId`, `eventId` (**obrigatório** pela reunião)

Redact: incluir `*.secret`, `*.previousSecret`, `X-Signature` se logar headers.

### Métricas (contador/gauge no logger ou futuro export; nomes)

Necessárias para a questão em aberto de rate limit (“observar e decidir depois”):

- `webhook_outbox_pending` (gauge)
- `webhook_delivery_success_total` / `webhook_delivery_failure_total`
- `webhook_retry_total` (por attempt)
- `webhook_dlq_total`
- `webhook_delivery_duration_ms` (latência HTTP)
- `webhook_enqueue_to_first_attempt_ms` (ver SLA &lt; 10 s)

Implementação mínima: log estruturado com esses campos; um scraper/dashboard pode vir depois. Não adicionar Prometheus nesta fase (não discutido).

### Tracing

Reusar correlação já existente: `src/middlewares/request-logger.middleware.ts` gera `requestId` (`X-Request-Id`). No `changeStatus`, o log de enqueue deve carregar `requestId` de `req.id` se o service receber o contexto (hoje o service não recebe; o controller pode logar, ou passar `requestId` opcional). No worker, o correlator é `eventId` (`X-Event-Id`) em **todos** os logs do processor. Não introduzir OpenTelemetry nesta fase (não discutido).

## 10. Integração com o sistema existente

### 10.1 `src/modules/orders/order.service.ts`

Único gancho de publicação. Após `tx.order.update` e `tx.orderStatusHistory.create` (hoje linhas 158–167), chamar `publishWebhookEvent(tx, { ...order, status: to }, from, to)` **antes** do `findUnique` de retorno. O `tx` é `Prisma.TransactionClient` (`type TxClient` já existe no arquivo). Falha na outbox aborta a transação inteira — incluindo débito de estoque — o que é o comportamento desejado pela reunião.

Não publicar em `create()` (histórico inicial com `fromStatus: null` / `reason: 'order created'`).

### 10.2 `src/shared/errors/http-errors.ts` e `src/middlewares/error.middleware.ts`

Novas subclasses `AppError` com codes `WEBHOOK_*`. O middleware já ramifica `instanceof AppError` e devolve `{ error: { code, message, details } }`. **Não** alterar o middleware para a feature funcionar, desde que os erros estendam `AppError`.

### 10.3 `src/middlewares/auth.middleware.ts` e `src/modules/users/user.routes.ts`

CRUD de webhook: `router.use(authenticate)` como em `src/modules/orders/order.routes.ts`. Replay: `authenticate, requireRole('ADMIN')` como o `GET /users/:id`. Roles disponíveis: `'ADMIN' | 'OPERATOR'` (Prisma `UserRole`).

### 10.4 `src/app.ts` e `src/routes/index.ts`

Estender `Controllers` com `webhooks: WebhookController`. Em `buildControllers`, instanciar repository/service/controller. Em `buildApiRouter`: `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. Prefixos finais: `/api/v1/webhooks` e `/api/v1/admin/webhooks`.

### 10.5 Complementares

| Arquivo | Integração |
| --- | --- |
| `src/server.ts` | Permanece só a API. Não iniciar o worker aqui. |
| `src/worker.ts` (novo) | Entry análoga: Prisma + processor + loop + shutdown. |
| `src/config/database.ts` | Worker chama `createPrismaClient()` (não reusa o singleton da API: processos distintos). |
| `src/shared/logger/index.ts` | Mesmo logger; `base.service` pode ser `order-management-worker` no processo do worker. Redact de secret. |
| `prisma/schema.prisma` | Novos models; IDs UUID; `OrderStatus` reusado no filtro. |
| `src/modules/orders/order.status.ts` | Sem mudança de transições; `subscribedStatuses` valida contra o mesmo enum. |
| `src/middlewares/validate.middleware.ts` + `src/modules/orders/order.schemas.ts` | Copiar o estilo Zod (`z.string().uuid()`, `z.nativeEnum(OrderStatus)`). |
| `package.json` | Script `"worker": "tsx watch --env-file=.env src/worker.ts"` / start equivalente. *(arquivo de config: a implementação futura o altera; este FDD só registra o gancho combinado na call.)* |
| `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |

## 11. Dependências e compatibilidade

- Node ≥ 20, Express, Prisma 5 / MySQL, Zod, Pino, JWT — já no `package.json`. HMAC: `crypto.createHmac` da stdlib; sem lib nova obrigatória.
- Compatível com a máquina de estados atual; não altera transições.
- Clientes da API de orders não mudam contrato; ganham efeito colateral assíncrono.
- Cliente B2B precisa HTTPS, HMAC-SHA256 e dedup por `X-Event-Id`.
- Dois processos em produção (API + worker).

## 12. Critérios de aceite técnicos

1. `changeStatus` bem-sucedido com webhook inscrito no `toStatus` deixa linha `PENDING` na outbox na mesma transação; falha na insert reverte o status.
2. `changeStatus` sem endpoint inscrito **não** cria outbox.
3. Worker entrega em HTTPS com os quatro headers combinados; HTTP `http://` recusado no cadastro.
4. HMAC-SHA256 do body conferível com a secret devolvida no POST.
5. Rotação: secret nova na resposta; antiga aceita pelo cliente até 24 h (documentado); nós assinamos com a atual.
6. Após 5 falhas transitórias no backoff combinado, o evento está na DLQ.
7. Replay ADMIN reenfileira e gera log com `userId`; OPERATOR recebe 403.
8. `GET …/deliveries` devolve no máximo 100 registros com payload, sucesso/falha, response e duração.
9. Payload sem `items`; `event_type` = `order.status_changed`.
10. Códigos de erro de domínio usam prefixo `WEBHOOK_`.
11. Testes (quando forem escritos, fora deste pacote documental): unitário da transação com rollback; processor retry/DLQ; HMAC; Zod https.

## 13. Riscos e mitigação (implementação)

| Risco | Mitigação |
| --- | --- |
| Publicar fora da transação e “perder” o outbox | Função recebe `tx`; code review focado em `changeStatus` |
| Worker no mesmo processo da API | Entry `src/worker.ts` + script separado; health da API não inclui o worker |
| Secret em log de delivery | Redact Pino; nunca persistir secret em `webhook_deliveries` |
| Payload grande aborta status | Payload sem items; 64 KB; monitorar `WEBHOOK_PAYLOAD_TOO_LARGE` |
| Duplicata no cliente | `X-Event-Id` estável desde a inserção; portal (PM) |
| Single-worker vira gargalo | Métrica de pending; escala é questão em aberto, não desta entrega |
| 4xx vs 5xx não especificados na call | Tratar 4xx (exceto 408/429) como permanente; registrar no Tracker como derivação |
