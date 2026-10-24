# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Produto | Order Management System (OMS) |
| Responsável de produto | Marcos (PM) |
| Status | Rascunho para revisão |
| Documentos relacionados | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [TRACKER](TRACKER.md) |

## 1. Resumo e contexto

O OMS permite criar pedidos e acompanhar seu ciclo de vida (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, ou CANCELLED). Hoje não há nenhuma forma de **avisar** sistemas externos quando o status muda. Esta feature entrega **webhooks de saída**: a plataforma envia uma chamada HTTP ao endpoint do cliente sempre que o status de um pedido dele muda para um status de seu interesse. A entrega é somente da plataforma para o cliente ([09:02] Marcos).

## 2. Problema e motivação

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente notificação em tempo real. Hoje consultam `GET /orders` periodicamente, o que deixa a integração "lenta e cara pra eles" ([09:00] Marcos). A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre ([09:00] Marcos); o prazo informado é fim de novembro ([09:45] Marcos). Para os clientes, menos de 10 segundos já é "tempo real" ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

**Público:** sistemas de integração dos clientes B2B, operados por seus times técnicos. Os clientes se cadastram pela API da plataforma, autenticados com JWT de usuários que os representam ([09:32] Marcos). Internamente: operadores e administradores da plataforma (replay de falhas) ([09:36] Sofia).

**Cenários**
1. *Acompanhar entregas:* a Atlas cadastra um webhook só para `SHIPPED` e `DELIVERED` e atualiza seu ERP assim que o pedido muda ([09:33] Marcos).
2. *Cliente fora do ar:* o endpoint do cliente cai por manutenção; a plataforma tenta novamente por até ~15 h e, se ainda falhar, guarda o evento para reprocessamento manual ([09:17] Diego, [09:18] Diego).
3. *Auditoria de integração:* o cliente consulta as últimas entregas (sucesso/falha, payload, resposta, tempo) para depurar ([09:34] Marcos).
4. *Secret exposta:* o cliente vazou a secret em um log; ele pede uma nova e migra seus sistemas dentro de 24 h ([09:21] Sofia, [09:22] Diego).
5. *Reprocessamento:* um ADMIN reenvia um evento da fila de falhas ([09:36] Sofia).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| O1 | Notificação em tempo real | Latência entre a mudança de status e o envio ao cliente (caminho feliz) | **< 10 s** ([09:02] Marcos); polling de 2 s ([09:09] Diego) |
| O2 | Não perder notificações por indisponibilidade temporária | Janela coberta pelo retry | **≈ 15 h** (5 tentativas: 1m/5m/30m/2h/12h) ([09:17] Diego) |
| O3 | Consistência status ⇔ evento | Mudanças de status sem evento correspondente (para webhooks elegíveis) | **0** ([09:40] Bruno) |
| O4 | Atender o prazo comercial | Entrega da feature | **3 sprints**, em tempo para o fim de novembro ([09:46] Larissa, [09:45] Marcos) |
| O5 | Reduzir a necessidade de polling dos clientes | Clientes migrados do polling de `GET /orders` para webhooks | 3 clientes solicitantes [meta nominal do pedido; não quantificada na reunião] |

## 5. Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por cliente.
- Filtro por status de pedido de interesse.
- Envio assinado (HMAC-SHA256) com retry, DLQ e replay administrativo.
- Rotação de secret com período de convivência de 24 h.
- Histórico de entregas por webhook.

### Fora de escopo (descartado ou adiado na reunião)
| Item | Situação | Origem |
| --- | --- | --- |
| Notificação por e-mail ao cliente quando o webhook falha | Adiado para uma próxima fase, após medir impacto | [09:37] Larissa |
| Rate limiting de envio por cliente | Adiado: observar e decidir depois | [09:39] Larissa |
| Dashboard/painel visual para o cliente | Descartado nesta fase; projeto separado do frontend | [09:40] Larissa |
| Webhooks de entrada (cliente → plataforma) | Fora: somente saída | [09:02] Marcos |
| Garantia de ordering global / múltiplos workers | Limitação conhecida; futuro | [09:13] Larissa |
| Arquivamento de eventos entregues | Fora do escopo da feature | [09:08] Diego |
| Entrega exactly-once | Descartada | [09:25] Diego |
| Itens do pedido no payload | Descartado, o cliente consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | Cadastrar webhook (`POST`) informando URL e lista de status de interesse; a secret é gerada pela plataforma e devolvida na criação | [09:31] Marcos |
| RF-02 | Associar o webhook a um cliente (`customer_id` informado na requisição, não derivado do JWT) | [09:32] Larissa |
| RF-03 | Editar (`PATCH`) um webhook | [09:33] Bruno |
| RF-04 | Remover (`DELETE`) um webhook | [09:33] Bruno |
| RF-05 | Listar (`GET`) os webhooks de um cliente | [09:33] Bruno |
| RF-06 | Filtrar eventos por status: só gerar notificação para os status que o webhook escolheu | [09:33] Marcos, [09:34] Bruno |
| RF-07 | Gerar um evento a cada mudança de status de pedido, atomicamente com a mudança | [09:40] Bruno |
| RF-08 | Enviar o evento por HTTP ao endpoint do cliente com payload JSON (id do evento, tipo, timestamp, pedido, status anterior e novo, cliente, total) | [09:43] Diego |
| RF-09 | Assinar cada envio (HMAC-SHA256) e enviar headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | [09:44] Diego, [09:44] Sofia |
| RF-10 | Reenviar automaticamente em caso de falha (5 tentativas com backoff 1m/5m/30m/2h/12h) | [09:17] Larissa |
| RF-11 | Mover para fila de falhas (DLQ) os eventos que esgotaram as tentativas | [09:18] Diego |
| RF-12 | Reprocessar manualmente um item da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), restrito a ADMIN, registrando quem executou | [09:36] Sofia, [09:36] Larissa |
| RF-13 | Rotacionar a secret do webhook via API; a antiga segue válida por 24 h | [09:21] Sofia |
| RF-14 | Consultar o histórico de entregas de um webhook (`GET /webhooks/:id/deliveries`), com sucesso/falha, payload, resposta e tempo | [09:34] Marcos |
| RF-15 | Aceitar apenas URLs `https` | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | Latência: notificação em menos de 10 s; polling a cada 2 s | [09:02] Marcos, [09:09] Diego |
| RNF-02 | Segurança: autenticidade e integridade via HMAC-SHA256; secret única por endpoint | [09:20] Sofia, [09:21] Sofia |
| RNF-03 | Segurança: TLS obrigatório (`https`) | [09:23] Sofia |
| RNF-04 | Confiabilidade: entrega at-least-once, com `X-Event-Id` para deduplicação | [09:26] Larissa |
| RNF-05 | Limite de payload: 64 KB; acima disso, erro | [09:24] Larissa |
| RNF-06 | Timeout de 10 s por chamada ao cliente | [09:42] Diego |
| RNF-07 | Isolamento: o envio não pode impactar a transação de mudança de status nem outros pedidos | [09:04] Bruno |
| RNF-08 | Ordering por pedido (`order_id`) enquanto houver um único worker; sem garantia global | [09:13] Larissa |
| RNF-09 | Operação: worker em processo separado da API | [09:11] Diego |
| RNF-10 | Consistência com o projeto: erros `WEBHOOK_*`, Pino, módulo em `src/modules/webhooks` | [09:30] Larissa |
| RNF-11 | Revisão de segurança (HMAC e geração de secret) com ao menos 2 dias úteis antes do deploy | [09:46] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off | ADR |
| --- | --- | --- |
| Outbox no MySQL | Sem infra nova e atômico, mas a tabela cresce e exige worker | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s | Simples e atende < 10 s, mas latência mínima de até 2 s e um processo a mais | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| 5 tentativas + DLQ | Cobre ~15 h, mas além disso exige intervenção manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256 por endpoint | Seguro e padrão, mas exige gestão de secrets e rotação | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-por-endpoint.md) |
| At-least-once + `X-Event-Id` | Sem perda, mas o cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Consistência e velocidade, herdando as limitações atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Evento gravado dentro da transação de status | Garantia forte, mas acopla a falha da outbox à mudança de status | [ADR-007](adrs/ADR-007-publicacao-transacional-no-order-service.md) |

## 9. Dependências

- Aplicação existente: `OrderService.changeStatus`, MySQL/Prisma, JWT e roles (`ADMIN`, `OPERATOR`).
- Cliente precisa expor endpoint `https` e implementar deduplicação por `X-Event-Id`.
- Time de segurança (Sofia) disponível para revisar antes do deploy ([09:46] Sofia).
- Portal de desenvolvedores: documentação de integração a cargo do PM ([09:26] Marcos, [09:40] Marcos).
- Equipe de infraestrutura para executar o worker como processo separado ([09:11] Diego).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Perder o prazo da Atlas e o cliente migrar para o concorrente | Média | Alto | Escopo enxuto, 3 sprints planejadas, itens adiados fora do caminho crítico |
| Cliente recebe eventos duplicados e processa em duplicidade | Média | Médio | Header `X-Event-Id` e documentação destacada no portal |
| Vazamento de secret por parte do cliente | Média | Alto | Secret por endpoint e rotação com 24 h de convivência |
| Cliente indisponível por mais de ~15 h perde a entrega automática | Baixa | Médio | DLQ persistida e replay manual por ADMIN |
| Falha na gravação do evento impede mudança de status | Baixa | Alto | Insert simples na mesma transação; testes ponta a ponta |
| Rajada de eventos sobrecarrega o cliente | Baixa | Médio | Sem rate limiting agora; observar e decidir depois |

## 11. Critérios de aceitação

- [ ] Um cliente cadastra um webhook `https`, recebe a secret e escolhe os status desejados.
- [ ] Ao mudar o status de um pedido para um status assinado, o endpoint do cliente recebe a notificação em menos de 10 s.
- [ ] Mudanças para status não assinados não geram notificação.
- [ ] A notificação traz os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e a assinatura é verificável com a secret.
- [ ] Com o endpoint do cliente fora do ar, o sistema retenta conforme 1m/5m/30m/2h/12h e, depois, o evento aparece na DLQ.
- [ ] Um ADMIN reprocessa um item da DLQ; um OPERATOR recebe acesso negado; o replay fica registrado em log.
- [ ] O cliente rotaciona a secret e a anterior continua válida por 24 h.
- [ ] O cliente lista as entregas (sucesso/falha, payload, resposta, tempo) de um webhook.
- [ ] Cadastro com `http://` é rejeitado.
- [ ] Nenhum item "fora de escopo" foi implementado.

## 12. Estratégia de testes e validação

- **Unitários:** assinatura HMAC, cálculo do backoff, filtro de eventos, validação de URL e de tamanho (Vitest, já usado em `tests/`).
- **Integração:** `changeStatus` insere na outbox e faz rollback conjunto; CRUD de webhooks; autorização do replay (ADMIN vs OPERATOR).
- **Ponta a ponta:** worker com endpoint de cliente simulado nos cenários sucesso, 5xx, timeout, duplicidade, esgotamento para DLQ e replay ([09:46] Larissa inclui testes ponta a ponta no plano).
- **Segurança:** revisão de código por Sofia antes do deploy ([09:46] Sofia).
- **Validação com clientes:** acompanhar a adoção pelos três clientes solicitantes após a entrega [derivado].
