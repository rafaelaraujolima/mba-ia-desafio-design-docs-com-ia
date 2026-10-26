# FDD: Sistema de Webhooks de Notificação de Pedidos

> Especificação de implementação. Para o *porquê* de cada decisão veja os [ADRs](adrs/); para a proposta resumida, o [RFC](RFC.md); para escopo de produto, o [PRD](PRD.md).
>
> **Convenção de origem:** itens sem timestamp da reunião são marcados como **[derivado]**: são escolhas de detalhe feitas aqui para tornar a decisão da reunião implementável, e estão listadas em §12 para validação.

## 1. Contexto e motivação técnica

A aplicação (Node.js + TypeScript + Express + Prisma/MySQL) não possui eventos, filas ou webhooks. Três clientes B2B precisam saber, em menos de 10 s, quando o status de seus pedidos muda ([09:00] Marcos, [09:02] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa transação que já atualiza `orders`, `order_status_history` e estoque. Chamar o cliente dentro dessa transação é inviável ([09:04] Bruno); por isso usamos outbox + worker ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-com-polling.md)).

## 2. Objetivos técnicos

- **OT-1** Nenhuma mudança de status sem evento correspondente (e vice-versa): inserção na outbox na mesma transação ([09:40] Bruno).
- **OT-2** Latência de detecção ≤ 2 s (polling) e entrega ponta a ponta < 10 s no caminho feliz ([09:02] Marcos, [09:09] Diego).
- **OT-3** Autenticidade e integridade verificáveis pelo cliente (HMAC-SHA256) ([09:19] Sofia).
- **OT-4** Entrega at-least-once com deduplicação via `X-Event-Id` ([09:26] Larissa).
- **OT-5** Resiliência a clientes indisponíveis: 5 retries com backoff e DLQ ([09:17] Larissa).
- **OT-6** Reuso integral dos padrões do projeto ([09:30] Larissa).

## 3. Escopo e exclusões

**Dentro:** tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`; módulo `src/modules/webhooks`; worker `src/worker.ts`; CRUD de configuração; rotação de secret; histórico de entregas; replay de DLQ; extensão de `changeStatus`.

**Fora (descartado ou adiado na reunião):**

| Item | Origem |
| --- | --- |
| E-mail ao cliente quando o webhook falha | [09:37] Larissa |
| Rate limiting de envio ao cliente ("observar e decidir depois") | [09:39] Larissa |
| Dashboard visual para o cliente (projeto do time de frontend) | [09:40] Larissa |
| Webhooks de entrada (cliente → plataforma) | [09:02] Marcos |
| Múltiplos workers / ordering global | [09:13] Larissa |
| Arquivamento de eventos entregues (~30 dias) | [09:08] Diego |
| Exactly-once | [09:25] Diego |
| Itens do pedido no payload | [09:43] Diego |

## 4. Modelo de dados (Prisma)

Todas as tabelas usam `id String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), como em `prisma/schema.prisma`. Nova migração em `prisma/migrations/`.

```prisma
enum WebhookOutboxStatus { PENDING PROCESSING FAILED DELIVERED }   // [09:08] Diego

model WebhookEndpoint {            // @@map("webhook_endpoints")
  id                      String   @id @default(uuid()) @db.Char(36)
  customerId              String   @db.Char(36)          // FK customers.id   [09:21] Bruno
  url                     String   @db.VarChar(2048)     // https obrigatório [09:23] Sofia
  secret                  String   @db.VarChar(255)      // única por endpoint [09:21] Sofia
  previousSecret          String?  @db.VarChar(255)      // rotação           [09:21] Sofia
  previousSecretExpiresAt DateTime?                      // +24h              [09:21] Sofia
  events                  Json                           // lista de OrderStatus [09:31] Marcos
  active                  Boolean  @default(true)        //                   [09:21] Bruno
  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt
  @@index([customerId])
}

model WebhookOutbox {              // @@map("webhook_outbox")
  id            String   @id @default(uuid()) @db.Char(36)
  eventId       String   @unique @db.Char(36)            // X-Event-Id [09:25] Diego
  webhookId     String   @db.Char(36)                    // FK webhook_endpoints.id
  orderId       String   @db.Char(36)
  eventType     String   @db.VarChar(64)                 // "order.status_changed"
  payload       Json                                     // snapshot [09:52] Larissa
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int      @default(0)
  nextAttemptAt DateTime @default(now())                 // [derivado] suporta o backoff
  lastError     String?  @db.VarChar(500)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  @@index([status])                                      // [09:08] Diego
  @@index([createdAt])                                   // [09:08] Diego
  @@index([status, nextAttemptAt])                       // [derivado]
}

model WebhookDelivery {            // @@map("webhook_deliveries")  [09:34] Marcos
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  eventId      String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  requestBody  Json                                      // "payload"
  responseBody String?  @db.Text                         // "response" (truncado) [derivado]
  durationMs   Int                                       // "tempo de resposta"
  error        String?  @db.VarChar(500)
  createdAt    DateTime @default(now())
  @@index([webhookId, createdAt])
}

model WebhookDeadLetter {          // @@map("webhook_dead_letter") [09:18] Diego
  id            String   @id @default(uuid()) @db.Char(36)
  outboxId      String   @db.Char(36)
  webhookId     String   @db.Char(36)
  eventId       String   @db.Char(36)
  payload       Json
  failureReason String   @db.VarChar(500)
  failedAt      DateTime @default(now())
  replayedAt    DateTime?                               // [derivado] auditoria
  replayedById  String?  @db.Char(36)                   // [09:36] Sofia (auditoria)
}
```

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Chamado de `OrderService.changeStatus` dentro da `prisma.$transaction` ([ADR-007](adrs/ADR-007-publicacao-transacional-no-order-service.md)).

1. `changeStatus` valida a transição, ajusta estoque, executa `tx.order.update` e `tx.orderStatusHistory.create` (código existente).
2. Chama `publishWebhookEvent(tx, order, from, to)` ([09:41] Bruno).
3. A função carrega `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })` e **filtra** os cujo `events` contém `to` ([09:34] Bruno). Se nenhum sobrar, retorna sem inserir.
4. Para cada webhook elegível, renderiza o **snapshot** do payload (§6.8), gera `eventId = uuid()` ([09:25] Diego) e valida o tamanho serializado ≤ **64 KB**; se exceder, lança `WebhookPayloadTooLargeError` ([09:24] Larissa) e a transação inteira faz rollback.
5. Insere em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
6. Se qualquer insert falhar, o rollback desfaz também a mudança de status ([09:40] Bruno).

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts`, lógica em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno). `PrismaClient` próprio ([09:30] Bruno).

1. Loop: a cada **2 s** ([09:09] Diego) busca um **batch pequeno** (tamanho **[derivado]**, configurável) de linhas `status = PENDING AND nextAttemptAt <= now()` ordenadas por `createdAt` ASC ([09:12] Diego).
2. Marca a linha como `PROCESSING` com `UPDATE ... WHERE id = ? AND status = 'PENDING'` (claim); se 0 linhas afetadas, pula.
3. Carrega o `WebhookEndpoint`. Se inexistente/inativo, trata como falha permanente (§5.4).
4. Serializa `payload` em JSON e calcula `X-Signature` (HMAC-SHA256 do corpo com a secret do endpoint), monta os headers (§6.9) e faz `POST` com **timeout de 10 s** ([09:42] Diego).
5. Registra uma linha em `webhook_deliveries` (sucesso/falha, status HTTP, resposta, duração).
6. **Sucesso (2xx [derivado])**: `status = DELIVERED`.
7. **Falha** (não-2xx, timeout, erro de rede): segue §5.3.
8. Ordenação: um único worker; entrega em ordem de `createdAt`, o que dá ordem por `order_id` ([09:12] Diego, [09:13] Larissa). Em falha de um evento o worker segue para os demais (a ordem de reentrega **não** é garantida em retry [derivado]).
9. Recuperação: linhas `PROCESSING` com `updatedAt` antigo (worker morreu) voltam a `PENDING` no início do loop (limite **[derivado]**, sugestão 5 min).
10. Shutdown gracioso em `SIGINT`/`SIGTERM` no mesmo estilo de `src/server.ts` (termina o ciclo, `prisma.$disconnect()`).

### 5.3 Retry com backoff

Após cada falha incrementa-se `attempts` e grava-se `lastError`:

| Falha nº | Espera até a próxima tentativa |
| --- | --- |
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |
| 6ª | → DLQ |

Origem: [09:17] Diego/Larissa (1m/5m/30m/2h/12h, "quase 15 horas entre primeira falha e última tentativa", soma = 14 h 36 min). Enquanto há espera, `status = PENDING` com `nextAttemptAt = now + espera`.

> **Interpretação [derivado]:** a reunião diz "5 tentativas" e "15 horas entre a primeira falha e a última tentativa". Somar 1m+5m+30m+2h+12h = 14h36 só é coerente com **1 envio inicial + 5 retries**. Adotamos essa leitura; confirmar com Diego (§12).

### 5.4 DLQ

Quando a 6ª tentativa (5º retry) falha, ou quando a falha é permanente (webhook removido/inativo):

1. Na mesma transação: `webhook_outbox.status = FAILED` e `INSERT` em `webhook_dead_letter` com `payload`, `failureReason` e `failedAt` ([09:18] Diego).
2. Log `webhook_dead_lettered` (§9).

**Replay** (`POST /admin/webhooks/dead-letter/:id/replay`, role ADMIN): em transação, cria nova linha `PENDING` na outbox com o mesmo `payload` e **mesmo `eventId`** (ver [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) [derivado: como `event_id` é `@unique`, a linha original `FAILED` é reaproveitada: `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`], marca `replayedAt`/`replayedById` e loga quem executou ([09:36] Sofia).

### 5.5 Rotação de secret

1. `POST /webhooks/:id/rotate-secret` gera nova secret aleatória criptograficamente segura.
2. `previousSecret = secret`, `previousSecretExpiresAt = now + 24 h`, `secret = nova` ([09:21] Sofia).
3. Passadas 24 h a anterior deixa de valer: o worker ignora/limpa `previousSecret` quando `previousSecretExpiresAt < now`.
4. Como assinar durante o grace period está **em aberto** (RFC Q6); ver §12.

## 6. Contratos públicos

Convenções herdadas do projeto: `Authorization: Bearer <JWT>`, `authenticate` em todas as rotas (`src/middlewares/auth.middleware.ts`); erros no formato `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`); listas no formato `{ "data": [], "pagination": {...} }` (`src/shared/http/response.ts`); campos da API em camelCase. O `customerId` vem no body/query, **não** do JWT ([09:32] Larissa). O CRUD aceita qualquer role autenticada ([09:37] Sofia).

### 6.1 `POST /webhooks` — cadastrar

Request:

```json
{
  "customerId": "6f0f0d3e-9b8a-4c52-8a55-0e2b1f6a1c11",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a **secret é devolvida só aqui** e na rotação — [09:31] Marcos):

```json
{
  "id": "b2a6e7c4-1f0d-4f3a-9a11-3c2f8f9d0a77",
  "customerId": "6f0f0d3e-9b8a-4c52-8a55-0e2b1f6a1c11",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2c...redacted",
  "createdAt": "2026-10-24T12:00:00.000Z",
  "updatedAt": "2026-10-24T12:00:00.000Z"
}
```

Status: `201`; `400 VALIDATION_ERROR` (URL não-https, `events` vazio ou com status inválido); `401`; `404 NOT_FOUND` (customer inexistente).

### 6.2 `GET /webhooks?customerId=&page=&pageSize=` — listar ([09:33] Bruno)

Response `200`:

```json
{
  "data": [
    {
      "id": "b2a6e7c4-1f0d-4f3a-9a11-3c2f8f9d0a77",
      "customerId": "6f0f0d3e-9b8a-4c52-8a55-0e2b1f6a1c11",
      "url": "https://api.atlas.example.com/hooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-24T12:00:00.000Z",
      "updatedAt": "2026-10-24T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Nunca retorna `secret`. Status: `200`, `400`, `401`.

### 6.3 `PATCH /webhooks/:id` — editar ([09:33] Bruno)

Request (todos opcionais):

```json
{ "url": "https://api.atlas.example.com/hooks/v2", "events": ["PAID", "SHIPPED"], "active": false }
```

Response `200`: mesmo objeto de §6.2 (sem secret). Status: `200`, `400 VALIDATION_ERROR`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /webhooks/:id` — remover ([09:33] Bruno)

Response `204 No Content` (sem body). Status: `204`, `401`, `404 WEBHOOK_NOT_FOUND`. Eventos pendentes dessa configuração vão para a DLQ no processamento (§5.4).

### 6.5 `POST /webhooks/:id/rotate-secret` — rotacionar secret ([09:21] Sofia)

Request: sem body. Response `200`:

```json
{
  "id": "b2a6e7c4-1f0d-4f3a-9a11-3c2f8f9d0a77",
  "secret": "whsec_new...redacted",
  "previousSecretExpiresAt": "2026-10-25T12:00:00.000Z"
}
```

Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`. O caminho exato do endpoint é **[derivado]** (a reunião só fala em "endpoint pro cliente conseguir pedir nova secret pela API").

### 6.6 `GET /webhooks/:id/deliveries?page=&pageSize=` — histórico ([09:34] Marcos)

`pageSize` máximo 100 ("últimos 100"). Response `200`:

```json
{
  "data": [
    {
      "id": "0d7a...",
      "eventId": "c1e2d3f4-0a1b-4c2d-8e3f-a4b5c6d7e8f9",
      "attempt": 1,
      "success": false,
      "statusCode": 503,
      "durationMs": 412,
      "requestBody": { "event_type": "order.status_changed", "...": "..." },
      "responseBody": "Service Unavailable",
      "error": null,
      "createdAt": "2026-10-24T12:00:05.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Ordenado do mais recente ao mais antigo [derivado]. Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ ([09:18] Diego)

Exige `requireRole('ADMIN')` ([09:36] Larissa). Request: sem body. Response `202 Accepted` [derivado]:

```json
{ "deadLetterId": "9a8b...", "eventId": "c1e2d3f4-0a1b-4c2d-8e3f-a4b5c6d7e8f9", "status": "PENDING" }
```

Status: `202`, `401`, `403 FORBIDDEN` (role OPERATOR), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_REPLAY_NOT_ALLOWED` (já reprocessado).

### 6.8 Payload enviado ao cliente (saída)

Campos conforme [09:43] Diego, snake_case como dito na reunião; **sem `items`**:

```json
{
  "event_id": "c1e2d3f4-0a1b-4c2d-8e3f-a4b5c6d7e8f9",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-24T12:00:00.000Z",
  "order_id": "5e1a2b3c-4d5e-6f70-8192-a3b4c5d6e7f8",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f0f0d3e-9b8a-4c52-8a55-0e2b1f6a1c11",
  "total_cents": 125000
}
```

Limite: 64 KB; o cliente consulta `GET /orders/:id` para detalhes ([09:43] Diego, [09:24] Larissa).

### 6.9 Headers do request de entrega

| Header | Valor | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento | [09:25] Diego |
| `X-Signature` | HMAC-SHA256 (hex) do corpo bruto com a secret do endpoint [formato hex: derivado] | [09:20] Sofia |
| `X-Timestamp` | momento do envio | [09:44] Diego |
| `X-Webhook-Id` | id do `webhook_endpoints` | [09:44] Sofia |

Resposta esperada do cliente: **2xx** = sucesso [derivado]; qualquer outra coisa ou timeout de 10 s = falha com retry.

## 7. Matriz de erros

Padrão: classes que estendem as de `src/shared/errors/http-errors.ts`, com código `WEBHOOK_*` ([09:28] Bruno, [09:29] Larissa). Apenas `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` foram citados na reunião; os demais seguem o mesmo padrão [derivado].

| Código | HTTP | Classe base | Quando |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError`/`AppError` | Webhook inexistente |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL persistida não é `https` ao enviar (defesa em profundidade; no cadastro o Zod devolve `VALIDATION_ERROR` com `path: url`) |
| `WEBHOOK_SECRET_REQUIRED` | 500 interno (não exposto) / falha de envio | `AppError` | Endpoint sem secret ao assinar |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload > 64 KB na inserção |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` | Id de DLQ inexistente |
| `WEBHOOK_REPLAY_NOT_ALLOWED` | 409 | `ConflictError` | Entrada de DLQ já reprocessada |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (worker) | `AppError` | Sem resposta em 10 s |
| `WEBHOOK_DELIVERY_FAILED` | — (worker) | `AppError` | Resposta não-2xx ou erro de rede |
| `WEBHOOK_RETRIES_EXHAUSTED` | — (worker) | `AppError` | 5º retry falhou → DLQ |

Erros comuns continuam usando `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, tratados por `error.middleware.ts` sem alteração. Códigos "(worker)" não retornam HTTP; vão para `lastError`, `webhook_deliveries.error`, `failureReason` e logs.

## 8. Resiliência

| Aspecto | Estratégia | Origem |
| --- | --- | --- |
| Timeout HTTP | 10 s por chamada; estouro = falha com retry | [09:42] Diego |
| Retry | 5 retries, backoff 1m/5m/30m/2h/12h | [09:17] Larissa |
| Fallback final | DLQ persistida + replay manual ADMIN | [09:18] Diego |
| Isolamento | HTTP fora da transação; cliente lento não afeta API | [09:04] Bruno |
| Worker separado | restart da API não afeta o worker | [09:11] Diego |
| Duplicatas | at-least-once, dedup por `X-Event-Id` | [09:26] Larissa |
| Crash do worker | claim `PROCESSING` + reaproveitamento de linhas presas | [derivado] |
| Tamanho | rejeita payload > 64 KB | [09:24] Larissa |
| Sem fallback por e-mail | fora desta fase | [09:37] Larissa |

## 9. Observabilidade

A reunião só estabeleceu o uso do **Pino** existente ([09:29] Bruno) e não adicionar ferramentas novas ([09:29] Bruno). Métricas e tracing abaixo são **[derivados]**, sem nova dependência.

**Logs** (Pino, `src/shared/logger/index.ts`): eventos `webhook_outbox_inserted`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_dead_lettered`, `webhook_replayed` (com `userId` de quem executou — [09:36] Sofia), `worker_started`, `worker_shutdown`. Campos: `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`. As secrets (`secret`, `previousSecret`) e o header `X-Signature` devem entrar nos `redactPaths`.

**Métricas** (contadores/medidores a derivar da outbox e dos logs; sem backend de métricas definido):
- `webhook_outbox_pending` (backlog, `status = PENDING`) e idade do evento pendente mais antigo, para validar a meta < 10 s.
- `webhook_deliveries_total{result=success|failure}` e taxa de sucesso.
- `webhook_delivery_duration_ms` (p95).
- `webhook_dead_letter_total`.

**Tracing / correlação:** sem ferramenta nova. Correlação por `eventId` + `webhookId` + `orderId` nos logs e em `webhook_deliveries`; opcionalmente guardar o `requestId` (`X-Request-Id`, gerado em `src/middlewares/request-logger.middleware.ts`) na outbox para ligar a requisição que mudou o status ao envio [derivado].

## 10. Integração com o sistema existente

| Arquivo | Integração |
| --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` dentro de `this.prisma.$transaction`, após `tx.orderStatusHistory.create`. Reutiliza o tipo `TxClient` (`Prisma.TransactionClient`). O resto do método é inalterado. |
| `src/modules/orders/order.status.ts` | Fonte dos status válidos (`OrderStatus`/`canTransition`): o campo `events` do webhook é validado contra o enum `OrderStatus` do Prisma. |
| `prisma/schema.prisma` | Adição dos modelos §4 e `WebhookOutboxStatus`; relação `Customer` → `WebhookEndpoint`; ids UUID `Char(36)`. Nova pasta em `prisma/migrations/`. |
| `src/shared/errors/app-error.ts` e `http-errors.ts` | Erros `WEBHOOK_*` estendem `AppError`/`NotFoundError`/`ConflictError`/`UnprocessableEntityError`; `src/shared/errors/index.ts` não precisa mudar se o módulo exportar os seus próprios. |
| `src/middlewares/error.middleware.ts` | Sem alteração: já serializa `AppError`, `ZodError` e erros Prisma ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` no replay ([09:36] Larissa). |
| `src/middlewares/validate.middleware.ts` | `validate({ body, params, query })` com schemas Zod do módulo (inclui regra `https` — [09:23] Sofia). |
| `src/routes/index.ts` | Registrar `router.use('/webhooks', ...)` e a rota `/admin/webhooks/...`; ampliar o tipo `Controllers`. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e injeta o que for preciso no `OrderService`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts` (bootstrap, logs `*_started`, shutdown por sinais). Sem mudança. |
| `src/config/database.ts` | `createPrismaClient()` reutilizado pelo worker, que cria sua própria instância ([09:30] Bruno). |
| `src/config/env.ts` | Novas variáveis opcionais do worker (intervalo, batch) com defaults, validadas por Zod [derivado]. |
| `src/shared/logger/index.ts` | Mesmo logger Pino; incluir campos de secret em `redactPaths`. |
| `src/shared/http/response.ts` | `paginated()` nas listagens. |
| `package.json` | Novo script `"worker"` (ex.: `tsx watch --env-file=.env src/worker.ts`, mesmo estilo do script `dev`) ([09:11] Larissa). |
| `tests/` (`tests/helpers/factories.ts`, `tests/orders.test.ts`) | Novos testes reutilizando factories e padrão Vitest. |

Novos arquivos: `src/worker.ts` e `src/modules/webhooks/{webhook.controller,webhook.service,webhook.repository,webhook.routes,webhook.schemas,webhook.processor,webhook.events,webhook.errors}.ts` (estrutura de módulo de [09:27] Bruno; `webhook.events.ts` com `publishWebhookEvent` e `webhook.errors.ts` são **[derivados]**).

## 11. Dependências e compatibilidade

- Sem novas dependências obrigatórias: `crypto` (nativo do Node ≥ 20, `engines` em `package.json`) para HMAC/random; `fetch`/`undici` nativo para HTTP; `uuid`, `zod`, `pino`, `@prisma/client` já presentes.
- MySQL: a migração só cria tabelas novas; `changeStatus` continua compatível com chamadores atuais (`PATCH /orders/:id/status` não muda de contrato).
- Compatibilidade retroativa: pedidos de clientes sem webhook cadastrado não geram linhas na outbox.
- Operação: o worker usa a mesma `DATABASE_URL`.

## 12. Pontos a validar [derivados]

1. 5 tentativas = 1 envio + 5 retries (§5.3).
2. Reuso do `eventId` no replay (§5.4).
3. Forma de assinar durante o grace period e armazenamento seguro da secret (RFC Q6).
4. Emissão de evento na criação do pedido (RFC Q7).
5. Critério de sucesso 2xx; formato hex do `X-Signature`; caminho de `rotate-secret`; status `202` do replay.
6. Tamanho do batch e limite de recuperação de `PROCESSING`.

## 13. Critérios de aceite técnicos

- [ ] Mudar o status de um pedido cujo customer tem webhook assinando o novo status cria exatamente 1 linha em `webhook_outbox` por webhook elegível, na mesma transação.
- [ ] Forçar falha no insert da outbox faz rollback de `orders`, `order_status_history` e estoque.
- [ ] Webhook que não assina o status não gera linha.
- [ ] O worker entrega em < 10 s no caminho feliz e envia os 5 headers de §6.9.
- [ ] A assinatura recalculada pelo cliente com a secret coincide com `X-Signature`.
- [ ] Resposta 5xx/timeout (10 s) agenda retry conforme §5.3; após esgotar, existe linha em `webhook_dead_letter`.
- [ ] Replay por ADMIN recoloca o evento como `PENDING`; OPERATOR recebe `403`; o log registra o usuário.
- [ ] Cadastro com `http://` é rejeitado com `400`.
- [ ] Payload > 64 KB é rejeitado.
- [ ] Após rotação, a secret antiga deixa de valer após 24 h.
- [ ] `secret` nunca aparece em listagens nem em logs.
- [ ] Reiniciar a API não interrompe o worker.
- [ ] `npm run lint` e `npm test` passam.

## 14. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Falha na outbox bloqueia mudança de status | Baixa | Alto | Decisão consciente ([09:40] Bruno); inserts simples, índices, testes de transação |
| Crescimento da outbox | Média | Médio | Índices em `status`/`created_at` ([09:08] Diego); arquivamento em fase futura |
| Worker único cai ou atrasa | Média | Alto | Processo supervisionado, recuperação de `PROCESSING`, métrica de backlog |
| Vazamento de secret | Média | Alto | Secret por endpoint, rotação 24 h, `redact` em logs ([09:22] Diego) |
| Clientes não deduplicam | Média | Médio | Documentação destacada no portal ([09:26] Marcos) |
| Rajada de eventos sobrecarrega cliente | Baixa | Médio | Sem rate limiting agora; observar ([09:39] Larissa) |
| Quebra de ordenação ao escalar workers | Baixa | Médio | Limitação documentada ([09:13] Larissa) |
