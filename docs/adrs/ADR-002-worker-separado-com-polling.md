# ADR-002: Worker em processo separado com polling de 2 segundos

## Status

Aceito — [09:10] Larissa: "Worker em polling, 2s", [09:11] Diego (processo separado).

## Contexto

Os eventos gravados na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) precisam ser lidos e enviados por HTTP. O requisito de negócio é que "qualquer coisa abaixo de 10 segundos" seja tempo real ([09:02] Marcos). O projeto hoje tem uma única entry-point, `src/server.ts`, que sobe a API Express e tem shutdown próprio (`SIGINT`/`SIGTERM`).

## Decisão

- O worker faz **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em **batch pequeno**, processando e marcando-os ([09:09] Diego, [09:08] Diego).
- Roda como **processo separado da API**, para que reiniciar a API não derrube o worker ([09:11] Diego).
- Nova entry-point `src/worker.ts` e script `npm run worker` ([09:11] Larissa). A lógica de processamento fica em `src/modules/webhooks/` (ex.: `webhook.worker.ts` ou `webhook.processor.ts`) ([09:28] Bruno).
- Mesmo banco e mesma `DATABASE_URL`, mas **uma instância própria de `PrismaClient`**, pois o client é por processo ([09:30] Bruno).
- **Single-worker**: a ordenação segue `created_at` da outbox, o que dá ordem por `order_id`. Sem garantia de ordering global ([09:12] Diego, [09:13] Larissa).
- Timeout do HTTP call do worker: 10 segundos; estourou, é falha e vai para retry ([09:42] Diego).

## Alternativas Consideradas

1. **Trigger do banco / notificação reativa** — descartada: MySQL não tem `LISTEN/NOTIFY`; trigger só executa SQL e não avisa processo externo ([09:09] Diego).
2. **Worker dentro do processo da API** — descartado: restart da API perderia o worker ([09:11] Diego).
3. **Múltiplos workers em paralelo** — adiado: perderia a ordenação; futuramente seria possível particionar por `order_id` ou usar lock pessimista ([09:13] Diego).

## Consequências

**Positivas**
- Latência máxima de leitura de ~2 s no pior caso, dentro da meta de 10 s ([09:10] Larissa aceita a latência).
- Simplicidade operacional: mesma stack, mesmo banco.
- Falhas da API e do worker são independentes.

**Negativas / trade-offs**
- Consultas periódicas ao banco mesmo sem eventos.
- Throughput limitado a um worker; escalar exige redesenho (partição por `order_id` ou lock pessimista).
- Ordenação só por `order_id` e apenas enquanto houver um único worker (limitação conhecida, documentada [09:13] Larissa).
- Um segundo processo para implantar e monitorar.
