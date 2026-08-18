# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

## Contexto

A mudança de status de um pedido já ocorre dentro de uma transação SQL em `OrderService.changeStatus`: atualiza `orders`, insere em `order_status_history` e, quando aplicável, debita ou repõe `stock_quantity`. A feature de webhooks precisa notificar clientes B2B quando esse status muda, sem acoplar a entrega HTTP a essa transação.

Dois riscos foram colocados na mesa: (1) uma chamada HTTP síncrona no meio de `changeStatus` faria um cliente lento travar a transição de status de outros pedidos; (2) se o endpoint do cliente estiver fora do ar, não há como dar rollback na mudança de status já persistida. A aplicação hoje não possui fila, broker nem mecanismo de eventos — o único datastore disponível é o MySQL já usado via Prisma.

## Decisão

Adotar o padrão **Transactional Outbox no MySQL existente**.

Quando o status do pedido muda, **na mesma transação** que atualiza `orders` e `order_status_history`, inserir uma linha em `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP. Se a transação principal commitar, o evento está registrado; se der rollback, o evento some junto. Não há inconsistência possível entre status persistido e evento publicado.

A tabela de outbox terá índice em `status` (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`. O worker lê apenas pendentes em batch pequeno. Arquivamento de linhas entregues após 30 dias fica fora desta feature.

IDs da outbox seguem UUID, o mesmo padrão do restante do schema Prisma.

## Alternativas consideradas

- **HTTP síncrono dentro de `changeStatus`.** Descartada: a transação já é pesada (pedido + histórico + estoque); um cliente lento bloquearia outras transições, e uma falha HTTP não pode reverter o status. ([09:03] Larissa, [09:04] Bruno, [09:06] Diego)
- **Redis Streams (ou fila externa equivalente).** Descartada: exigiria nova infra (Redis Cluster) para um time pequeno; outbox no MySQL existente resolve o problema de atomicidade sem overengineering. ([09:07] Larissa, [09:07] Diego)

## Consequências

**Positivas**

- Atomicidade entre mudança de status e registro do evento, sem two-phase commit nem broker.
- Sem nova infraestrutura: reusa o MySQL e o Prisma já em produção.
- Rollback da transação de pedidos elimina automaticamente o evento correspondente.

**Negativas / trade-off**

- A outbox compete pelo mesmo banco das transações de pedido; volume alto de eventos aumenta escrita no MySQL.
- Entrega deixa de ser imediata: o worker precisa ler a tabela (ver [ADR-005](ADR-005-worker-separado-em-polling.md)).
- Linhas entregues acumulam até um arquivamento futuro (30 dias, fora de escopo).

O trade-off aceito: consistência e simplicidade operacional valem mais, nesta fase, do que latência de milissegundos ou um broker dedicado.
