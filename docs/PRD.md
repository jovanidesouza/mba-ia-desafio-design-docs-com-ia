# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS (Order Management System) hoje expõe apenas consulta via `GET /orders` para que clientes B2B acompanhem o status de seus pedidos. Três clientes relevantes — **Atlas Comercial, MaxDistribuição e Nova Cargo** — solicitaram formalmente a capacidade de serem notificados automaticamente quando o status de um pedido muda, em vez de precisar consultar a API repetidamente ([09:00] Marcos). Esta feature introduz um sistema de **webhooks outbound**: a plataforma envia uma notificação HTTP ao sistema do cliente sempre que o status de um pedido dele muda.

## Problema e motivação

Os clientes fazem polling periódico em `GET /orders` para detectar mudanças, o que eles descrevem como uma integração "lenta e cara" ([09:00] Marcos). Isso gera carga desnecessária na API e atraso na percepção de mudanças de status pelos sistemas dos clientes. Há risco comercial direto: a Atlas Comercial sinalizou que pode migrar para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre ([09:00] Marcos).

## Público-alvo e cenários de uso

- **Público-alvo:** clientes B2B integrados via API que precisam reagir a mudanças de status de pedidos em seus próprios sistemas (ex. atualizar status em um ERP interno, disparar um processo de logística).
- **Cenário principal:** um pedido do cliente Atlas Comercial muda de `PAID` para `SHIPPED`. A plataforma identifica automaticamente que a Atlas tem um webhook cadastrado para o status `SHIPPED`, monta o evento e o entrega ao endpoint HTTP configurado pela Atlas, assinado e identificado de forma única.
- **Cenário de falha temporária:** o endpoint do cliente está indisponível (ex. deploy em andamento). A plataforma tenta novamente de forma espaçada, sem perder o evento, até esgotar as tentativas.
- **Cenário de auditoria:** o time de operação da plataforma precisa investigar por que um cliente não recebeu uma notificação; consulta o histórico de entregas daquele webhook.

## Objetivos e métricas de sucesso

- **Latência de notificação:** entregar eventos de mudança de status em **menos de 10 segundos** desde a mudança de status, no cenário de operação normal (sem falhas de rede) — meta explicitamente reportada pelos clientes como equivalente a "tempo real" ([09:02] Marcos). O worker com polling de 2 segundos atende essa meta com margem.
- **Confiabilidade de entrega:** garantir que **100% das mudanças de status elegíveis gerem um evento de outbox**, sem perda por falha da transação principal (consequência direta do padrão outbox, [ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md)).
- **Retenção de clientes B2B estratégicos:** evitar a migração da Atlas Comercial para concorrente, citada como risco explícito na reunião ([09:00] Marcos).

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por cliente (CRUD de configuração).
- Filtro de quais status de pedido cada webhook deseja receber.
- Entrega assíncrona via worker dedicado, com retry e Dead Letter Queue (DLQ).
- Autenticação de payload via HMAC-SHA256, com secret única por endpoint e rotação com grace period.
- Identificação única de evento (`X-Event-Id`) para suportar deduplicação no cliente.
- Consulta de histórico de entregas por webhook.
- Reprocessamento manual de itens em DLQ, restrito a administradores.

### Fora de escopo

- **Notificação de fallback por e-mail** quando um webhook falha repetidamente — adiado para uma fase futura, após medição do impacto da entrega atual ([09:37]-[09:38] Marcos/Larissa).
- **Dashboard visual** para o cliente acompanhar seus webhooks — considerado fora de escopo desta feature, por ser um projeto de frontend separado ([09:39]-[09:40] Larissa/Marcos).
- **Rate limiting de envio de saída** para clientes com alto volume de mudanças de status — não decidido nesta fase; a equipe optou por observar o comportamento em produção antes de desenhar uma solução ([09:38]-[09:39] Diego/Larissa).
- **Garantia de ordenação global** entre múltiplos workers em paralelo — fora de escopo; a solução atual garante ordenação apenas por `order_id` em cenário de worker único ([09:12]-[09:14] Diego/Bruno/Larissa).
- **Arquivamento automático** de eventos já entregues (ex. após 30 dias) — citado como possível evolução futura, não faz parte desta entrega ([09:08] Diego).
- **Entrega inbound** (clientes enviando webhooks para a plataforma) — explicitamente fora de escopo; o fluxo é somente outbound ([09:02]-[09:03] Marcos/Sofia).

## Requisitos funcionais

1. O cliente (via API autenticada) deve poder **cadastrar um webhook**, informando `url` e lista de status de interesse; a `secret` é gerada pela plataforma e devolvida apenas na criação ([09:31]-[09:32] Marcos).
2. O cliente deve poder **editar** um webhook existente (URL, filtro de eventos, ativo/inativo) ([09:33] Bruno).
3. O cliente deve poder **remover** um webhook cadastrado ([09:33] Bruno).
4. O cliente deve poder **listar** os webhooks cadastrados para um customer ([09:33] Bruno).
5. Cada webhook deve poder **filtrar quais status de pedido** deseja receber (ex. apenas `SHIPPED` e `DELIVERED`); o filtro é aplicado no momento da geração do evento, não no envio ([09:33]-[09:34] Marcos/Bruno/Diego).
6. O cliente deve poder **consultar o histórico das últimas 100 entregas** de um webhook, incluindo sucesso/falha, payload, resposta recebida e tempo de resposta ([09:34] Marcos).
7. Um administrador deve poder **reprocessar manualmente** um evento que caiu em DLQ, via endpoint restrito à role `ADMIN` ([09:18] Diego, [09:35]-[09:36] Sofia/Larissa).
8. O cliente deve poder **rotacionar a secret** de um webhook via API, mantendo a secret anterior válida por 24 horas em paralelo ([09:21] Sofia).
9. A plataforma deve **validar que a URL cadastrada é HTTPS**, recusando URLs `http://` ([09:23] Sofia).
10. A plataforma deve **recusar eventos cujo payload ultrapasse 64KB**, em vez de truncá-los ([09:23]-[09:24] Sofia/Diego).

## Requisitos não funcionais

- **Latência:** entrega em até 10 segundos em condições normais, via polling de 2 segundos ([09:02]-[09:10]).
- **Confiabilidade:** garantia de entrega **at-least-once**, com `X-Event-Id` único por evento para suportar deduplicação do lado do cliente ([09:24]-[09:26] Diego).
- **Resiliência:** retry com backoff exponencial de 5 tentativas (1m/5m/30m/2h/12h) antes de mover para DLQ ([09:15]-[09:17] Diego).
- **Segurança:** assinatura HMAC-SHA256 por payload, secret única por endpoint, TLS obrigatório, timeout de 10s por chamada ([09:19]-[09:24], [09:42] Sofia/Diego).
- **Isolamento operacional:** entrega processada por worker em processo separado da API, para não impactar a disponibilidade da API em caso de falha do worker ([09:11] Diego/Larissa).
- **Consistência transacional:** geração do evento deve ocorrer na mesma transação da mudança de status, sem exceções ([09:40]-[09:41] Bruno/Diego).

## Decisões e trade-offs principais

- Outbox no MySQL em vez de fila dedicada, trocando simplicidade operacional por uma latência mínima de alguns segundos (ver [RFC](./RFC.md) e [ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md)).
- At-least-once em vez de exactly-once, transferindo a responsabilidade de deduplicação para o cliente em troca de uma implementação muito mais simples ([ADR-004](./adrs/ADR-004-at-least-once-event-id.md)).
- 5 tentativas de retry (~15h de janela) em vez de um número menor, para cobrir indisponibilidades reais de clientes já observadas no passado ([ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)).
- Ordenação garantida apenas por `order_id` com worker único, aceitando a limitação em troca de não introduzir complexidade de particionamento prematuramente ([09:12]-[09:14]).

## Dependências

- Banco de dados MySQL e Prisma já existentes (sem nova infraestrutura).
- Middleware de autenticação JWT e `requireRole` já existentes (`src/middlewares/auth.middleware.ts`).
- Hierarquia de erros `AppError` e logger Pino já existentes (`src/shared/errors/`, `src/shared/logger/`).
- Disponibilidade da equipe de segurança (Sofia) para revisão dedicada antes do deploy — 2 dias úteis reservados ([09:46] Sofia).
- Comunicação ao cliente sobre o comportamento de deduplicação via portal do desenvolvedor ([09:26] Marcos).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Vazamento de secret de webhook em logs do cliente (incidente já ocorrido antes) | Média | Alto | Rotação de secret com grace period de 24h, permitindo resposta rápida sem quebrar integração ativa ([09:21]-[09:22] Sofia/Diego) |
| Atraso na revisão de segurança compromete o prazo de 3 sprints / fim de novembro acordado com a Atlas | Média | Alto | Reservar antecipadamente 2 dias úteis da Sofia no cronograma ([09:46]-[09:47]) |
| Worker indisponível (crash/travamento) atrasa todas as notificações sem erro visível na API | Baixa | Alto | Processo separado da API, para isolar falhas; necessidade de monitoramento operacional dedicado ([09:11] Diego) |
| Cliente não implementa deduplicação e sofre efeitos colaterais duplicados | Média | Médio | Documentação explícita do comportamento at-least-once no portal do desenvolvedor ([09:25]-[09:26] Diego/Marcos) |

## Critérios de aceitação

- Todos os 10 requisitos funcionais listados estão implementados e cobertos por testes automatizados.
- Uma mudança de status de pedido elegível sempre resulta em um evento de outbox correspondente, mesmo sob falha simulada de rede do worker.
- Um webhook cadastrado com URL `http://` é rejeitado na criação.
- Um evento que falha 5 vezes aparece na DLQ e pode ser reprocessado manualmente por um usuário `ADMIN`.
- O histórico de entregas de um webhook retorna corretamente sucesso/falha, payload e tempo de resposta das últimas 100 tentativas.

## Estratégia de testes e validação

- **Testes unitários:** cobertura da função `publishWebhookEvent` (inserção condicionada ao filtro de eventos), cálculo de backoff exponencial, geração e validação de assinatura HMAC.
- **Testes de integração:** fluxo completo de `changeStatus` validando que o evento de outbox é criado na mesma transação (incluindo cenário de rollback sem geração de evento), seguindo o padrão já usado em `tests/orders.test.ts`.
- **Testes do worker:** simulação de endpoint de cliente indisponível/lento (timeout) validando a progressão de retry e a movimentação final para DLQ.
- **Testes de segurança:** validação de que a assinatura HMAC rejeita payload adulterado; validação da janela de 24h de grace period na rotação de secret — a serem revisados por Sofia antes do deploy ([09:46] Sofia).
- **Testes de contrato:** validação de schemas Zod para todos os endpoints (URL HTTPS obrigatória, limite de 64KB no payload do evento).

