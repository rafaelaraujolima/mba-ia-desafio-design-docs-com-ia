# ADR-007: Publicação do evento dentro da transação de `changeStatus`

## Status

Aceito — [09:41] Diego: "Boa, função pura recebendo o tx."

## Contexto

O ponto de integração crítico é `OrderService.changeStatus` em `src/modules/orders/order.service.ts`. Hoje, dentro de `this.prisma.$transaction(async (tx) => ...)`, ele valida a transição (`canTransition`), movimenta estoque, faz `tx.order.update` e `tx.orderStatusHistory.create`. A garantia de que "status mudou ⇒ evento existe" só vale se a inserção na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) ocorrer na mesma transação ([09:40] Bruno, [09:41] Diego).

## Decisão

- Criar a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` no módulo de webhooks, que recebe o **client transacional** (`Prisma.TransactionClient`, já usado no tipo `TxClient` do `OrderService`) ([09:41] Bruno).
- `OrderService.changeStatus` a chama dentro da transação, depois de `tx.orderStatusHistory.create`. Se a inserção na outbox falhar, a transação sofre rollback ([09:40] Bruno).
- A função **não** exige injetar um repository inteiro no `OrderService` ([09:41] Diego).
- A função consulta os webhooks ativos do customer que assinam o `toStatus` e só insere na outbox para esses ([09:34] Bruno), gravando o payload renderizado ([09:52] Larissa).
- Payload: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`; **sem `items`** ([09:43] Diego).

## Alternativas Consideradas

1. **Injetar um `WebhookRepository` no `OrderService`** — descartada: acoplamento maior; a função recebendo `tx` basta ([09:41] Bruno, [09:41] Diego).
2. **Publicar o evento depois do commit (fora da transação)** — descartada: "se ficar fora da transação, perde a garantia toda" ([09:41] Diego).
3. **Ler o histórico (`order_status_history`) no worker e derivar eventos** — plausível, não discutida; descartada aqui porque não permite filtro na inserção nem snapshot do payload, exigidos pela [ADR-001](ADR-001-outbox-no-mysql.md).

## Consequências

**Positivas**
- Consistência forte entre mudança de status e evento.
- Alteração pequena e localizada no `OrderService`; as demais regras (estoque, transições) ficam intactas.
- Função testável isoladamente com um `tx` simulado.

**Negativas / trade-offs**
- A transação de `changeStatus` fica um pouco mais pesada (consulta de webhooks assinantes + insert(s)).
- Uma falha no módulo de webhooks passa a impedir a mudança de status (decisão consciente: "não pode ter caso de status mudar e evento não sair", [09:40] Bruno).
- Cria dependência do módulo de pedidos para o módulo de webhooks (`order.service.ts` → `webhooks`).
- `OrderService.create` também grava histórico (`PENDING` inicial), mas a reunião só cobriu `changeStatus`; se a criação deve emitir evento é questão em aberto (ver RFC).
