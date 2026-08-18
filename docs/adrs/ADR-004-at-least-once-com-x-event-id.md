# ADR-004 — Entrega at-least-once com `X-Event-Id`

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM)

## Contexto

O worker pode reenviar o mesmo evento: timeout depois que o cliente já processou, retry após falha ambígua, replay de DLQ. Garantir exactly-once exigiria coordenação dos dois lados (nós e o cliente) e aumentaria muito a complexidade.

O cliente precisa de um identificador estável para deduplicar. IDs no projeto já são UUID (`@default(uuid())` no Prisma).

## Decisão

- Garantia de entrega **at-least-once**. O cliente **pode receber o mesmo evento duas vezes** e deve estar preparado.
- Cada evento recebe um **UUID gerado na inserção da outbox**, enviado no header **`X-Event-Id`**.
- Deduplicação é responsabilidade do cliente, usando esse `event_id`. Documentar de forma destacada no portal de desenvolvedor (ação do PM).
- Headers adicionais do envio: `X-Signature`, `X-Timestamp` (detecção opcional de replay attack pelo cliente), `X-Webhook-Id` (qual cadastro originou o envio), `Content-Type: application/json`.

Não há garantia de ordering **global**. Com single-worker, a ordem é a de `created_at` da outbox, implícita por `order_id`. Escalar workers em paralelo no futuro perde essa garantia (ver questões em aberto no RFC).

## Alternativas consideradas

- **Exactly-once.** Descartada: exigiria protocolo de coordenação entre plataforma e cliente; at-least-once com `event_id` é o padrão de mercado (Stripe, GitHub) e cobre a maior parte dos casos. ([09:24–09:25] Diego, [09:25] Sofia)

## Consequências

**Positivas**

- Implementação alinhada a retries/DLQ sem transação distribuída com o cliente.
- `X-Event-Id` estável permite dedup simples no lado do receptor.
- `X-Timestamp` e `X-Webhook-Id` dão contexto extra sem mudar o contrato de idempotência.

**Negativas / trade-off**

- A responsabilidade de idempotência fica no cliente; receptor mal implementado pode processar duplicatas (ex.: baixar estoque duas vezes).
- Documentação e onboarding precisam ser explícitos — o PM assumiu o portal.

O trade-off aceito: simplicidade operacional e padrão de mercado em troca de exigir deduplicação no cliente, em vez de um protocolo exactly-once.
