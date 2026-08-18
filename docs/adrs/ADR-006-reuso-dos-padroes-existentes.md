# ADR-006 — Reuso dos padrões existentes do projeto

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Bruno (Pedidos), Larissa (Tech Lead), Diego (Plataforma)

## Contexto

O OMS já tem um padrão de módulo e de tratamento de erro. Inventar uma stack paralela para webhooks (logger próprio, formato de erro diferente, auth ad hoc) aumentaria o custo de manutenção e quebraria o middleware centralizado.

Padrões concretos no código hoje:

- Cada domínio vive em `src/modules/<dominio>/` com `controller`, `service`, `repository`, `routes` e `schemas` (Zod). Exemplo: `src/modules/orders/`.
- Erros de domínio estendem `AppError` em `src/shared/errors/app-error.ts`, com subclasses em `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` / `INVALID_STATUS_TRANSITION`, `InsufficientStockError` / `INSUFFICIENT_STOCK`).
- `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e erros Prisma, serializando `{ error: { code, message, details } }`.
- Auth: `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`. Role `ADMIN` já é exigida em `src/modules/users/user.routes.ts`.
- Logger Pino em `src/shared/logger/index.ts`, com redação de `password`, `token` e headers sensíveis.
- IDs UUID (`@default(uuid()) @db.Char(36)`) em `prisma/schema.prisma`.
- DI manual em `src/app.ts` (`buildControllers`) e montagem em `src/routes/index.ts` sob `/api/v1`.
- Listagens paginadas via `src/shared/http/response.ts`.

## Decisão

Reuso máximo do que já existe. Webhooks são um módulo igual aos outros:

- Pasta `src/modules/webhooks/` com controller, service, repository, routes e schemas Zod.
- Códigos de erro com prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo estilo `SCREAMING_SNAKE_CASE` dos códigos atuais.
- Sem logger novo: Pino do projeto.
- Sem mudar o error middleware: novas classes `AppError` caem no fluxo existente.
- CRUD autenticado com `authenticate`; replay de DLQ com `requireRole('ADMIN')`.
- IDs UUID, alinhados ao schema.
- Integração no `OrderService` via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` da transação atual — sem injetar o repository inteiro de webhooks no service de pedidos.
- Worker: PrismaClient próprio no processo (`src/config/database.ts` cria o client; o worker instancia o seu).

## Alternativas consideradas

- **Módulo/stack à parte** (erros sem `AppError`, logger diferente, worker sem Prisma). Descartada: duplicaria contratos HTTP de erro e quebraria o tratamento centralizado já existente. ([09:27–09:30] Bruno, Larissa, Diego)
- **Injetar `WebhookRepository` inteiro em `OrderService`.** Descartada em favor da função pura que recebe `tx`, para não acoplar o service de pedidos ao módulo de webhooks além do ponto de publicação. ([09:41] Bruno, [09:41] Diego)

## Consequências

**Positivas**

- Um único formato de erro, auth e log em toda a API.
- O time de Pedidos já conhece o layout do módulo; onboarding da feature é localizado.
- Error middleware e `requireRole` não precisam de redesign.

**Negativas / trade-off**

- `changeStatus` em `src/modules/orders/order.service.ts` ganha uma dependência de publicação (função `publishWebhookEvent`), mesmo que estreita.
- Prefixos e classes novas em `http-errors.ts` aumentam o arquivo compartilhado.
- Worker e API compartilham schema Prisma: migrações de webhook afetam o mesmo banco das orders.

O trade-off aceito: pequeno acoplamento pontual em `changeStatus` e no arquivo de erros, em troca de não criar uma segunda maneira de fazer API no repositório.
