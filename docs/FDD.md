# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

> Documento de implementação. Para contexto de produto, ver [PRD](./PRD.md); para a proposta de arquitetura e alternativas, ver [RFC](./RFC.md); para o racional de cada decisão isolada, ver os ADRs em [`docs/adrs/`](./adrs/).

## Contexto e motivação técnica

O OMS atual não possui nenhum mecanismo de eventos, filas ou notificação externa. Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling em `GET /orders` para detectar mudanças de status, gerando custo de integração e latência percebida alta ([09:00] Marcos). A mudança de status já acontece de forma transacional em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que atualiza `orders`, grava `order_status_history` e ajusta `stock_quantity`. Este documento especifica como estender esse fluxo para publicar eventos de webhook de forma consistente, e como um worker dedicado processa e entrega esses eventos com resiliência e segurança.

## Objetivos técnicos

- Garantir que **nenhuma mudança de status ocorra sem o evento correspondente ser registrado**, e vice-versa (atomicidade via outbox, [ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md)).
- Entregar eventos aos clientes em **latência média abaixo de 10 segundos**, via worker em polling de 2s ([ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md)).
- Garantir **resiliência a indisponibilidade temporária** de clientes via retry com backoff exponencial e DLQ ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)).
- Garantir **autenticidade e integridade** dos payloads entregues via HMAC-SHA256 por endpoint ([ADR-003](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
- Garantir **semântica at-least-once** com suporte à deduplicação do cliente via `X-Event-Id` ([ADR-004](./adrs/ADR-004-at-least-once-event-id.md)).
- **Reaproveitar integralmente** os padrões de código já estabelecidos (módulos, erros, logger, autorização) ([ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md)).

## Escopo e exclusões

**Dentro do escopo:**
- CRUD de configuração de webhook por cliente (URL, secret, filtro de eventos por status).
- Publicação de eventos de mudança de status na outbox, dentro da transação de `changeStatus`.
- Worker de entrega com retry, backoff e DLQ.
- Consulta de histórico de entregas por webhook.
- Rotação de secret com grace period.
- Endpoint administrativo de reprocessamento manual de itens em DLQ.

**Fora do escopo (ver também seção "Fora de escopo" do [PRD](./PRD.md)):**
- Notificação de fallback por e-mail em caso de falhas repetidas (adiado — [09:37]-[09:38]).
- Dashboard visual para o cliente acompanhar webhooks (fora de escopo — [09:39]-[09:40]).
- Rate limiting de envio de saída (apenas observação em produção — [09:38]-[09:39]).
- Suporte a múltiplos workers em paralelo com ordenação global garantida (limitação conhecida — [09:12]-[09:14]).
- Arquivamento automático de eventos entregues após 30 dias ([09:08] Diego).

## Modelo de dados (novo)

Novas tabelas em `prisma/schema.prisma`, seguindo o padrão de id já usado no projeto (`@id @default(uuid()) @db.Char(36)`, ver `User`, `Customer`):

- **`WebhookEndpoint`**: `id`, `customerId`, `url`, `secret` (ativa), `previousSecret` (nullable, para grace period), `secretRotatedAt` (nullable), `eventTypes` (lista de status subscritos), `active` (boolean), `createdAt`, `updatedAt`.
- **`WebhookOutboxEvent`**: `id` (UUID = `event_id`), `webhookEndpointId`, `orderId`, `eventType`, `payload` (JSON, snapshot — [ADR-007](./adrs/ADR-007-snapshot-payload-outbox.md)), `status` (`PENDING` | `PROCESSING` | `FAILED` | `DELIVERED`), `attempts`, `nextAttemptAt`, `createdAt`. Índices em `status` e `createdAt` ([09:08] Diego).
- **`WebhookDelivery`**: `id`, `webhookOutboxEventId`, `attemptNumber`, `httpStatus` (nullable), `responseBody` (truncado), `durationMs`, `success` (boolean), `createdAt`. Base para `GET /webhooks/:id/deliveries`.
- **`WebhookDeadLetter`**: `id`, `webhookOutboxEventId`, `payload`, `failureReason`, `createdAt`. Base para o fluxo de DLQ ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)).

## Fluxos detalhados

### 1. Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` executa a transição de status normalmente (validação via `canTransition`, débito/reposição de estoque, update de `orders`, insert em `order_status_history`), tudo dentro da transação Prisma (`tx`) já existente.
2. Antes do commit, o service chama uma nova função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, proposta por Bruno e validada por Diego como "função pura recebendo o tx" ([09:41] Bruno/Diego), sem precisar injetar um repository completo no `OrderService`.
3. `publishWebhookEvent` busca, dentro da mesma `tx`, os `WebhookEndpoint` ativos do customer do pedido cujo `eventTypes` inclui o `toStatus` (filtro aplicado **na inserção**, não no envio — [09:33]-[09:34] Bruno/Diego, economiza linhas na tabela).
4. Para cada endpoint elegível, monta o payload (ver "Contratos públicos") e insere uma linha em `WebhookOutboxEvent` com `status = PENDING`, `payload` já renderizado (snapshot) e `id` = novo UUID.
5. Se qualquer etapa de inserção falhar, a transação inteira sofre rollback — a mudança de status nunca é persistida sem o evento correspondente, e vice-versa ([09:40]-[09:41] Bruno/Diego).
6. A transação commita normalmente; o pedido retorna ao controller como hoje.

### 2. Processamento pelo worker

1. `src/worker.ts` inicia um loop com `setInterval`/loop assíncrono de **2 segundos** ([09:09] Diego).
2. A cada ciclo, busca um lote de `WebhookOutboxEvent` com `status = PENDING` e `nextAttemptAt <= now()`, ordenado por `createdAt` ascendente (garante ordenação por `order_id` em cenário single-worker — [ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md)).
3. Marca o lote como `PROCESSING` (evita disputa caso o worker seja escalado futuramente).
4. Para cada evento, monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e executa um `POST` HTTP para a `url` do endpoint, com **timeout de 10 segundos** ([09:42] Diego/Sofia).
5. Registra uma linha em `WebhookDelivery` com o resultado (status HTTP, corpo de resposta truncado, duração, sucesso/falha).
6. Em caso de sucesso (2xx): marca o evento como `DELIVERED`.
7. Em caso de falha (timeout, erro de rede, status não-2xx): segue para o fluxo de retry.

### 3. Retry com backoff exponencial

1. Em caso de falha, incrementa `attempts` e calcula `nextAttemptAt` conforme a tabela de backoff ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)):

   | Tentativa | Intervalo desde a falha anterior |
   |---|---|
   | 1ª retry | 1 minuto |
   | 2ª retry | 5 minutos |
   | 3ª retry | 30 minutos |
   | 4ª retry | 2 horas |
   | 5ª retry | 12 horas |

2. Evento volta para `status = PENDING` com `nextAttemptAt` no futuro; o worker só o pega novamente quando esse horário chegar.
3. Após a 5ª tentativa falhar, o evento segue para o fluxo de DLQ.

### 4. Dead Letter Queue (DLQ)

1. Ao esgotar as 5 tentativas, o worker cria uma linha em `WebhookDeadLetter` com o payload original e o motivo da última falha, e marca o `WebhookOutboxEvent` como `FAILED`.
2. Um administrador pode consultar os itens em DLQ e disparar reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay`.
3. O replay recria (ou reativa) o `WebhookOutboxEvent` correspondente com `status = PENDING`, `attempts = 0`, para ser pego novamente pelo worker no próximo ciclo.
4. O endpoint de replay registra em log (Pino) o `userId` do administrador que executou a ação, para auditoria ([09:36] Sofia).

## Contratos públicos

Todos os endpoints de configuração/consulta usam autenticação JWT padrão (`authenticate` middleware). O `customerId` é informado explicitamente no path ou no body — **não é extraído do JWT**, pois o JWT representa o usuário operador, não o cliente ([09:32] Bruno/Larissa).

### `POST /customers/:customerId/webhooks`

Cria um novo endpoint de webhook para o customer.

**Request:**
```json
{
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "eventTypes": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`:**
```json
{
  "id": "6f1a4b2e-0e2a-4e36-9b6a-1a2b3c4d5e6f",
  "customerId": "f3c1...",
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "secret": "8f2e1c6a9b3d4f5e6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f",
  "eventTypes": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-10-01T12:00:00.000Z"
}
```
A `secret` só é retornada **na criação** ([09:31] Marcos). Erros possíveis: `WEBHOOK_INVALID_URL` (422, URL não-HTTPS), `WEBHOOK_VALIDATION_ERROR` (400), `CUSTOMER_NOT_FOUND` (404, reutilizando `NotFoundError`).

### `PATCH /webhooks/:id`

Edita `url`, `eventTypes` ou `active` de um webhook existente ([09:33] Bruno).

**Request:**
```json
{ "eventTypes": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

**Response `200 OK`:**
```json
{
  "id": "6f1a4b2e-0e2a-4e36-9b6a-1a2b3c4d5e6f",
  "customerId": "f3c1...",
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "eventTypes": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-10-01T13:00:00.000Z"
}
```
O campo `secret` nunca é retornado fora da criação. Erros: `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (422).

### `DELETE /webhooks/:id`

Remove um webhook cadastrado ([09:33] Bruno). Não possui corpo de requisição.

**Response `204 No Content`** (sem corpo de resposta). Erros: `WEBHOOK_NOT_FOUND` (404).

### `GET /customers/:customerId/webhooks`

Lista os webhooks cadastrados de um customer ([09:33] Bruno).

**Response `200 OK`:**
```json
{
  "data": [
    { "id": "6f1a4b2e-...", "url": "https://atlas-comercial.example.com/webhooks/orders", "eventTypes": ["SHIPPED", "DELIVERED"], "active": true }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```
Segue o padrão de paginação já existente em `src/shared/http/response.ts` (`paginated`).

### `GET /webhooks/:id/deliveries`

Histórico dos últimos 100 envios de um webhook, com payload, resposta e tempo de resposta ([09:34] Marcos).

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "d1e2...",
      "eventId": "6f1a4b2e-...",
      "attemptNumber": 1,
      "httpStatus": 200,
      "success": true,
      "durationMs": 184,
      "createdAt": "2026-10-01T12:00:02.000Z"
    },
    {
      "id": "d1e3...",
      "eventId": "6f1a4b2e-...",
      "attemptNumber": 1,
      "httpStatus": 503,
      "success": false,
      "durationMs": 10000,
      "createdAt": "2026-10-01T11:00:02.000Z"
    }
  ],
  "page": 1,
  "pageSize": 100,
  "total": 2
}
```
Erros: `WEBHOOK_NOT_FOUND` (404).

### `POST /webhooks/:id/secret/rotate`

Gera uma nova secret para o endpoint; a secret anterior permanece válida por 24h ([ADR-003](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)). Não possui corpo de requisição.

**Response `200 OK`:**
```json
{
  "id": "6f1a4b2e-...",
  "secret": "1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b",
  "previousSecretValidUntil": "2026-10-02T12:00:00.000Z"
}
```
Erros: `WEBHOOK_NOT_FOUND` (404).

### `POST /admin/webhooks/dead-letter/:id/replay`

Reprocessa manualmente um item em DLQ. Exige role `ADMIN` via `requireRole('ADMIN')` ([09:35]-[09:36] Sofia/Larissa). Não possui corpo de requisição.

**Response `202 Accepted`:**
```json
{ "id": "a1b2c3...", "status": "PENDING", "requeuedAt": "2026-10-01T12:00:00.000Z" }
```
Erros: `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404), `FORBIDDEN` (403, role insuficiente, reutilizando `ForbiddenError`).

### Payload de evento enviado ao cliente

```json
{
  "event_id": "6f1a4b2e-0e2a-4e36-9b6a-1a2b3c4d5e6f",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-01T12:00:00.000Z",
  "order_id": "9c8b7a6f-...",
  "order_number": "OM-2026-00123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "f3c1...",
  "total_cents": 45990
}
```
Itens do pedido **não** são incluídos, para manter o payload enxuto; o cliente consulta `GET /orders/:id` se precisar de detalhes ([09:43]-[09:44] Diego/Bruno).

**Headers enviados:** `X-Event-Id` (UUID do evento), `X-Signature` (HMAC-SHA256 do body com a secret do endpoint), `X-Timestamp` (ISO 8601 do envio), `X-Webhook-Id` (id do `WebhookEndpoint`, para customers com múltiplos cadastros — [09:44]-[09:45] Sofia/Diego), `Content-Type: application/json`.

## Matriz de erros (`WEBHOOK_*`)

| Código | Status HTTP | Cenário | Classe base reutilizada |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook não encontrado por `id` | `NotFoundError` |
| `WEBHOOK_INVALID_URL` | 422 | URL cadastrada não é HTTPS | `UnprocessableEntityError` |
| `WEBHOOK_VALIDATION_ERROR` | 400 | Payload de criação/edição inválido (schema Zod) | `ValidationError` |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Operação que exige secret ativa sem secret configurada | `UnprocessableEntityError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento ultrapassa 64KB ([09:23]-[09:24] Sofia/Diego) | `UnprocessableEntityError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ não encontrado para replay | `NotFoundError` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno, não HTTP) | Timeout de 10s na chamada ao cliente; não gera resposta HTTP ao chamador da API, é registrado em `WebhookDelivery` e dispara retry | — |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | Tentativa de operar sobre webhook com `active = false` | `ConflictError` |

Todas as classes estendem `AppError` (`src/shared/errors/http-errors.ts`), seguindo exatamente o padrão de `InsufficientStockError`/`InvalidStatusTransitionError` ([09:28] Bruno). Nenhuma alteração é necessária em `src/middlewares/error.middleware.ts` — ele já trata qualquer subclasse de `AppError` genericamente.

## Estratégias de resiliência

- **Timeout de entrega:** 10 segundos por tentativa HTTP ([09:42] Diego). Timeout é tratado como falha e segue para retry.
- **Retry com backoff exponencial:** 5 tentativas, intervalos 1m/5m/30m/2h/12h ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)).
- **DLQ como fallback terminal:** falhas após 5 tentativas são isoladas em `WebhookDeadLetter`, não bloqueando o processamento de outros eventos pendentes.
- **Isolamento do worker:** processo separado da API; falha ou crash do worker não afeta disponibilidade da API, apenas atrasa entregas até o processo ser reiniciado (monitoramento operacional, fora do código da feature).
- **Idempotência no cliente:** `X-Event-Id` estável entre tentativas (mesmo evento = mesmo id em todas as tentativas), permitindo deduplicação client-side ([ADR-004](./adrs/ADR-004-at-least-once-event-id.md)).
- **Limite de payload:** eventos cujo payload renderizado ultrapasse 64KB são rejeitados com `WEBHOOK_PAYLOAD_TOO_LARGE` no momento da inserção na outbox, em vez de truncados ([09:23]-[09:24] Sofia/Diego).

## Observabilidade

- **Logs:** todo o módulo usa o logger Pino já configurado em `src/shared/logger/index.ts`, incluindo redaction de dados sensíveis (a secret de um webhook deve ser adicionada aos `redactPaths` existentes, ex. `*.secret`). O worker loga início/fim de cada ciclo de polling, cada tentativa de entrega (sucesso/falha, `event_id`, `webhookEndpointId`, `durationMs`), e cada replay de DLQ com o `userId` do administrador.
- **Métricas (propostas, a instrumentar na implementação):** contagem de eventos inseridos na outbox por `eventType`; contagem de entregas por resultado (sucesso/falha/timeout); tamanho da fila de pendentes (`WebhookOutboxEvent` com `status = PENDING`); taxa de itens movidos para DLQ; latência de entrega (tempo entre `createdAt` do evento e `DELIVERED`).
- **Tracing:** reaproveitar o `request-logger.middleware.ts` existente para correlacionar `requestId` nas chamadas HTTP de configuração de webhook. Nas chamadas de saída do worker, usar o `event_id` como correlação entre o log de processamento e a linha de `WebhookDelivery` gerada.

## Dependências e compatibilidade

- Prisma/MySQL já existentes; novas tabelas via migration em `prisma/migrations/`.
- Nenhuma nova dependência de infraestrutura (sem Redis, sem broker de mensageria — [ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md)).
- Novo script `npm run worker` e novo entry-point `src/worker.ts`, espelhando `src/server.ts` ([09:11] Larissa).
- Biblioteca de HMAC: módulo nativo `crypto` do Node.js (sem dependência nova).
- Mantém compatibilidade total com os endpoints existentes de `orders`; nenhuma mudança de contrato nos endpoints atuais.

## Integração com o sistema existente

- **`src/modules/orders/order.service.ts`**: o método `changeStatus` é estendido para, dentro da mesma transação (`tx`), chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` após a atualização de status e histórico, antes do commit. Nenhuma mudança na assinatura pública do método ou no comportamento de erros existente (`InvalidStatusTransitionError`, `InsufficientStockError` continuam funcionando como hoje).
- **`src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts`**: novas classes de erro do módulo de webhooks (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`/`NotFoundError`/`UnprocessableEntityError` exatamente como `InsufficientStockError` e `InvalidStatusTransitionError` já fazem, reutilizando a mesma estrutura de `statusCode` + `errorCode` + `details`.
- **`src/middlewares/auth.middleware.ts`**: o endpoint `POST /admin/webhooks/dead-letter/:id/replay` reutiliza `requireRole('ADMIN')`, já usado para proteger rotas administrativas em outros módulos — nenhuma alteração nesse middleware é necessária.
- **`src/middlewares/error.middleware.ts`**: trata automaticamente qualquer erro `WEBHOOK_*` por herdar de `AppError`, assim como já trata `ZodError` e `Prisma.PrismaClientKnownRequestError` — nenhuma alteração necessária.
- **`src/shared/logger/index.ts`**: reutilizado tanto pela API (rotas de webhook) quanto pelo novo processo `src/worker.ts`; a lista `redactPaths` deve ser estendida para incluir o campo `secret`/`previousSecret` do `WebhookEndpoint`.
- **`src/server.ts`**: serve de modelo estrutural para o novo `src/worker.ts` — mesmo padrão de inicialização de `PrismaClient` e logger, porém como loop de polling em vez de servidor HTTP.
- **`prisma/schema.prisma`**: novos models `WebhookEndpoint`, `WebhookOutboxEvent`, `WebhookDelivery`, `WebhookDeadLetter`, seguindo o padrão de chave primária já usado (`@id @default(uuid()) @db.Char(36)`, visto em `User` e `Customer`), com relação `customerId` para o model `Customer` já existente.
- **`src/modules/customers/` (estrutura de referência)**: o novo módulo `src/modules/webhooks/` replica a mesma divisão de arquivos (`webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`), acrescida de `webhook.worker.ts`/`webhook.processor.ts` para a lógica de processamento usada pelo entry-point `src/worker.ts`.

## Critérios de aceite técnicos

- Nenhuma mudança de status de pedido é persistida sem que o(s) evento(s) de webhook elegíveis sejam inseridos na mesma transação (ou nenhum dos dois, em caso de rollback).
- Um evento inserido na outbox é entregue (ou movido para retry) em até ~2 segundos após sua inserção, no cenário de único worker ativo.
- Após 5 tentativas falhas, o evento aparece em `WebhookDeadLetter` e some da fila de pendentes ativa.
- Toda requisição de entrega inclui `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- `X-Signature` validável pelo cliente usando a secret ativa (ou a secret anterior, se dentro do grace period de 24h).
- Endpoint de replay de DLQ retorna `403` para usuários sem role `ADMIN` e registra log de auditoria em caso de sucesso.
- URLs não-HTTPS são rejeitadas na criação/edição do webhook com `WEBHOOK_INVALID_URL`.
- Payloads acima de 64KB nunca são inseridos na outbox; a operação retorna `WEBHOOK_PAYLOAD_TOO_LARGE`.

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Escrita adicional na transação de `changeStatus` degrada performance de mudança de status | Média | Médio | Medir latência antes/depois em staging; índices adequados em `WebhookOutboxEvent` |
| Worker trava ou cai sem alerta, atrasando todas as entregas | Média | Alto | Monitoramento/alerta operacional de processo vivo (fora do código desta feature, mas pré-requisito de deploy) |
| Vazamento de secret em log de cliente (já ocorreu no passado — [09:22] Diego) | Média | Alto | Rotação com grace period de 24h ([ADR-003](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)); redaction de secret nos próprios logs internos |
| Cliente não implementa deduplicação por `X-Event-Id` e processa eventos duplicados | Média | Médio | Documentação clara no portal do desenvolvedor ([09:26] Marcos) |

