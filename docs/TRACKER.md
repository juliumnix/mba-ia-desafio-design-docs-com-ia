# Tracker de rastreabilidade

Mapa de cada item dos design docs para a origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código do OMS. Se um item não puder preencher `Localização`, ele não deveria existir nos documentos.

Colunas obrigatórias: **ID** · **Documento** · **Tipo** · **Conteúdo (resumo)** · **Fonte** · **Localização**.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco de produto | Atlas pode migrar ao concorrente se não houver entrega até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Feature somente outbound (nós enviamos; cliente não envia) | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-01 | docs/PRD.md | Métrica | “Tempo real” = latência abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-MET-02 | docs/PRD.md | Métrica | Polling de 2 s atende o orçamento de 10 s | TRANSCRICAO | [09:10] Marcos |
| PRD-MET-03 | docs/PRD.md | Métrica | Prazo: três sprints, Atlas até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | POST de cadastro: url, secret gerada pela plataforma, lista de status, customer_id no body/path | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Restrição | customer_id não vem do JWT (JWT é de usuário operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | PATCH, DELETE e GET para listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtro de eventos = lista de status que o endpoint ouve | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-03b | docs/PRD.md | Decisão | Filtrar na inserção da outbox, não na hora de mandar | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries com últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| PRD-FR-05b | docs/PRD.md | Restrição | Replay exige role ADMIN e log de quem fez | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | CRUD autenticado; qualquer role JWT nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Somente webhooks saindo da plataforma | TRANSCRICAO | [09:02] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | HMAC-SHA256, secret por endpoint, rotação com grace 24 h | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | URL https obrigatória; http recusado no schema | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | At-least-once; cliente deduplica por X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-FR-10b | docs/PRD.md | Dependência | PM documenta idempotência no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Evento nasce da mudança de status do pedido | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Após teto de retry, evidência em DLQ e replay manual | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência percebida &lt; 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP do worker = 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload sem items; teto 64 KB com erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-04b | docs/PRD.md | Requisito Não Funcional | Payload JSON: event_id, event_type order.status_changed, timestamp ISO, campos básicos, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Logger Pino já existente; sem logger novo | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Dois dias úteis de review de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | IDs UUID, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail quando o webhook falha (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída (observar depois) | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivar outbox entregue após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | HTTP síncrono em changeStatus | TRANSCRICAO | [09:06] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Redis Streams | TRANSCRICAO | [09:07] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Múltiplos workers / ordering global | TRANSCRICAO | [09:13] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente offline além da janela de retry | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Secret vazada no cliente (incidente já ocorreu) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Churn da Atlas se o trimestre escorregar | TRANSCRICAO | [09:00] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Reuso de requireRole ADMIN já existente | TRANSCRICAO | [09:36] Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | GET /orders/:id para o cliente buscar items | TRANSCRICAO | [09:43] Diego |
| RFC-SUM-01 | docs/RFC.md | Proposta | Outbox MySQL + worker polling + HMAC + at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono na transação de status (trava / rollback incoerente) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams (infra extra, time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger MySQL para acordar worker (sem NOTIFY) | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Escala multi-worker / partição por order_id | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Endurecer papéis do CRUD mais à frente | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Extensão pontual de changeStatus na mesma transação | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Entry src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação SQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Chamada HTTP síncrona | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Diego |
| ADR-001-IDX | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índice em status e created_at; worker lê só pendentes | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa descartada | 3 tentativas (cobriria só ~30 min) | TRANSCRICAO | [09:16] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa descartada | Marcar failed na própria outbox | TRANSCRICAO | [09:18] Diego |
| ADR-002-TO | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Restrição | Timeout HTTP 10 s | TRANSCRICAO | [09:42] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, grace 24 h, HTTPS | TRANSCRICAO | [09:22] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-003-CFG | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Requisito Funcional | Tabela de config: url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | At-least-once com UUID em X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-004-ALT-01 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once (coordenação dos dois lados) | TRANSCRICAO | [09:25] Diego |
| ADR-004-HDR | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| ADR-004-HDR2 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Contrato | X-Webhook-Id sugerido pela segurança | TRANSCRICAO | [09:44] Sofia |
| ADR-004-ORD | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Limitação | Ordering só por order_id enquanto single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Processo separado, polling 2 s, PrismaClient próprio | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-worker-separado-em-polling.md | Alternativa descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-worker-separado-em-polling.md | Alternativa descartada | Trigger MySQL | TRANSCRICAO | [09:09] Diego |
| ADR-005-MOD | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Lógica em webhook.worker.ts / webhook.processor.ts no módulo | TRANSCRICAO | [09:28] Bruno |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo src/modules/webhooks no padrão do projeto | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | AppError + códigos tipo WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| ADR-006-MW | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Error middleware existente pega os erros sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-FN | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | publishWebhookEvent(tx, …) em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Classe AppError e campos statusCode/errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | InvalidStatusTransitionError / INSUFFICIENT_STOCK | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CODE-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | authenticate e requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-CODE-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Pino com redact de password/token | CODIGO | src/shared/logger/index.ts |
| ADR-006-CODE-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | IDs UUID @db.Char(36) | CODIGO | prisma/schema.prisma |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserir outbox dentro da $transaction de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Worker poll 2 s, batch de pendentes, marca entregue | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Retry com backoff e DLQ após teto | TRANSCRICAO | [09:15] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks cria endpoint e devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks lista por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH e DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret (path inferido do padrão REST) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Payload outbound order.status_changed sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de entrega X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:45] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | Falha se payload &gt; 64 KB (não truncar) | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Fallback = DLQ + replay ADMIN (sem e-mail) | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino; replay deve logar admin | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas de fila/retry/DLQ para observar rate limit | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por requestId já existente na API | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Gancho em OrderService.changeStatus após history.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Novos AppError caem no error middleware atual | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-03 | docs/FDD.md | Integração | requireRole('ADMIN') igual à rota de users | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Montar router em buildApiRouter / buildControllers | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Wiring de controllers em buildApp | CODIGO | src/app.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Máquina de estados reusada no filtro subscribedStatuses | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Worker não sobe em server.ts | CODIGO | src/server.ts |
| FDD-INT-08 | docs/FDD.md | Integração | PrismaClient por processo via createPrismaClient | CODIGO | src/config/database.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Listagem paginada reusa paginated() | CODIGO | src/shared/http/response.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Schemas Zod no estilo order.schemas | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-DER-01 | docs/FDD.md | Derivação | Path rotate-secret e prefixo /api/v1 inferidos do app | CODIGO | src/app.ts |
| FDD-DER-02 | docs/FDD.md | Derivação | 4xx (exceto 408/429) como falha permanente — não falado na call; política operacional do FDD | TRANSCRICAO | [09:17] Diego |
| FDD-CODE-TX | docs/FDD.md | Integração | Transação atual já atualiza order + history + stock | CODIGO | src/modules/orders/order.service.ts |
| FDD-CODE-AUTH | docs/FDD.md | Integração | Roles ADMIN e OPERATOR no JWT | CODIGO | src/middlewares/auth.middleware.ts |

## Notas de cobertura

- Itens identificáveis nos docs (FRs, NFRs, OOS, riscos, alternativas RFC, ADRs, contratos FDD, integrações) têm linha nesta tabela.
- A maioria das linhas aponta `TRANSCRICAO` no formato `[hh:mm] Nome`.
- Linhas `CODIGO` usam caminhos reais do repositório (`src/…`, `prisma/schema.prisma`).
- `FDD-DER-02` marca de propósito uma regra de retry 4xx que **não** foi literal na reunião, derivada da política de DLQ, para não misturar invenção com decisão.
- Path `POST /webhooks/:id/rotate-secret` e prefixo `/api/v1` não foram ditos na call; a origem combinada é [09:21] Sofia (existência do endpoint) + `src/app.ts` / rotas REST existentes.
