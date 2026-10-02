# ADR-005: Worker em processo separado com polling de 2 segundos

## Status

Aceito

## Contexto

Com o padrão outbox definido (ADR-001), era preciso decidir como o processo que entrega os webhooks lê os eventos pendentes e onde esse processo roda.

Bruno perguntou se seria possível usar um trigger do banco para reagir de forma mais reativa aos novos eventos ([09:09] Bruno). Diego explicou que o MySQL não possui um mecanismo nativo equivalente ao `LISTEN`/`NOTIFY` do PostgreSQL: um trigger consegue executar SQL, mas não consegue notificar um processo externo; fazer isso exigiria soluções improvisadas (ex. escrever em arquivo ou chamar um endpoint a partir do trigger), consideradas inadequadas ([09:09] Diego). A alternativa escolhida foi **polling em loop**, a cada 2 segundos, buscando os eventos pendentes mais antigos, processando e marcando como entregues ([09:09] Diego). Esse intervalo atende com folga o requisito de latência "abaixo de 10 segundos" reportado pelos clientes B2B ([09:02] Marcos, [09:10] Marcos/Larissa).

Também ficou definido que o worker **deve rodar como processo separado** da API HTTP principal — "Senão se a API reinicia, perde o worker" ([09:11] Diego) — reaproveitando o mesmo padrão de entry-point já existente em `src/server.ts`, com um novo `src/worker.ts` e um script `npm run worker` ([09:11] Larissa). O worker conecta ao mesmo banco de dados, mas com sua própria instância de `PrismaClient`, já que esse cliente é vinculado ao processo Node em que roda ([09:11] Bruno/Diego, [09:29]-[09:30] Diego/Bruno).

Sobre ordenação de eventos: com um único worker processando em ordem de `created_at`, a entrega ao cliente respeita a ordem de mudança de status por pedido. Caso o sistema escale no futuro para múltiplos workers em paralelo, essa garantia se perde, a menos que se implemente particionamento por `order_id` ou lock pessimista — explicitamente documentado como problema futuro, não desta fase ([09:12]-[09:14] Diego/Larissa). Os clientes nunca pediram garantia de ordenação global, apenas de cada pedido individualmente ([09:14] Marcos).

## Decisão

- O envio de webhooks é feito por um **worker rodando em processo Node.js separado** da API HTTP (`src/server.ts`), com entry-point próprio (`src/worker.ts`) e script dedicado (`npm run worker`).
- O worker usa **polling em loop a cada 2 segundos**, buscando o lote mais antigo de eventos pendentes na outbox, processando-os e marcando-os como entregues ou agendando retry.
- O worker mantém sua **própria instância de `PrismaClient`**, conectada ao mesmo banco (`DATABASE_URL`) usado pela API, mas isolada por processo.
- **Limitação conhecida e aceita:** a ordenação de entrega é garantida apenas por `order_id` e apenas enquanto houver um único worker ativo. Não há garantia de ordenação global entre pedidos diferentes, nem entre múltiplos workers em paralelo (não implementado nesta fase).

## Alternativas Consideradas

- **Reação via trigger de banco de dados:** descartada porque o MySQL não oferece um mecanismo de notificação de processos externos (sem equivalente a `LISTEN`/`NOTIFY`); soluções alternativas via trigger (escrever em arquivo, chamar endpoint) foram consideradas inadequadas e frágeis ([09:09] Diego).
- **Worker embutido no mesmo processo da API:** mais simples operacionalmente (um único processo para subir), mas acopla o ciclo de vida do worker ao da API — um reinício ou crash da API derrubaria o worker junto. Descartada ([09:11] Diego).
- **Múltiplos workers em paralelo desde o início:** melhoraria a capacidade de processamento, mas introduz o problema de perda de garantia de ordenação por pedido sem mecanismo adicional (particionamento ou locking). Adiada para quando houver necessidade real de escala ([09:13] Diego).

## Consequências

**Positivas:**
- Isola o ciclo de vida do worker do ciclo de vida da API, evitando perda de processamento de eventos pendentes em reinícios/deploys da API.
- Polling de 2s é simples de implementar e operar, sem infraestrutura adicional, e atende com margem confortável o SLA de latência percebido pelo cliente (<10s).
- Ordenação por `order_id` com single-worker é suficiente para o requisito real dos clientes (não pediram ordenação global).

**Negativas:**
- Introduz uma segunda unidade de deploy/operação (processo worker) que precisa ser monitorada e mantida viva separadamente da API.
- Latência mínima de até 2 segundos é inerente ao modelo de polling (vs. um modelo reativo), ainda que aceitável.
- Escalar para múltiplos workers no futuro exigirá trabalho adicional de particionamento ou locking para não quebrar a garantia de ordenação por pedido — dívida técnica intencionalmente aceita.
