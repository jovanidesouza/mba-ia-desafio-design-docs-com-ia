# Tracker de Rastreabilidade

Mapeia cada item relevante registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/*.md` à sua origem na transcrição (`TRANSCRICAO.md`) ou no código-fonte da aplicação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B pedem notificação em tempo real em vez de polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-01 | docs/PRD.md | Restrição | Risco de a Atlas migrar para concorrente se feature não sair até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Meta quantitativa: latência de notificação < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Decisão | Polling do worker a 2s atende a meta de <10s | TRANSCRICAO | [09:10] Marcos/Larissa |
| PRD-OBJ-03 | docs/PRD.md | Restrição | Retenção da Atlas Comercial como objetivo de negócio | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (url, secret gerada, filtro de status) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção do outbox | TRANSCRICAO | [09:33]-[09:34] Marcos/Bruno/Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de DLQ restrito a ADMIN | TRANSCRICAO | [09:18] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Validação de URL obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rejeição de payload de evento acima de 64KB | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega em condições normais | TRANSCRICAO | [09:02]-[09:10] Marcos/Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com X-Event-Id | TRANSCRICAO | [09:24]-[09:26] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Retry com backoff exponencial, 5 tentativas | TRANSCRICAO | [09:15]-[09:17] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret por endpoint, TLS obrigatório, timeout 10s | TRANSCRICAO | [09:19]-[09:24] Sofia/Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego/Larissa |
| PRD-NFR-06 | docs/PRD.md | Restrição | Evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Outbox no MySQL em vez de fila dedicada | TRANSCRICAO | [09:06]-[09:07] Diego/Larissa |
| PRD-DEC-02 | docs/PRD.md | Trade-off | At-least-once em vez de exactly-once | TRANSCRICAO | [09:24]-[09:25] Diego/Sofia |
| PRD-DEC-03 | docs/PRD.md | Trade-off | 5 tentativas (~15h) em vez de número menor de retries | TRANSCRICAO | [09:15]-[09:17] Diego/Bruno |
| PRD-DEC-04 | docs/PRD.md | Restrição | Ordenação garantida apenas por order_id em worker único | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| PRD-ESCOPO-FORA-01 | docs/PRD.md | Restrição | E-mail de fallback em falhas repetidas adiado | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| PRD-ESCOPO-FORA-02 | docs/PRD.md | Restrição | Dashboard visual fora de escopo | TRANSCRICAO | [09:39]-[09:40] Larissa/Marcos |
| PRD-ESCOPO-FORA-03 | docs/PRD.md | Restrição | Rate limiting de saída não decidido, só observação | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| PRD-ESCOPO-FORA-04 | docs/PRD.md | Restrição | Ordenação global multi-worker fora de escopo | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| PRD-ESCOPO-FORA-05 | docs/PRD.md | Restrição | Arquivamento automático após 30 dias fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-FORA-06 | docs/PRD.md | Restrição | Webhook é somente outbound, não inbound | TRANSCRICAO | [09:02]-[09:03] Marcos/Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | MySQL/Prisma existentes, sem nova infraestrutura | CODIGO | prisma/schema.prisma |
| PRD-DEP-02 | docs/PRD.md | Dependência | Middleware de autenticação JWT e requireRole existentes | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-03 | docs/PRD.md | Dependência | Hierarquia AppError e logger Pino existentes | CODIGO | src/shared/errors/http-errors.ts |
| PRD-DEP-04 | docs/PRD.md | Dependência | 2 dias úteis de revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Vazamento de secret em log do cliente (já ocorreu antes) | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Atraso na revisão de segurança compromete prazo | TRANSCRICAO | [09:46]-[09:47] Larissa/Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Worker indisponível atrasa notificações sem erro visível | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente sem deduplicação sofre efeito colateral duplicado | TRANSCRICAO | [09:25]-[09:26] Diego/Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox transacional no MySQL | TRANSCRICAO | [09:06]-[09:08] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker dedicado em polling de 2s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry e DLQ | TRANSCRICAO | [09:15]-[09:18] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256 por endpoint | TRANSCRICAO | [09:19]-[09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24]-[09:26] Diego |
| RFC-PROP-06 | docs/RFC.md | Decisão | Reuso dos padrões do projeto | TRANSCRICAO | [09:27]-[09:30] Bruno/Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Redis Streams descartado por overengineering | TRANSCRICAO | [09:06]-[09:07] Diego/Larissa |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Disparo síncrono descartado por acoplar latência de cliente | TRANSCRICAO | [09:04] Bruno/Larissa |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartado, MySQL sem LISTEN/NOTIFY | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | 3 retries descartado por ser agressivo demais | TRANSCRICAO | [09:16] Bruno/Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Exactly-once descartado por complexidade de coordenação | TRANSCRICAO | [09:24]-[09:25] Diego/Sofia |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Rate limiting de saída fica em aberto | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Escalabilidade do worker/ordenação multi-worker em aberto | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Notificação de fallback por e-mail adiada | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| RFC-IMPACT-01 | docs/RFC.md | Risco | Escrita adicional na transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Nova superfície operacional (worker) | TRANSCRICAO | [09:11] Diego |
| RFC-IMPACT-03 | docs/RFC.md | Risco | Risco de segurança em segredos | TRANSCRICAO | [09:22] Diego |
| RFC-IMPACT-04 | docs/RFC.md | Risco | Risco de prazo (3 sprints, fim de novembro) | TRANSCRICAO | [09:45]-[09:47] Larissa/Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Padrão Outbox no MySQL | TRANSCRICAO | [09:06]-[09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Retry com backoff exponencial e DLQ em tabela separada | TRANSCRICAO | [09:15]-[09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação | TRANSCRICAO | [09:19]-[09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24]-[09:26] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling 2s | TRANSCRICAO | [09:08]-[09:11] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso de AppError, Pino, error middleware, requireRole, padrão de módulos | CODIGO | src/shared/errors/http-errors.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-outbox.md | Decisão | Snapshot do payload na inserção + UUID como id | TRANSCRICAO | [09:51]-[09:52] Larissa/Diego/Bruno |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Criação do evento dentro da transação de changeStatus via publishWebhookEvent(tx,...) | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Processamento pelo worker: busca lote PENDING ordenado por createdAt | TRANSCRICAO | [09:08]-[09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Movimentação para DLQ após 5ª falha e replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-MODELO-01 | docs/FDD.md | Decisão | Chaves primárias em UUID para novas tabelas, padrão do projeto | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /customers/:customerId/webhooks | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | GET /customers/:customerId/webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | POST /webhooks/:id/secret/rotate | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Decisão | Payload sem items, campos enxutos (event_id, order_id, total_cents etc.) | TRANSCRICAO | [09:43]-[09:44] Diego/Bruno |
| FDD-PAYLOAD-02 | docs/FDD.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44]-[09:45] Diego/Sofia |
| FDD-ERRO-01 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND reutiliza NotFoundError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-02 | docs/FDD.md | Restrição | WEBHOOK_INVALID_URL por validação de HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Restrição | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64KB | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| FDD-ERRO-04 | docs/FDD.md | Restrição | WEBHOOK_DEAD_LETTER_NOT_FOUND no replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-05 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | [09:28]-[09:29] Bruno/Larissa |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego/Sofia |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Backoff exponencial 5 tentativas | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Requisito Não Funcional | DLQ como fallback terminal | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Requisito Não Funcional | Idempotência no cliente via X-Event-Id estável | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logs via Pino, incluindo redaction de secret | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Decisão | Correlação por requestId reaproveitando request-logger existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Extensão de changeStatus para publicar evento na mesma transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Decisão | Novas classes WEBHOOK_* estendem AppError/NotFoundError/UnprocessableEntityError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-03 | docs/FDD.md | Decisão | Endpoint de replay reutiliza requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | Erros WEBHOOK_* tratados automaticamente pelo middleware de erro centralizado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Decisão | src/server.ts como modelo estrutural para novo src/worker.ts | CODIGO | src/server.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | Novos models no schema Prisma seguindo padrão de uuid | CODIGO | prisma/schema.prisma |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Estrutura do módulo webhooks replica módulo customers | CODIGO | src/modules/customers/customer.service.ts |
| FDD-INTEG-08 | docs/FDD.md | Decisão | PrismaClient próprio do worker, mesmo banco, processo distinto | TRANSCRICAO | [09:29]-[09:30] Diego/Bruno |
| FDD-ACEITE-01 | docs/FDD.md | Requisito Funcional | Evento entregue em até ~2s após inserção em cenário single-worker | TRANSCRICAO | [09:09]-[09:10] Diego |
| FDD-ACEITE-02 | docs/FDD.md | Requisito Funcional | Evento falho 5x aparece em DLQ e some da fila ativa | TRANSCRICAO | [09:17]-[09:18] Diego |
| FDD-ACEITE-03 | docs/FDD.md | Requisito Funcional | Replay de DLQ retorna 403 para role insuficiente | TRANSCRICAO | [09:35]-[09:36] Sofia/Larissa |
| FDD-ACEITE-04 | docs/FDD.md | Requisito Funcional | URLs não-HTTPS rejeitadas na criação/edição | TRANSCRICAO | [09:23] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Escrita adicional degrada performance da transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-RISK-02 | docs/FDD.md | Risco | Worker trava sem alerta | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Vazamento de secret em log do cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Cliente não implementa deduplicação | TRANSCRICAO | [09:25]-[09:26] Diego/Marcos |

