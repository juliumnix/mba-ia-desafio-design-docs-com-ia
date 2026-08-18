# ADR-002 — Retry com backoff exponencial e DLQ persistida

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)

## Contexto

Clientes B2B recebem webhooks outbound em endpoints fora da nossa infra. Indisponibilidade temporária (manutenção planejada, pico, rede) é esperada. Já houve cliente com janela de duas horas fora do ar. Se o worker desistir cedo demais, o cliente perde o evento; se retentar para sempre, a outbox fica com eventos pendurados indefinidamente.

A entrega precisa de uma política de retry com teto, evidência de falha permanente e um caminho manual de reprocessamento.

## Decisão

- **5 tentativas** por evento, com backoff **1 minuto / 5 minutos / 30 minutos / 2 horas / 12 horas** (~15 horas entre a primeira falha e a última tentativa).
- Após o teto, mover o evento para a tabela **`webhook_dead_letter`**, persistindo payload, motivo da falha e timestamp — evidência para debug e reprocessamento.
- Replay manual via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o item na outbox como pendente. Somente role `ADMIN`; a operação deve ser auditada (quem reprocessou).

Timeout de cada chamada HTTP do worker: **10 segundos**. Cliente que não responder nesse prazo conta como falha e entra na política de retry.

## Alternativas consideradas

- **3 tentativas, mais agressivo.** Descartada: cobriria ~30 minutos e mataria o evento no meio de uma manutenção de duas horas, cenário já observado. ([09:16] Bruno sugeriu; [09:16] Diego contra)
- **Retry indefinido com backoff.** Descartada: evento de cliente que sumiu ficaria pendurado para sempre. ([09:15] Diego)
- **Marcar `failed` na própria outbox, sem tabela separada.** Descartada em favor de `webhook_dead_letter`: a outbox principal permanece limpa para o worker, e a DLQ funciona como evidência isolada. ([09:17] Larissa perguntou; [09:18] Diego)

## Consequências

**Positivas**

- Cobre janelas reais de manutenção (~15 h) sem retry infinito.
- DLQ separada facilita operação, debug e replay pontual.
- Timeout de 10 s impede que um cliente lento segure o worker.

**Negativas / trade-off**

- Eventos podem chegar com atraso de até ~15 horas; o cliente precisa tratar atraso, não só duplicata.
- Replay é manual (sem auto-reprocessamento nem alerta por e-mail nesta fase).
- Cinco tentativas aumentam carga no endpoint do cliente em relação a um teto de 3.

O trade-off aceito: privilegiar entrega em janelas longas de indisponibilidade, com falha permanente explícita e reprocessamento humano, em vez de desistir cedo ou retentar para sempre.
