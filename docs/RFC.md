# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Reunião técnica realizada em quinta-feira, 09:00–09:53 (ver `TRANSCRICAO.md`) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |
| **Documentos relacionados** | [PRD](./PRD.md), [FDD](./FDD.md), ADRs em [`docs/adrs/`](./adrs/) |

## TL;DR

Vamos construir um sistema de **webhooks outbound** para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de pedidos, substituindo o polling atual via `GET /orders`. A solução usa o **padrão Outbox no MySQL já existente** (sem nova infraestrutura de mensageria): a mudança de status em `OrderService.changeStatus` insere um evento na mesma transação, e um **worker em processo separado**, rodando em polling de 2 segundos, lê a outbox e entrega os eventos via HTTP, com **retry exponencial de 5 tentativas e DLQ** para falhas permanentes. Segurança é garantida por **HMAC-SHA256 com secret por endpoint, rotacionável**, e a semântica de entrega é **at-least-once**, com deduplicação do lado do cliente via `X-Event-Id`. O módulo segue os padrões de código já estabelecidos no projeto (estrutura modular, `AppError`, Pino, `requireRole`).

## Contexto e problema

A plataforma opera um Order Management System em produção. Três clientes B2B relevantes — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente a capacidade de serem notificados em tempo real quando o status de seus pedidos muda ([09:00] Marcos). Hoje, esses clientes fazem polling periódico em `GET /orders`, o que é descrito como "lento e caro" para a integração deles ([09:00] Marcos). Há risco comercial concreto: a Atlas sinalizou possível migração para um concorrente caso a entrega não ocorra até o fim do trimestre ([09:00] Marcos).

O requisito de "tempo real" foi quantificado: para os clientes, qualquer latência abaixo de **10 segundos** já é aceitável ([09:02] Marcos). O escopo é estritamente **outbound** — a plataforma envia notificações para os sistemas dos clientes; não há necessidade de os clientes enviarem webhooks de volta ([09:02]-[09:03] Marcos/Sofia).

A base de código atual não possui nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status de pedidos já acontece dentro de uma transação SQL robusta (`OrderService.changeStatus`, em `src/modules/orders/order.service.ts`), que atualiza o pedido, grava histórico de status e ajusta estoque. Essa feature precisa se integrar a esse fluxo sem comprometer sua atomicidade nem sua performance.

## Proposta técnica

A proposta combina seis decisões arquiteturais centrais, cada uma formalizada em um ADR dedicado:

1. **Outbox transacional no MySQL** ([ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md)): a inserção do evento de webhook ocorre na mesma transação Prisma que já altera o status do pedido. Isso garante que todo evento de status gerado tenha exatamente uma mudança de estado correspondente (e vice-versa), sem depender de infraestrutura de mensageria adicional.

2. **Worker dedicado em polling** ([ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md)): um processo Node.js separado (`src/worker.ts`), com seu próprio `PrismaClient`, varre a outbox a cada 2 segundos em busca de eventos pendentes e dispara as entregas HTTP. Rodar como processo separado evita que o worker seja derrubado junto com reinícios da API.

3. **Resiliência via retry e DLQ** ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)): entregas que falham são reprocessadas com backoff exponencial (1m/5m/30m/2h/12h, 5 tentativas). Esgotadas as tentativas, o evento vai para uma tabela de Dead Letter Queue dedicada, reprocessável manualmente por um administrador via endpoint autenticado.

4. **Segurança por HMAC-SHA256** ([ADR-003](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)): cada endpoint de webhook cadastrado tem uma secret própria, usada para assinar o corpo de cada requisição (header `X-Signature`). Secrets são rotacionáveis com grace period de 24 horas.

5. **Entrega at-least-once com deduplicação no cliente** ([ADR-004](./adrs/ADR-004-at-least-once-event-id.md)): cada evento carrega um `event_id` (UUID) enviado no header `X-Event-Id`, permitindo que o cliente identifique e ignore entregas duplicadas — modelo adotado por players como Stripe e GitHub.

6. **Reuso integral dos padrões do projeto** ([ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md)): o módulo `src/modules/webhooks/` segue a mesma estrutura (`controller`/`service`/`repository`/`routes`/`schemas`) dos módulos existentes; erros usam a hierarquia `AppError` com códigos prefixados `WEBHOOK_*`; logging usa Pino; autorização administrativa reusa `requireRole`.

Complementarmente, o payload de cada evento é um **snapshot renderizado no momento da inserção** na outbox, não recalculado no envio ([ADR-007](./adrs/ADR-007-snapshot-payload-outbox.md)), garantindo que todas as tentativas de um mesmo evento carreguem conteúdo idêntico e fiel ao instante da transição de status.

Do ponto de vista de superfície de API, o sistema expõe: CRUD de configuração de webhook por cliente (`url`, `secret` gerada automaticamente, filtro de eventos por status), consulta de histórico de entregas, rotação de secret, e um endpoint administrativo de reprocessamento de DLQ restrito a `ADMIN`. O detalhamento de contratos, payloads e fluxos fica no [FDD](./FDD.md).

## Alternativas consideradas

- **Fila dedicada (ex. Redis Streams) em vez de Outbox no MySQL.** Resolveria o desacoplamento entre a transação de negócio e a entrega, mas exigiria subir e operar nova infraestrutura de mensageria. Para um time pequeno, isso foi julgado overengineering frente ao ganho — o MySQL já existente, com índices adequados, atende à necessidade. **Descartada** ([09:06]-[09:07] Diego/Larissa).

- **Disparo síncrono do webhook dentro da transação de `changeStatus`.** Seria a implementação mais direta, mas acoplaria a disponibilidade/latência de sistemas de clientes externos à transação crítica que também atualiza estoque e histórico — um cliente lento travaria mudanças de status de outros pedidos, e não haveria como reverter uma chamada HTTP já efetuada em caso de rollback. **Descartada** ([09:04] Bruno/Larissa).

- **Reação via trigger de banco de dados em vez de polling.** O MySQL não possui mecanismo equivalente ao `LISTEN`/`NOTIFY` do PostgreSQL; um trigger não é capaz de notificar um processo externo, apenas executar SQL. Soluções de contorno (escrever em arquivo, chamar endpoint a partir do trigger) foram consideradas frágeis e fora de padrão. **Descartada** em favor de polling de 2 segundos ([09:09] Diego).

- **3 tentativas de retry (em vez de 5).** Mais agressivo e liberaria recursos mais rápido, mas esgotaria em cerca de 30 minutos — insuficiente para cobrir janelas reais de indisponibilidade de clientes (ex. manutenção planejada de até 2 horas já observada no passado). **Descartada** em favor de 5 tentativas com backoff estendido até 12h ([09:16] Bruno/Diego).

- **Garantia de entrega exactly-once.** Eliminaria a necessidade de deduplicação do lado do cliente, mas exigiria coordenação transacional bidirecional entre plataforma e cliente, com complexidade de implementação muito maior para resolver um problema que a deduplicação via `event_id` já resolve na prática. **Descartada** em favor de at-least-once ([09:24]-[09:25] Diego/Sofia).

## Questões em aberto

- **Rate limiting de envio ao cliente.** Se um cliente tiver um volume alto de pedidos mudando de status em um curto intervalo (ex. 50 em um minuto), a plataforma hoje dispararia todas as chamadas sem controle de taxa. A equipe decidiu não endereçar isso nesta fase, mas **observar o comportamento em produção e decidir depois** se é necessário implementar throttling de saída ([09:38]-[09:39] Diego/Larissa).

- **Escalabilidade do worker para múltiplas instâncias.** A garantia de ordenação de entrega por `order_id` depende de haver um único worker ativo processando a outbox em ordem de `created_at`. Caso seja necessário escalar para múltiplos workers em paralelo no futuro, será preciso introduzir particionamento por `order_id` ou lock pessimista — nenhuma dessas soluções foi desenhada ou decidida nesta reunião, ficando como trabalho futuro explicitamente fora do escopo atual ([09:12]-[09:14] Diego/Bruno/Larissa).

- **Notificação de fallback (e-mail) em caso de falhas repetidas.** Foi cogitado alertar o cliente por e-mail após falhas consecutivas de entrega, mas a decisão foi adiar essa funcionalidade para uma fase futura, condicionada à medição do impacto real da feature atual ([09:37]-[09:38] Marcos/Larissa).

## Impacto e riscos

- **Impacto em `OrderService.changeStatus`:** a transação principal de mudança de status passa a incluir a inserção do evento de outbox. Isso é intencional (ADR-001) para preservar atomicidade, mas introduz uma nova dependência de escrita dentro de uma transação já considerada "pesada" pelo time ([09:04] Bruno) — deve ser medido o impacto de performance dessa escrita adicional.
- **Nova superfície operacional:** o worker é um novo processo que precisa ser deployado, monitorado e mantido vivo de forma independente da API. Falhas no worker (travamento, crash) não derrubam a API, mas pausam a entrega de notificações sem gerar erro visível ao usuário da API — exige observabilidade dedicada (ver [FDD](./FDD.md)).
- **Risco de segurança em segredos:** a plataforma já teve um incidente de vazamento de secret em log de cliente ([09:22] Diego). A rotação com grace period de 24h (ADR-003) mitiga esse risco ao permitir resposta rápida sem quebrar integrações ativas.
- **Risco de prazo:** a estimativa de Larissa é de 3 sprints até fim de novembro, incluindo 2 dias úteis reservados para revisão de segurança por Sofia antes do deploy ([09:45]-[09:47]). Atraso nessa revisão pode comprometer o prazo comprometido com a Atlas.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](./adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com X-Event-Id](./adrs/ADR-004-at-least-once-event-id.md)
- [ADR-005 — Worker em processo separado com polling](./adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./adrs/ADR-006-reuso-padroes-existentes.md)
- [ADR-007 — Snapshot do payload na inserção do evento](./adrs/ADR-007-snapshot-payload-outbox.md)
