# Architectural Decision Records

Este diretório registra as decisões arquiteturais da feature **Sistema de Webhooks de Notificação de Pedidos**, no formato MADR.

Todas as decisões abaixo foram fechadas na reunião técnica documentada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md). Detalhe de implementação: [`docs/FDD.md`](../FDD.md). Proposta para revisão: [`docs/RFC.md`](../RFC.md).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Outbox transacional no MySQL | Aceita |
| [ADR-002](ADR-002-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ persistida | Aceita |
| [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint | Aceita |
| [ADR-004](ADR-004-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceita |
| [ADR-005](ADR-005-worker-separado-em-polling.md) | Worker em processo separado com polling | Aceita |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto | Aceita |
| [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do payload na inserção da outbox | Aceita |
