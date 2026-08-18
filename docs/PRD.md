# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Autor de produto** | Marcos (PM) |
| **Parceiros técnicos** | Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Data da decisão** | Reunião técnica de ~55 min (ver `TRANSCRICAO.md`) |
| **Prazo combinado** | Três sprints, com revisão de segurança no fim; compromisso Atlas até fim de novembro |

Complementos: [RFC](RFC.md) (proposta), [FDD](FDD.md) (implementação), [ADRs](adrs/README.md) (decisões pontuais).

## 1. Resumo e contexto da feature

Clientes B2B precisam saber, sem polling, quando um pedido deles muda de status no OMS. A plataforma passa a **enviar** HTTP autenticado para um URL cadastrado pelo cliente, filtrado pelos status que ele escolheu. A feature é só **saída** (outbound): o cliente recebe; não envia eventos para nós.

O OMS já opera pedidos com máquina de estados, estoque transacional e auditoria. Não há notificação externa hoje — os integradores consultam `GET /orders` repetidamente. Esta feature preenche esse vácuo sem degradar a mudança de status (entrega assíncrona, fora do request do operador).

## 2. Problema e motivação

Três contas (Atlas Comercial, MaxDistribuição, Nova Cargo) formalizaram o pedido na semana da reunião. O polling atual deixa a integração lenta e cara para eles. A Atlas sinalizou risco de migração para concorrente se a capacidade não existir até o fim do trimestre.

“Tempo real” para esses clientes **não** é milissegundo: qualquer atraso **abaixo de 10 segundos** já atende. O valor é eliminar a espera manual e o polling contínuo.

## 3. Público-alvo e cenários de uso

**Público primário:** integradores B2B que já consomem a API do OMS (JWT de usuário operador/admin da nossa plataforma; o `customer_id` do cadastro do webhook é informado na API, não vem do token).

**Público interno:** time de Pedidos (publicação no ciclo de status), Plataforma (worker/outbox), Segurança (HMAC/rotação), operação ADMIN (replay de DLQ), PM (portal de desenvolvedor e prazo Atlas).

**Fora deste público nesta fase:** usuário final com UI (dashboard é projeto do frontend).

Cenários:

1. **Atlas só quer despacho e entrega.** Cadastra URL HTTPS, inscreve `SHIPPED` e `DELIVERED`, guarda a secret gerada por nós, verifica HMAC e deduplica por `X-Event-Id`.
2. **MaxDistribuição ficou 2 h em manutenção.** Não recebe na hora; retries cobrem a janela (~15 h no teto combinado). Se passar do teto, o evento cai em DLQ e um ADMIN reprocessa.
3. **Nova Cargo vazou secret no log dela.** Pede rotação na API; a secret antiga vale 24 h para ela migrar.
4. **Operador nosso muda pedido para `PAID`.** O cliente inscrito nesse status é notificado; o operador não espera o HTTP do cliente.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Substituir polling por push percebido como tempo real | Tempo entre persistir o novo status e a **primeira tentativa** de HTTP no URL do cliente | **&lt; 10 segundos** no caminho feliz (polling do worker = 2 s) |
| Entregar no prazo que segura a Atlas | Data de go-live da API + worker em produção | **Fim de novembro**, em **três sprints** incluindo review da Sofia (2 dias úteis) |
| Cliente consegue integrar só com API | Existência de CRUD + deliveries + documentação no portal (ação do PM) | Sem dependência de painel visual nesta fase |

Não há meta percentual de delivery (ex.: 99,9%) na reunião — não se inventa SLA. Sucesso operacional se observa depois (fila, DLQ, retries).

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de endpoints de webhook autenticados.
- Filtro por lista de status do pedido; geração de secret na criação; rotação com grace 24 h.
- Disparo outbound apenas quando o **status do pedido muda** (não no create inicial).
- Histórico das últimas 100 entregas por endpoint (sucesso/falha, payload, response, tempo).
- Replay manual de mensagens em DLQ por usuário `ADMIN`, com auditoria de quem reprocessou.
- Entrega assíncrona com retries e DLQ; cliente trata duplicata via identificador de evento.

### 5.2 Fora de escopo

Itens **explicitamente** descartados ou adiados na reunião (mínimo exigido; lista completa):

1. **E-mail** quando o webhook falha N vezes — “futuro / próxima fase”, depois de medir impacto. ([09:37] Marcos, Larissa)
2. **Dashboard visual** para o cliente ver webhooks — projeto separado do time de frontend; agora só endpoints. ([09:39–09:40] Marcos, Larissa)
3. **Rate limiting** de envio para o cliente — observar e decidir depois. ([09:38–09:39] Diego, Larissa)
4. **Arquivamento** de linhas entregues da outbox após 30 dias. ([09:08] Diego)
5. **Webhooks inbound** (cliente enviando para nós). ([09:02] Sofia, Marcos)
6. Entrega **exactly-once**. ([09:25] Diego)
7. **Redis / fila externa** e **HTTP síncrono** no fluxo de status. ([09:04–09:07])
8. **Múltiplos workers** e garantia de ordering global. ([09:12–09:13])

## 6. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| FR-01 | O cliente cadastra webhook via `POST`, informando URL e a lista de status que deseja ouvir. A secret é **gerada pela plataforma** e devolvida na criação. `customer_id` vai no body/path, não no JWT. |
| FR-02 | O cliente lista, edita (`PATCH`) e remove (`DELETE`) webhooks de um customer. |
| FR-03 | Cada endpoint escolhe quais eventos receber (subconjunto dos status do pedido). A plataforma **não enfileira** evento se nenhum webhook daquele customer está inscrito naquele status. |
| FR-04 | O cliente consulta o histórico das últimas **100** entregas daquele endpoint: sucesso/falha, payload, response, tempo de resposta. |
| FR-05 | ADMIN reprocessa item de DLQ via `POST /admin/webhooks/dead-letter/:id/replay`. A operação é **auditada** (quem fez). Role `ADMIN` obrigatória. |
| FR-06 | CRUD de configuração autenticado com JWT; qualquer role autenticada nesta fase (`ADMIN` ou `OPERATOR`). |
| FR-07 | Feature **somente outbound**. |
| FR-08 | Cliente valida origem/integridade com HMAC-SHA256 (`X-Signature`); secret **por endpoint**; rotação pela API com antiga válida **24 h**. |
| FR-09 | URL do webhook **obrigatoriamente HTTPS**; `http` recusado na validação. |
| FR-10 | Entrega **at-least-once**; o mesmo evento pode chegar duas vezes; o cliente deduplica por `X-Event-Id` (UUID gerado na criação do evento). Destacar no portal de desenvolvedor. |
| FR-11 | Disparo associado à **mudança de status** do pedido já persistida no OMS. |
| FR-12 | Após o teto de retries, o evento permanece recuperável via DLQ (replay), não some em silêncio. |

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| NFR-01 | Latência percebida &lt; 10 s no caminho feliz; intervalo de polling do worker = 2 s. |
| NFR-02 | Timeout da chamada HTTP ao cliente = 10 s; estouro conta como falha e entra em retry. |
| NFR-03 | Até 5 tentativas, backoff 1m / 5m / 30m / 2h / 12h (~15 h no teto). |
| NFR-04 | Payload JSON enxuto (status, totais, ids; **sem items**); teto **64 KB** — acima disso, **erro**, não truncate. |
| NFR-05 | Logs no Pino já usado pelo OMS; secrets não vazam em log. |
| NFR-06 | Códigos de erro de domínio com prefixo `WEBHOOK_`. |
| NFR-07 | Worker em **processo separado** da API; restart da API não derruba o worker. |
| NFR-08 | Revisão de segurança (HMAC e geração de secret) com **pelo menos dois dias úteis** da Sofia antes do deploy. |
| NFR-09 | IDs UUID, alinhados ao restante do OMS. |

## 8. Decisões e trade-offs principais

Resumo de produto (detalhe em ADRs). Não repetir o desenho do worker.

| Decisão | Trade-off aceito |
| --- | --- |
| Entrega assíncrona (outbox), não no request do operador | Status não espera o cliente; o cliente aceita até ~2 s de atraso base |
| At-least-once em vez de exactly-once | Cliente implementa dedup; nós não coordenamos two-phase com o destino |
| Secret por endpoint + rotação 24 h | Mais operação que uma chave global; vazamento não derruba todos os clientes |
| Sem e-mail e sem dashboard nesta fase | Integração só via API + portal; operação de falha é DLQ + ADMIN |
| Sem rate limit de saída no lançamento | Risco de burst no cliente; vamos **medir** antes de throttle |
| Três sprints incluindo segurança | Cabe no discurso de fim de novembro com a Atlas, com pouca folga |

## 9. Dependências

- OMS atual: auth JWT, roles `ADMIN`/`OPERATOR`, módulo de orders e transições de status, Prisma/MySQL, portal de desenvolvedor (PM documenta HMAC, `X-Event-Id`, payload).
- Cliente B2B: endpoint HTTPS público, verificação HMAC, armazenamento de `event_id` para dedup, `GET /orders/:id` se precisar de items.
- Segurança: janela de review da Sofia antes de produção.
- Operação: processo de worker além da API; alguém `ADMIN` para replay.

Não depende de Redis, provedor de e-mail, frontend novo nem ferramenta de APM nova.

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R-01 | Cliente fora do ar além da janela de retries (~15 h); evento só na DLQ | Média | Alto para aquele customer (pedido mudou e ele não soube) | DLQ persistida + replay ADMIN; e-mail ficou fora de propósito, então a operação precisa olhar DLQ |
| R-02 | Secret vazada no ambiente do cliente (já ocorreu) | Média | Alto (spoofing de webhook daquele endpoint) | Secret única por endpoint; rotação com 24 h; HTTPS |
| R-03 | Atlas não recebe a feature até fim de novembro e avalia concorrente | Média | Alto (churn) | Recorte enxuto (sem dashboard/e-mail/Redis); três sprints + buffer de review da Sofia |
| R-04 | Cliente não implementa dedup e processa evento duas vezes | Média | Médio (efeito de negócio no lado deles) | `X-Event-Id` + destaque no portal (Marcos) |
| R-05 | Burst de mudanças de status sobrecarrega o URL do cliente | Baixa/incerta (por isso está em aberto) | Médio | Observar métricas de envio; rate limit **não** nesta fase |

## 11. Critérios de aceitação

1. Dado um webhook HTTPS ativo inscrito em `SHIPPED`, quando um pedido daquele customer vai para `SHIPPED`, o URL recebe POST JSON assinado em menos de 10 s no caminho feliz.
2. Dado um webhook inscrito só em `DELIVERED`, uma transição para `PAID` **não** gera envio (nem linha inútil de fila, do ponto de vista do cliente).
3. Cadastro com `http://` é recusado; com `https://` é aceito e devolve secret uma vez.
4. Operador autenticado consegue CRUD; usuário sem JWT não.
5. OPERATOR **não** reprocessa DLQ; ADMIN sim, e a ação fica registrada em log.
6. Cliente consegue listar as últimas entregas daquele endpoint (até 100) com resultado e payload.
7. Cliente rotaciona secret e tem 24 h de overlap.
8. Após falhas repetidas até o teto combinado, o evento não desaparece: há caminho ADMIN de replay.
9. Documentação no portal alerta at-least-once e `X-Event-Id` (entrega do PM, paralela à API).
10. Sofia assina a revisão de HMAC/secret antes do deploy.

## 12. Estratégia de testes e validação

- **Produto / UAT:** um dos três clientes (preferência Atlas) cadastra URL de staging, muda um pedido de teste pelos status inscritos e confirma recebimento &lt; 10 s, HMAC válido e comportamento de duplicata (forçar retry).
- **Contrato:** suíte da API de configuração (CRUD, 400 em http, 403 no replay, deliveries ≤ 100) — detalhe técnico no FDD.
- **Resiliência:** endpoint de teste que devolve 500 e depois 200; conferir backoff e, no teto, DLQ + replay.
- **Segurança:** Sofia revisa geração/armazenamento/rotação de secret e ausência de secret em logs (2 dias úteis).
- **Não nesta fase:** teste de carga de rate limit (só observação) nem teste de UI.

Go-live condicionado a: critérios 1–8 da seção 11 em staging, review da Sofia, e portal com a nota de idempotência.
