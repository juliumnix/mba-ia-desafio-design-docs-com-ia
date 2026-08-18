# ADR-007 — Snapshot do payload na inserção da outbox

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

## Contexto

O evento da outbox pode guardar só `order_id` e montar o JSON na hora do HTTP, ou persistir o payload já renderizado. Entre a inserção e o envio (retries de minutos a 12 horas, ver [ADR-002](ADR-002-retry-backoff-e-dlq.md)) o pedido pode mudar de novo: outro status, outro total, outra observação.

A reunião fechou o formato do payload (campos básicos da order, **sem items**; detalhes extras via `GET /orders/:id`) e o limite de 64 KB com erro se ultrapassar.

## Decisão

Persistir o **payload já renderizado** no momento da inserção na outbox (snapshot). O worker envia exatamente aquele JSON. Se o pedido mudar depois, o evento continua refletindo o estado de quando aquele status mudou.

IDs da outbox e do evento são UUID, no padrão do schema.

## Alternativas consideradas

- **Guardar só `order_id` e renderizar no envio.** Descartada: o payload passaria a refletir o pedido *atual*, não o da transição, gerando casos esquisitos (ex.: evento `PAID` enviado com `to_status` já `SHIPPED`). ([09:51] Bruno perguntou; [09:52] Larissa e Diego)

## Consequências

**Positivas**

- Cada entrega é um registro fiel da transição que a originou, inclusive em retry e replay de DLQ.
- O worker não precisa reler o pedido para montar o body; reduz race com transições seguintes.

**Negativas / trade-off**

- A outbox armazena JSON duplicado em relação à tabela `orders` (custo de espaço; aceitável dado o payload enxuto, sem items).
- Correção de um bug no formato do payload **não** reescreve eventos já snapshotados; só eventos novos saem no formato novo.
- Payload acima de 64 KB falha na inserção (não se trunca), o que pode abortar a transação de `changeStatus` se o snapshot for feito dentro dela.

O trade-off aceito: fidelidade histórica do evento em troca de um pouco de armazenamento e da impossibilidade de “consertar” payloads já gravados.
