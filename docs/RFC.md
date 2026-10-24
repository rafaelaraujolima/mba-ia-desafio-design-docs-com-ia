# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | Quinta-feira da reunião técnica (data exata não registrada na transcrição) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| Documentos relacionados | [PRD](PRD.md) · [FDD](FDD.md) · [TRACKER](TRACKER.md) |

## Resumo executivo (TL;DR)

Propomos notificar clientes B2B de cada mudança de status de pedido por **webhooks HTTP de saída**. Cada mudança grava um evento numa tabela **outbox** no MySQL, **na mesma transação** de `OrderService.changeStatus`. Um **worker em processo separado** lê a outbox por polling (2 s), assina cada requisição com **HMAC-SHA256** (secret por endpoint) e a envia. Falhas têm **5 retries com backoff** e depois vão para uma **DLQ** com replay manual por ADMIN. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação pelo cliente. Tudo reaproveita os padrões do projeto. Estimativa: **3 sprints**, incluindo a revisão de segurança.

## Contexto e problema

Três clientes (Atlas Comercial, MaxDistribuição, Nova Cargo) consultam `GET /orders` repetidamente para descobrir mudanças de status, o que torna a integração lenta e cara para eles; a Atlas condicionou a permanência à entrega até o fim do trimestre ([09:00] Marcos). Para eles, menos de 10 segundos é "tempo real" ([09:02] Marcos). A aplicação não tem hoje nenhum mecanismo de notificação, evento, fila ou webhook. A mudança de status ocorre numa transação já pesada (pedido, histórico e estoque) em `src/modules/orders/order.service.ts` ([09:04] Bruno). Os webhooks são **somente de saída** ([09:02] Marcos).

## Proposta técnica

Visão geral do fluxo (detalhes de implementação no [FDD](FDD.md)):

```text
changeStatus (API) ──tx──► orders + order_status_history + webhook_outbox
                                                │
                           worker (processo separado, poll 2s)
                                                │  HMAC-SHA256 + headers
                                                ▼
                                       endpoint HTTPS do cliente
                                   falha ─► retry 1m/5m/30m/2h/12h ─► DLQ
```

Pilares, cada um registrado em um ADR:

1. **Outbox no MySQL** com payload já renderizado, filtro por status na inserção e ids UUID — [ADR-001](adrs/ADR-001-outbox-no-mysql.md).
2. **Inserção transacional** via `publishWebhookEvent(tx, ...)` chamada de dentro de `changeStatus` — [ADR-007](adrs/ADR-007-publicacao-transacional-no-order-service.md).
3. **Worker separado** (`src/worker.ts`, `npm run worker`) em polling de 2 s, single-worker — [ADR-002](adrs/ADR-002-worker-separado-com-polling.md).
4. **Retry (5 tentativas) e DLQ** em tabela separada, com replay administrativo — [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md).
5. **Segurança**: HMAC-SHA256, secret por endpoint, rotação com 24 h de grace, URLs `https` — [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md).
6. **Semântica de entrega**: at-least-once com `X-Event-Id` — [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md).
7. **Reuso dos padrões** (módulo `src/modules/webhooks`, `AppError`, Pino, códigos `WEBHOOK_*`) — [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).

Superfície de API (resumo): CRUD de configuração de webhooks, rotação de secret, histórico de entregas e replay de DLQ (admin).

## Alternativas consideradas

| Alternativa | Por que foi descartada (trade-off) |
| --- | --- |
| **Disparo HTTP síncrono no `OrderService`** | Cliente lento travaria mudanças de status de outros pedidos, e cliente offline forçaria rollback ([09:04] Bruno). Simples, mas acopla a disponibilidade do pedido à do cliente. |
| **Redis Streams / fila externa** | Entrega o que a outbox entrega, mas exige infraestrutura nova para um time pequeno: "overengineering" ([09:07] Larissa, [09:07] Diego). |
| **Trigger de banco para acordar o worker** | Mais reativo, mas o MySQL não tem `LISTEN/NOTIFY`; trigger só executa SQL, e avisar o processo externo seria "esquisito" ([09:09] Diego). Polling de 2 s já cumpre a meta de 10 s. |
| **Retry indefinido com backoff** | Nenhum evento perdido, mas eventos ficam pendurados para sempre se o cliente sumiu ([09:15] Diego). |
| **Apenas 3 tentativas** | Mais agressivo, mas uma indisponibilidade de 2 h (já ocorrida) mataria o evento ([09:16] Diego). |
| **Exactly-once** | Garantia mais forte, mas exige coordenação dos dois lados e é "muito mais complexo" ([09:25] Diego). |
| **Múltiplos workers desde já** | Mais throughput, porém perde a ordenação por `order_id` ([09:12] Diego). |

## Questões em aberto

| # | Questão | Origem |
| --- | --- | --- |
| Q1 | **Rate limiting de saída por cliente**: o que fazer se 50 pedidos mudam em um minuto? Decisão: "observar e decidir depois". | [09:38] Diego, [09:39] Larissa |
| Q2 | **Notificação por e-mail** ao cliente quando o webhook falha repetidamente: fora desta fase, a reavaliar após medir impacto. | [09:37] Marcos, [09:37] Larissa |
| Q3 | **Escala com múltiplos workers**: particionar por `order_id` ou usar lock pessimista — "problema do futuro". | [09:13] Diego |
| Q4 | **Arquivamento de linhas entregues** da outbox (~30 dias): fora do escopo da feature, sem política definida. | [09:08] Diego |
| Q5 | **Endurecimento de permissões** do CRUD (hoje qualquer role autenticada): "mais pra frente a gente pode endurecer". | [09:37] Sofia |
| Q6 | **Como assinar durante o grace period de 24 h** da rotação (uma ou duas assinaturas?) e **como armazenar a secret** com segurança: não detalhados na reunião; levar à revisão de Sofia. | Lacuna em [09:21]–[09:22] Sofia |
| Q7 | **A criação do pedido (`PENDING` inicial) emite evento?** A reunião tratou só de `changeStatus` ([09:40] Bruno). | Lacuna; ver `OrderService.create` |

## Impacto e riscos

### Impacto

- Código: novo módulo `src/modules/webhooks/`, nova entry-point `src/worker.ts`, nova migração Prisma (4 tabelas) e mudança localizada em `OrderService.changeStatus`. Nenhuma mudança no middleware de erro ([09:29] Bruno).
- Operação: um segundo processo para implantar e monitorar.
- Prazo: 3 sprints, mais pelo menos 2 dias úteis para revisão de segurança antes do deploy ([09:46] Larissa, [09:46] Sofia).


### Riscos

- A inserção na outbox dentro da transação torna `changeStatus` mais pesado e acopla sua falha à da outbox (consciente, [09:40] Bruno).
- Clientes que não deduplicam por `X-Event-Id` processarão duplicatas ([09:25] Sofia).
- Single-worker é gargalo de throughput e ponto único de falha do processamento (mitigado por ser processo supervisionável e separado da API).
- Vazamento de secret pelo cliente ([09:22] Diego): mitigado por secret por endpoint e rotação.

## Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 por endpoint](adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Publicação transacional no OrderService](adrs/ADR-007-publicacao-transacional-no-order-service.md)
