# ADR-004: Assinatura HMAC-SHA256 com secret única por endpoint

## Status

Aceito — [09:22] Sofia: "HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."

## Contexto

Os eventos carregam dados de pedidos e saem da nossa infraestrutura para endpoints de clientes. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256** e enviar a assinatura no header `X-Signature` ([09:20] Sofia).
- **Uma secret por endpoint de webhook**, nunca uma secret global; a tabela de configuração guarda `url`, `secret`, `customer_id` e estado ativo ([09:21] Sofia, [09:21] Bruno).
- A secret é **gerada pela plataforma** e devolvida na criação ([09:31] Marcos).
- **Rotação** por endpoint da API; a secret antiga permanece válida por **24 horas** em paralelo e depois expira ([09:21] Sofia).
- Headers adicionais: `X-Timestamp` (momento do envio, para detecção de replay pelo cliente) e `X-Webhook-Id` ([09:44] Diego, [09:44] Sofia).
- URL do webhook deve ser `https`; `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).
- Revisão de segurança da geração de secret e do HMAC por Sofia, com ao menos 2 dias úteis antes do deploy ([09:46] Sofia).

## Alternativas Consideradas

1. **Secret global da plataforma** — descartada: se vaza uma, vaza tudo ([09:21] Sofia).
2. **Sem assinatura (apenas TLS)** — descartada implicitamente: TLS não prova a origem nem a integridade fim a fim para o cliente; Sofia exigiu que o cliente consiga validar origem e adulteração ([09:19] Sofia).
3. **Rotação com corte imediato da secret antiga** — descartada: o cliente precisa de tempo para migrar seus sistemas ([09:21] Sofia).

## Consequências

**Positivas**
- Padrão de mercado, com bibliotecas disponíveis em qualquer stack de cliente ([09:20] Sofia).
- Vazamento de uma secret afeta um único endpoint.
- Rotação sem janela de indisponibilidade para o cliente.

**Negativas / trade-offs**
- Como o HMAC exige a secret em claro no servidor, ela precisa ser armazenada de forma recuperável (não apenas hash). O armazenamento seguro (por exemplo, criptografia em repouso) **não foi discutido** na reunião e entra como questão em aberto no RFC.
- Durante os 24 h de grace, o servidor mantém duas secrets por endpoint (complexidade extra).
- Como assinar durante a janela de grace também não foi especificado (questão em aberto no RFC).
- A secret não pode vazar em logs; o logger Pino do projeto (`src/shared/logger/index.ts`) usa `redact` por caminhos e precisará incluir os campos de secret.
