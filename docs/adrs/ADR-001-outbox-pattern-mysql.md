# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito

## Contexto

A feature de Webhooks de Notificação de Pedidos precisa publicar um evento sempre que o status de um pedido muda, de forma que um worker externo possa entregá-lo ao cliente via HTTP. A mudança de status já ocorre dentro de uma transação SQL em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos.

Durante a reunião técnica, a equipe descartou disparo síncrono do webhook dentro dessa transação: "Síncrono não rola. [...] Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno), e não haveria como fazer rollback de um envio HTTP já efetuado ([09:04] Bruno). A alternativa de introduzir uma fila externa (ex. Redis Streams) foi levantada e descartada por exigir infraestrutura nova para um time pequeno ([09:07] Larissa, Diego).

A solução proposta foi o padrão Outbox: dentro da mesma transação SQL que já altera o pedido, inserir também uma linha em uma tabela `webhook_outbox` com o evento a ser entregue. Um worker separado lê essa tabela de forma assíncrona e dispara as chamadas HTTP.

## Decisão

Adotar o padrão **Transactional Outbox** usando o MySQL já existente no projeto (via Prisma), sem introduzir nova infraestrutura de mensageria.

- A inserção do evento na tabela de outbox ocorre na **mesma transação Prisma** (`tx: Prisma.TransactionClient`) que atualiza `orders` e `order_status_history` em `OrderService.changeStatus`.
- Se a transação principal falhar, o evento nunca é persistido; se ela commitar, o evento está garantidamente registrado.
- A tabela de outbox possui índice nos campos de status de processamento (pendente, processando, falhou, entregue) e em `created_at`, para suportar leitura eficiente pelo worker ([09:08] Diego).
- Linhas já entregues não são removidas automaticamente por esta feature; arquivamento após 30 dias fica fora de escopo ([09:08] Diego).

## Alternativas Consideradas

- **Fila dedicada (ex. Redis Streams):** resolveria o desacoplamento, mas exigiria subir e operar nova infraestrutura. Descartada por ser "overengineering" para o tamanho do time atual ([09:07] Diego, Larissa).
- **Disparo síncrono dentro da transação de `changeStatus`:** mais simples de implementar, mas acopla a latência/disponibilidade de clientes externos à transação crítica de negócio, com risco de travar mudanças de status de outros pedidos e sem caminho claro de rollback. Descartada ([09:04] Bruno, Larissa).

## Consequências

**Positivas:**
- Consistência forte entre a mudança de estado do pedido e a geração do evento: nunca há mudança de status sem evento correspondente, nem evento "órfão" de uma transação que sofreu rollback.
- Reaproveita a infraestrutura de banco já existente (MySQL + Prisma), sem custo operacional adicional.
- Desacopla a latência de entrega HTTP da transação de negócio principal.

**Negativas:**
- Introduz latência mínima entre a mudança de status e a entrega do webhook, equivalente ao intervalo de polling do worker (ver ADR-005).
- A tabela de outbox cresce continuamente até que uma rotina de arquivamento seja implementada (fora do escopo atual), exigindo atenção futura de manutenção.
- Requer que toda alteração de status futura que precise gerar eventos passe pela mesma disciplina transacional — risco de inconsistência se um desenvolvedor futuro alterar `changeStatus` sem considerar a outbox.
