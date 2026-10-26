# Da Reunião ao Documento: Design Docs de Webhooks de Pedidos

O enunciado original do desafio está em [task.md](task.md).

## Sobre o desafio

A partir da transcrição de uma reunião técnica de ~55 minutos ([TRANSCRICAO.md](TRANSCRICAO.md)) e do código de um Order Management System (Node.js, TypeScript, Express, Prisma/MySQL), o desafio é produzir o pacote de design docs de uma feature nova: **Sistema de Webhooks de Notificação de Pedidos**. A entrega é puramente documental (PRD, RFC, FDD, ADRs, Tracker e este README), sem alterar `src/`, `prisma/` ou `tests/`.

A regra central é a rastreabilidade: tudo o que está nos documentos precisa ter origem na transcrição ou no código. Isso exige separar o que foi decidido do que foi descartado ou adiado (e-mail, rate limiting, dashboard) e apontar lacunas em vez de preenchê-las com invenções.

## Ferramentas de IA utilizadas

- **Claude Code (Claude Sonnet 5.5)**, no VS Code: leitura da transcrição e do código, redação dos documentos, verificação cruzada e commits.
- **Plugin superpowers** (skill `brainstorming`): alinhamento inicial do trabalho. As demais skills não se aplicaram (TDD, por exemplo, não cabe em entrega documental).

## Workflow adotado

Seguiu-se a ordem sugerida no enunciado, com **um commit por documento** em uma branch dedicada (`docs/design-docs-webhooks`, no padrão `tipo/descricao` do histórico do repositório):

1. **Contextualização:** leitura integral de `TRANSCRICAO.md` e dos arquivos-chave do código (`order.service.ts`, `order.status.ts`, `schema.prisma`, erros, middlewares, rotas, `server.ts`).
2. **ADRs (7):** uma decisão por arquivo, cobrindo as 6 decisões principais mais a publicação transacional no `OrderService`.
3. **RFC:** consolidou a proposta, as alternativas descartadas e as questões em aberto, apontando para os ADRs.
4. **FDD:** detalhou modelo de dados, fluxos, contratos, erros e integração com o código.
5. **PRD:** escrito por último entre os grandes documentos, como consolidação de alto nível.
6. **Tracker:** varredura dos documentos prontos, uma linha por item, com validação automática de que os caminhos de código citados existem.
7. **README** e revisão final contra a checklist de aceite.

Convenção de honestidade: qualquer detalhe que a reunião não fixou é marcado **[derivado]** no FDD e listado em "Pontos a validar" (§12), em vez de aparecer como decisão.

## Prompts customizados

**1. Extração dirigida, separando decidido, descartado e adiado**

```text
Leia TRANSCRICAO.md inteira. Gere uma tabela com 4 grupos: (a) decisões fechadas,
(b) requisitos funcionais explícitos, (c) itens DESCARTADOS ou ADIADOS (cite quem
disse e por quê), (d) lacunas: coisas que a reunião NÃO definiu mas que a
implementação exigiria. Para cada linha, cite [hh:mm] Falante. Não preencha lacunas
com sugestões: apenas liste-as. Se não houver fala que sustente o item, não o inclua.
```

**2. Geração de ADR com rastreabilidade obrigatória**

```text
Escreva o ADR-00N no formato MADR (Status, Contexto, Decisão, Alternativas
Consideradas, Consequências positivas e negativas). Regras: (1) cada afirmação da
Decisão deve trazer [hh:mm] Falante; (2) alternativas só podem ser as discutidas na
reunião, ou marcadas explicitamente como "plausível, não discutida"; (3) cite arquivos
reais do repositório quando houver ponto de integração; (4) trade-offs negativos
devem ser explícitos; (5) se algo necessário não foi decidido na reunião, escreva
"não discutido" nas consequências em vez de inventar.
```

## Iterações e ajustes

Foram cerca de **3 ciclos principais** de geração, revisão e correção:

1. **Interpretação do backoff.** A fala diz "5 tentativas" e "quase 15 horas" com atrasos 1m/5m/30m/2h/12h. A soma é 14h36, coerente apenas com 1 envio inicial + 5 retries. Em vez de escolher em silêncio, a ambiguidade foi registrada no FDD (§5.3, §12) e no Tracker.
2. **Lacunas que a primeira versão tendia a preencher.** A reunião exige HMAC com a secret em claro e grace period de 24 h, mas não diz como armazenar a secret nem como assinar durante a convivência de duas secrets. Esses pontos viraram questão em aberto no RFC (Q6) e nas consequências negativas do ADR-004, em vez de decisões inventadas. O mesmo vale para a criação do pedido emitir ou não evento (Q7).
3. **Observabilidade sem fonte.** O FDD exige métricas e tracing, mas a reunião só fala de Pino e de "não botar nada novo". As métricas foram escritas como derivadas da outbox e dos logs, sem nova dependência, e marcadas como [derivado].
4. **Ajustes de forma.** Correção de referência cruzada errada no FDD (§6.2 → §6.8), blocos de código sem linguagem e listas sem linha em branco apontados pelo linter do editor.
5. **Verificação do Tracker.** Conferência automática de que todos os caminhos `CODIGO` existem (nenhum faltando) e de que ~84% das linhas têm fonte TRANSCRICAO com `[hh:mm] Falante`.

## Como navegar a entrega

| Ordem | Arquivo | Para quê |
| --- | --- | --- |
| 1 | [docs/PRD.md](docs/PRD.md) | Problema, escopo, requisitos e métricas |
| 2 | [docs/RFC.md](docs/RFC.md) | Proposta técnica, alternativas e questões em aberto |
| 3 | [docs/adrs/](docs/adrs/) | 7 ADRs (ADR-001 a ADR-007), uma decisão cada |
| 4 | [docs/FDD.md](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, integração |
| 5 | [docs/TRACKER.md](docs/TRACKER.md) | Origem de cada item na transcrição ou no código |

Os documentos de apoio são [TRANSCRICAO.md](TRANSCRICAO.md) (insumo, não alterado) e [task.md](task.md) (enunciado).
