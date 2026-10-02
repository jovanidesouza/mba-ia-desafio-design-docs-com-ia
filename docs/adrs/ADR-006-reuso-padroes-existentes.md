# ADR-006: Reuso dos padrões arquiteturais existentes do projeto

## Status

Aceito

## Contexto

A aplicação já possui convenções consolidadas de estrutura de código, tratamento de erros, logging e autorização, usadas de forma consistente nos módulos de `auth`, `users`, `customers`, `products` e `orders`. Bruno propôs explicitamente que o novo módulo de webhooks siga o mesmo padrão, em vez de introduzir convenções novas ([09:27]-[09:30] Bruno, validado por Diego e Larissa).

Pontos de reuso identificados na reunião e confirmados no código:

- **Estrutura modular:** cada domínio em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (ver `src/modules/customers/` como referência). O módulo de webhooks seguirá `src/modules/webhooks/` com a mesma divisão, mais um arquivo de processamento do worker (`webhook.worker.ts` ou `webhook.processor.ts`) ([09:27]-[09:28] Bruno/Diego).
- **Tratamento de erros:** a hierarquia `AppError` e suas subclasses (`src/shared/errors/http-errors.ts`, ex. `InsufficientStockError`, `InvalidStatusTransitionError`) já define o padrão de erro de domínio com `statusCode` e `errorCode`. Os erros do módulo de webhooks seguirão o mesmo padrão, com códigos prefixados **`WEBHOOK_`** (ex. `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
- **Middleware de erro centralizado:** `src/middlewares/error.middleware.ts` já trata instâncias de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` de forma genérica. Os novos erros `WEBHOOK_*`, por estenderem `AppError`, são tratados automaticamente, sem necessidade de alteração nesse middleware ([09:29] Bruno).
- **Logger:** Pino (`src/shared/logger/index.ts`), já configurado com redaction de dados sensíveis, é reaproveitado tanto na API quanto no worker, sem introduzir nova biblioteca de logging ([09:29] Bruno).
- **Autorização por role:** o middleware `requireRole` (`src/middlewares/auth.middleware.ts`), já usado para restringir rotas a `ADMIN`/`OPERATOR`, é reaproveitado para proteger o endpoint de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), que exige role `ADMIN` ([09:35]-[09:36] Sofia/Larissa).
- **Schemas de validação:** uso de Zod para validação de payloads de entrada, incluindo a regra de URL obrigatoriamente HTTPS ([09:23] Sofia), seguindo o padrão já usado em `*.schemas.ts` dos demais módulos.

## Decisão

O módulo de webhooks **não introduz nenhum padrão arquitetural novo**. Ele reaproveita integralmente:

1. A estrutura de módulo (`controller`/`service`/`repository`/`routes`/`schemas`) replicada de módulos existentes (ex. `src/modules/customers/`).
2. A hierarquia de erros `AppError` (`src/shared/errors/`), com novas subclasses de erro específicas do domínio, todas usando o prefixo de código `WEBHOOK_`.
3. O middleware de erro centralizado existente (`src/middlewares/error.middleware.ts`), sem modificações.
4. O logger Pino existente (`src/shared/logger/index.ts`), tanto na API quanto no processo worker.
5. O middleware `requireRole` existente (`src/middlewares/auth.middleware.ts`) para proteger o endpoint administrativo de replay.
6. O padrão de validação com Zod já usado nos demais módulos.

## Alternativas Consideradas

- **Introduzir uma camada de eventos/domain events genérica e reutilizável para todo o sistema** (não só webhooks): tecnicamente mais extensível a longo prazo, mas aumentaria o escopo e o tempo de entrega muito além do prazo de 3 sprints acordado, e não foi proposta nem discutida como necessidade imediata na reunião. Rejeitada implicitamente em favor de reaproveitar os padrões já existentes e manter o escopo enxuto.
- **Criar um novo esquema de códigos de erro específico para webhooks, fora do padrão `AppError`:** permitiria eventual customização, mas quebraria a consistência com o resto da base de código e exigiria alterações no middleware de erro centralizado. Descartada ([09:28]-[09:29] Bruno).

## Consequências

**Positivas:**
- Reduz drasticamente a curva de aprendizado para qualquer desenvolvedor já familiarizado com o restante da base de código.
- Nenhuma alteração é necessária em `error.middleware.ts`, `logger/index.ts` ou `auth.middleware.ts` — reduz superfície de risco de regressão em código compartilhado.
- Acelera a entrega dentro do prazo de 3 sprints estimado por Larissa ([09:46] Larissa), já que boa parte da "arquitetura" de suporte já existe e está testada em produção.

**Negativas:**
- Qualquer limitação ou débito técnico já existente nos padrões reaproveitados (ex. formato de log, granularidade de `AppError`) é herdado pelo módulo de webhooks.
- Acopla a evolução futura do módulo de webhooks às decisões de padrões tomadas para os módulos de domínio de negócio (orders, customers etc.), mesmo que o domínio de webhooks tenha características distintas (ex. processamento assíncrono via worker).
