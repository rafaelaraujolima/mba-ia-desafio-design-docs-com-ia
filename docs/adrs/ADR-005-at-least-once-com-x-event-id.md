# ADR-005: Entrega at-least-once com `X-Event-Id` para deduplicação

## Status

Aceito — [09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

## Contexto

Com retries ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) e polling ([ADR-002](ADR-002-worker-separado-com-polling.md)), um evento pode ser enviado mais de uma vez (por exemplo, o cliente processou, mas a resposta expirou no timeout de 10 s). É necessário escolher a garantia de entrega.

## Decisão

- Garantia **at-least-once**: o cliente pode receber o mesmo evento duas vezes e deve estar preparado ([09:24] Diego).
- Cada evento recebe um **UUID gerado na inserção na outbox**, enviado no header `X-Event-Id`, único por evento ([09:25] Diego). O cliente deduplica por esse identificador.
- O mesmo `event_id` também aparece no corpo do payload ([09:43] Diego).
- A responsabilidade de deduplicar é do cliente; o PM documentará isso de forma destacada no portal de desenvolvedores ([09:26] Marcos).
- Em reenvios e replays da DLQ o `X-Event-Id` permanece o mesmo (o evento é o mesmo; inferência do fato de o replay "recolocar na outbox", [09:18] Diego).

## Alternativas Consideradas

1. **Exactly-once** — descartada: exigiria coordenação dos dois lados e é "muito mais complexo" ([09:25] Diego).
2. **At-most-once (sem retry)** — descartada: perderia eventos em qualquer indisponibilidade do cliente, contrariando a política de retry da [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).

## Consequências

**Positivas**
- Nenhum evento é perdido por falha transitória.
- Modelo adotado por Stripe e GitHub, familiar aos integradores ([09:25] Diego).
- "Resolve 99% dos casos" ([09:25] Diego).

**Negativas / trade-offs**
- Joga a responsabilidade de idempotência para o cliente ([09:25] Sofia).
- Exige comunicação clara na documentação de integração.
- Duplicatas são esperadas e não são tratadas como erro.
