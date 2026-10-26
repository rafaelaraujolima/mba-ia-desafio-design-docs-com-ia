# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

## Status

Aceito — [09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"; DLQ em tabela separada [09:18] Diego.

## Contexto

O endpoint do cliente pode estar indisponível, e já houve cliente com "indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego). É preciso uma política que cubra janelas razoáveis de indisponibilidade sem deixar eventos pendurados para sempre.

## Decisão

- **5 tentativas** com backoff exponencial **1 min / 5 min / 30 min / 2 h / 12 h**, cerca de 15 h entre a primeira falha e a última tentativa ([09:17] Diego, [09:17] Larissa).
- Esgotadas as tentativas, o evento é movido para a **DLQ**, a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp ([09:18] Diego).
- **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
- O endpoint de replay exige role `ADMIN`, reutilizando `requireRole` de `src/middlewares/auth.middleware.ts`, e **registra em log quem executou** o replay para auditoria ([09:36] Sofia, [09:36] Larissa).
- Timeout de 10 s por chamada conta como falha e entra no retry ([09:42] Diego).

## Alternativas Consideradas

1. **3 tentativas** — descartada: "3 é pouco", seriam 3 retries em 30 min e o evento morreria numa indisponibilidade matinal ([09:16] Diego).
2. **Retry indefinido com backoff** — descartada: evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego).
3. **Marcar `failed` na própria outbox em vez de DLQ separada** — descartada: tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).

## Consequências

**Positivas**
- Cobre indisponibilidades de várias horas (aceitável para o PM: "se um cliente meu cair por 15 horas, ele já tá com problema sério", [09:17] Marcos).
- Outbox principal permanece enxuta; a DLQ é auditável.
- Replay controlado e auditado.

**Negativas / trade-offs**
- Eventos de clientes fora do ar por mais de ~15 h exigem intervenção manual de um ADMIN.
- Mais uma tabela para manter e consultar.
- Notificação automática de falhas ao cliente (e-mail) está fora do escopo desta fase ([09:37] Larissa).
- Reentregas aumentam a chance de duplicatas (ver [ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
