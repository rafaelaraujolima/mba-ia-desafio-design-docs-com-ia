# Tracker de Rastreabilidade

Mapeia cada item dos documentos à sua origem. **Fonte** = `TRANSCRICAO` (localização: `[hh:mm] Falante`) ou `CODIGO` (caminho do arquivo no repositório). Itens marcados **[derivado]** nos documentos (detalhes de implementação sem fala literal) aparecem no final, ancorados na fala ou no código que os motivou.

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-A | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Inserir linha em webhook_outbox na mesma transação SQL | TRANSCRICAO | [09:06] Diego |
| ADR-001-B | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Síncrono inviável: cliente lento travaria a transação | TRANSCRICAO | [09:04] Bruno |
| ADR-001-C | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Payload snapshot já renderizado na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-D | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | ids UUID nas novas tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-001-E | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro por status na inserção na outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-001-F | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| ADR-001-G | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams descartado (infra extra) | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-A | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-B | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Entry-point src/worker.ts e `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-C | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | PrismaClient separado por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa | Trigger/notify do banco descartado (MySQL sem LISTEN/NOTIFY) | TRANSCRICAO | [09:09] Diego |
| ADR-002-E | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Ordering só por order_id e single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-F | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Timeout HTTP de 10 s | TRANSCRICAO | [09:42] Diego |
| ADR-002-G | docs/adrs/ADR-002-worker-separado-com-polling.md | Referência | Entry-point existente servindo de modelo | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-A | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-B | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-003-C | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay exige role ADMIN e loga quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-D | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | 3 tentativas descartado (pouco) | TRANSCRICAO | [09:16] Diego |
| ADR-003-E | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| ADR-003-F | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Referência | requireRole reutilizado para o replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-A | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Decisão | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-B | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Decisão | Secret única por endpoint, não global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-C | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Decisão | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-D | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Restrição | URL do webhook deve ser https (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-E | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Decisão | Headers X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| ADR-004-F | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Restrição | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-004-G | docs/adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md | Referência | Logger Pino com redact por caminhos | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-A | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | UUID gerado na entrada da outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-B | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa | Exactly-once descartado (complexidade) | TRANSCRICAO | [09:25] Diego |
| ADR-005-C | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup vai para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-D | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | PM documenta a dedup no portal de desenvolvedores | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-A | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-B | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-C | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Sem nada novo de logger; Pino e middleware de erro existentes | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência | Classe base AppError e erros HTTP | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-E | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência | Subclasses de erro com código (INSUFFICIENT_STOCK etc.) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-F | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência | Middleware de erro centralizado | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-G | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência | Padrão de módulo (exemplo: customers) | CODIGO | src/modules/customers/customer.routes.ts |
| ADR-006-H | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência | Registro de rotas e controllers | CODIGO | src/routes/index.ts |
| ADR-007 | docs/adrs/ADR-007-publicacao-transacional-no-order-service.md | Decisão | publishWebhookEvent(tx, order, from, to) dentro de changeStatus | TRANSCRICAO | [09:41] Bruno |
| ADR-007-A | docs/adrs/ADR-007-publicacao-transacional-no-order-service.md | Restrição | Falha na outbox causa rollback; status e evento nunca divergem | TRANSCRICAO | [09:40] Bruno |
| ADR-007-B | docs/adrs/ADR-007-publicacao-transacional-no-order-service.md | Decisão | Função pura recebendo tx, sem injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-007-C | docs/adrs/ADR-007-publicacao-transacional-no-order-service.md | Decisão | Payload sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D | docs/adrs/ADR-007-publicacao-transacional-no-order-service.md | Referência | changeStatus e prisma.$transaction | CODIGO | src/modules/orders/order.service.ts |

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Contexto | Clientes fazem polling em GET /orders; risco de churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | "Tempo real" = menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Somente webhooks de saída | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | docs/RFC.md | Contexto | Aplicação sem mecanismo de notificação; transação de status já pesada | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa | HTTP síncrono no OrderService | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams / fila externa | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Apenas 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa | Múltiplos workers desde já | TRANSCRICAO | [09:12] Diego |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | E-mail ao cliente em falha: próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | Escala com múltiplos workers (partição ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-06 | docs/RFC.md | Questão em aberto | Assinatura no grace period e armazenamento da secret (lacuna) | TRANSCRICAO | [09:22] Sofia |
| RFC-OQ-07 | docs/RFC.md | Questão em aberto | Evento na criação do pedido (lacuna; só changeStatus discutido) | CODIGO | src/modules/orders/order.service.ts |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo de 3 sprints + 2 dias de revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-RISK-02 | docs/RFC.md | Risco | Segundo processo a operar | TRANSCRICAO | [09:11] Diego |

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Prazo: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Clientes se cadastram pela API com JWT de usuários que os representam | TRANSCRICAO | [09:32] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Retry cobre ≈ 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero mudanças de status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Entrega em 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordering global / múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (URL, status, secret devolvida) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customer_id informado na requisição, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Evento a cada mudança de status, atômico | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Payload JSON com campos básicos do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry 5 tentativas com backoff | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | DLQ ao esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay de DLQ restrito a ADMIN, com log | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Somente URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda do cliente Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Duplicatas processadas pelo cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de eventos sobre o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Roles ADMIN/OPERATOR e JWT existentes | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal de desenvolvedores documentado pelo PM | TRANSCRICAO | [09:40] Marcos |

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox na transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker em loop de 2 s por ordem de created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Move para webhook_dead_letter ao esgotar | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Estados da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | webhook_endpoints: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | webhook_dead_letter: payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Campos de entrega: sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | Convenção de ids UUID Char(36) | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks (lista por customer) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Campos do payload de saída | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers de entrega | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Formato de erro e de paginação padrão | CODIGO | src/shared/http/response.ts |
| FDD-ERRO-01 | docs/FDD.md | Erro | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (> 64 KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-03 | docs/FDD.md | Erro | Classes de erro base reutilizadas | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | HTTP fora da transação | TRANSCRICAO | [09:04] Bruno |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Pino como logger | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | requestId (X-Request-Id) para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus estendido | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Status válidos e transições | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novos modelos e migração | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração | Registro de rotas | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Composição de dependências | CODIGO | src/app.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Bootstrap/shutdown como modelo do worker | CODIGO | src/server.ts |
| FDD-INT-07 | docs/FDD.md | Integração | PrismaClient reutilizável | CODIGO | src/config/database.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Validação de env com Zod | CODIGO | src/config/env.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Validação de requests com Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Script npm do worker | CODIGO | package.json |
| FDD-OOS-01 | docs/FDD.md | Exclusão | E-mail, rate limiting, dashboard, ordering global | TRANSCRICAO | [09:48] Larissa |

## Itens derivados (detalhe de implementação ancorado)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DER-01 | docs/FDD.md | Derivado | next_attempt_at para suportar o backoff (a reunião define o backoff, não a coluna) | TRANSCRICAO | [09:17] Diego |
| FDD-DER-02 | docs/FDD.md | Derivado | 5 tentativas lidas como 1 envio + 5 retries (soma 14h36 ≈ 15 h) | TRANSCRICAO | [09:17] Diego |
| FDD-DER-03 | docs/FDD.md | Derivado | Batch pequeno no polling (tamanho a definir) | TRANSCRICAO | [09:08] Diego |
| FDD-DER-04 | docs/FDD.md | Derivado | Replay reutiliza a linha da outbox e o event_id | TRANSCRICAO | [09:18] Diego |
| FDD-DER-05 | docs/FDD.md | Derivado | Rota POST /webhooks/:id/rotate-secret (a reunião só pede um endpoint) | TRANSCRICAO | [09:21] Sofia |
| FDD-DER-06 | docs/FDD.md | Derivado | Sucesso = HTTP 2xx; X-Signature em hex | TRANSCRICAO | [09:20] Sofia |
| FDD-DER-07 | docs/FDD.md | Derivado | Métricas e correlação sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-DER-08 | docs/FDD.md | Derivado | Regra "CRUD com qualquer role autenticada" | TRANSCRICAO | [09:37] Sofia |
| PRD-DER-01 | docs/PRD.md | Derivado | Validação pós-entrega com os 3 clientes | TRANSCRICAO | [09:47] Marcos |
