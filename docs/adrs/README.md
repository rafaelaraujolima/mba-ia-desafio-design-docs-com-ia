# Architectural Decision Records

Decisões arquiteturais da feature de Webhooks de Notificação de Pedidos, nomeadas `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 s |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ |
| [ADR-004](ADR-004-assinatura-hmac-sha256-por-endpoint.md) | HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [ADR-007](ADR-007-publicacao-transacional-no-order-service.md) | Publicação transacional no `OrderService` |
