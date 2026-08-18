# ADR-005 — Worker em processo separado com polling

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Bruno (Pedidos), Marcos (PM)

## Contexto

A outbox no MySQL (ver [ADR-001](ADR-001-outbox-no-mysql.md)) precisa de um consumidor. MySQL não oferece `LISTEN/NOTIFY` como o PostgreSQL; um trigger de banco executa SQL, mas não notifica um processo Node externo de forma limpa.

O requisito de produto é latência percebida **abaixo de 10 segundos** ("tempo real" para os clientes B2B). A API hoje sobe por `src/server.ts`; não existe worker, fila nem processo paralelo.

## Decisão

- Worker em **processo Node separado**, entry-point `src/worker.ts` e script `npm run worker`.
- **Polling a cada 2 segundos**: busca os eventos pendentes mais antigos, processa, marca. Latência mínima no pior caso de espera = 2 s, dentro do orçamento de 10 s.
- Mesmo banco, mesma `DATABASE_URL`, mesma stack Prisma. **PrismaClient novo por processo** — não compartilhar instância com a API, porque cada processo Node tem o seu pool.
- Lógica de processamento no módulo: `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`).
- **Single-worker nesta fase.** Ordering implícita por `order_id` via `created_at`. Múltiplos workers em paralelo ficam para o futuro (particionar por `order_id` ou lock pessimista).

## Alternativas consideradas

- **Worker no mesmo processo da API.** Descartada: restart da API derruba o worker. ([09:11] Diego)
- **Trigger MySQL para “acordar” o worker.** Descartada: trigger não notifica processo externo; improvisar arquivo ou HTTP a partir do banco fica esquisito. Polling de 2 s atende o requisito de &lt; 10 s. ([09:09] Bruno perguntou; [09:09] Diego)

## Consequências

**Positivas**

- Ciclo de vida do worker independente do HTTP server (`src/server.ts`).
- Atende o SLA de produto (&lt; 10 s) sem broker nem NOTIFY.
- Reusa Prisma/MySQL; operação previsível para um time pequeno.

**Negativas / trade-off**

- Polling gasta queries periódicas mesmo sem eventos.
- Deploy passa a ter dois processos (`npm start` + `npm run worker`).
- Ordering e throughput ficam limitados a um worker; escala horizontal exige desenho futuro.

O trade-off aceito: 2 s de atraso e um processo a mais, em troca de não acoplar o worker à API e de não improvisar notificação via trigger MySQL.
