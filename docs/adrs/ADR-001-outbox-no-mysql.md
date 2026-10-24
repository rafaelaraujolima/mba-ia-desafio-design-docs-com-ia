# ADR-001: Padrão Outbox no MySQL para eventos de webhook

## Status

Aceito — decidido na reunião técnica ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

## Contexto

Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) querem ser notificados quando o status de um pedido muda, hoje consultam `GET /orders` por polling ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e movimenta estoque ([09:04] Bruno).

Disparar o HTTP dentro dessa transação travaria mudanças de status por causa de clientes lentos, e um cliente fora do ar forçaria rollback ([09:04] Bruno). Era preciso um mecanismo que garantisse "status mudou ⇔ evento registrado" sem infraestrutura nova, pois o time é pequeno ([09:07] Diego).

## Decisão

Adotar o padrão **Outbox no MySQL existente**:

- Na mesma transação que atualiza `orders` e `order_status_history`, inserir uma linha em `webhook_outbox` ([09:06] Diego). Se a inserção falhar, a transação inteira sofre rollback ([09:40] Bruno).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
- O payload é gravado **já renderizado** (snapshot no momento da inserção) ([09:52] Larissa, [09:52] Diego).
- O `id` é UUID, seguindo o padrão do projeto ([09:51] Larissa).
- A filtragem por status de interesse do webhook ocorre **na inserção**: se nenhum webhook do customer quer aquele status, nenhuma linha é criada ([09:34] Bruno).
- Arquivamento de linhas entregues (≈30 dias) fica fora do escopo ([09:08] Diego).

## Alternativas Consideradas

1. **Chamada HTTP síncrona no `OrderService`** — descartada: acopla latência de terceiros à transação de pedidos e não tem resposta boa para cliente offline ([09:04] Bruno, [09:06] Diego "fora de questão").
2. **Redis Streams (ou fila similar)** — descartada: exigiria subir mais infraestrutura; "Redis Cluster pra isso é overengineering" ([09:07] Larissa, [09:07] Diego).
3. **Guardar só `order_id` e renderizar o payload no envio** — descartada: o evento deixaria de refletir o estado do momento da mudança ([09:52] Larissa).

## Consequências

**Positivas**
- Atomicidade: não existe status alterado sem evento, nem evento sem status alterado.
- Sem nova infraestrutura; usa MySQL e Prisma já presentes (`prisma/schema.prisma`).
- Payload imutável: reenvios e replays enviam exatamente o que foi observado.

**Negativas / trade-offs**
- A tabela cresce com o volume de pedidos; depende de rotina de arquivamento (fora do escopo, a definir).
- Mais escrita dentro da transação crítica de `changeStatus`.
- Latência depende do polling do worker (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).
- Ordenação só é garantida por `order_id` com um único worker (ver ADR-002).

## Referências

- [ADR-007](ADR-007-publicacao-transacional-no-order-service.md) — como o `OrderService` insere na outbox.
