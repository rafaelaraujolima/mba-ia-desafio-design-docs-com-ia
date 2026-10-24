# ADR-006: Reuso dos padrões existentes do projeto

## Status

Aceito — [09:30] Larissa: "Decisão: reuso máximo do que já existe."

## Contexto

O projeto segue uma arquitetura consistente: um módulo por domínio em `src/modules/<dominio>/` com controller, service, repository, routes e schemas; erros baseados em `AppError`; logger Pino; middleware de erro centralizado; ids UUID. A feature de webhooks é nova e deve parecer nativa ao código ([09:27] Bruno).

## Decisão

Implementar webhooks como **módulo igual aos demais**, reutilizando:

| Padrão | Arquivo / local no código | Uso em webhooks |
| --- | --- | --- |
| Estrutura de módulo | `src/modules/customers/*` (controller, service, repository, routes, schemas) | Novo `src/modules/webhooks/` ([09:27] Bruno) |
| Classe base de erro | `src/shared/errors/app-error.ts`, `http-errors.ts` | Erros do módulo estendem `AppError`/`NotFoundError`/`ConflictError` etc. |
| Códigos de erro | `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` | Prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa) |
| Middleware de erro | `src/middlewares/error.middleware.ts` | Já trata `AppError`, Zod e Prisma; nenhuma mudança necessária ([09:29] Bruno) |
| Logger | `src/shared/logger/index.ts` (Pino) | Nada novo é adicionado ([09:29] Bruno) |
| Validação | `src/middlewares/validate.middleware.ts` + schemas Zod | Schemas em `webhook.schemas.ts` |
| Autorização | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` | CRUD com qualquer role autenticada; replay só `ADMIN` ([09:36] Sofia) |
| Montagem de rotas | `src/routes/index.ts` e `buildControllers` em `src/app.ts` | Registrar `/webhooks` e rota admin |
| Identificadores | UUID `@default(uuid()) @db.Char(36)` em `prisma/schema.prisma` | Tabelas novas usam UUID ([09:51] Larissa) |
| Entry-points | `src/server.ts` | Novo `src/worker.ts` com o mesmo estilo de bootstrap/shutdown |

## Alternativas Consideradas

1. **Microsserviço/biblioteca independente com padrões próprios** — plausível, mas descartada pela decisão de reuso máximo e pelo tamanho do time ([09:07] Diego, [09:30] Larissa).
2. **Reaproveitar só parte dos padrões (ex.: logger próprio do worker)** — descartada: Bruno quer "não botar nada novo" ([09:29] Bruno).

## Consequências

**Positivas**
- Curva de aprendizado zero para quem já conhece o projeto; revisão mais simples.
- Erros e validações chegam ao cliente no mesmo formato `{ error: { code, message, details } }` já emitido por `error.middleware.ts`.
- Menos dependências novas.

**Negativas / trade-offs**
- O worker não passa pelo middleware HTTP; erros no worker precisam de tratamento próprio, usando o mesmo logger.
- Segue-se a convenção atual do código (por exemplo, repositories recebem `PrismaClient` no construtor), herdando suas limitações.
- Reutilização exige disciplina: novos códigos precisam seguir o prefixo `WEBHOOK_`.
