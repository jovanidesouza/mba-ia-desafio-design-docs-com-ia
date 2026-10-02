# ADR-007: Snapshot do payload no momento da inserção do evento na outbox

## Status

Aceito

## Contexto

Ao definir o que fica armazenado na tabela de outbox, Bruno levantou a dúvida: o evento guarda o payload já renderizado (os dados do pedido no formato final a ser enviado), ou guarda apenas o `order_id` e renderiza o payload no momento do envio pelo worker ([09:51] Bruno)?

Larissa decidiu pela primeira opção — payload renderizado (snapshot) no momento da inserção — justificando que, se o pedido mudar novamente antes do envio efetivo (ou entre tentativas de retry), o evento ainda deve refletir fielmente o estado do pedido **no momento em que aquela transição de status ocorreu**, evitando inconsistências entre o que o evento descreve e o estado atual do pedido no banco ([09:52] Larissa). Diego e Bruno concordaram, confirmando a decisão ([09:52] Diego/Bruno).

Essa decisão também interage com ADR-005 (retry): como o mesmo evento pode ser reenviado múltiplas vezes ao longo de até ~15 horas (ADR-002), é essencial que o conteúdo enviado em cada tentativa seja idêntico ao da primeira tentativa, correspondendo ao estado do pedido exatamente no instante da transição que originou o evento — e não ao estado atual (possivelmente já alterado) do pedido.

Relacionado a essa mesma decisão de modelagem, ficou definido que o identificador do evento na outbox segue o padrão já usado em todo o projeto: **UUID**, e não id auto incremental ([09:51] Larissa/Diego) — consistente com os modelos existentes em `prisma/schema.prisma` (ex. `User.id`, `Customer.id`, todos `@id @default(uuid()) @db.Char(36)`).

## Decisão

- O payload do evento de webhook é **renderizado e persistido (snapshot) no momento da inserção na outbox**, dentro da mesma transação que efetua a mudança de status (ver ADR-001). O worker não recalcula ou busca dados adicionais do pedido no momento do envio — ele envia exatamente o payload armazenado.
- O identificador do evento de outbox (`event_id`) é um **UUID gerado no momento da inserção**, seguindo o padrão de identificadores já estabelecido em todo o schema Prisma do projeto (`@default(uuid())`).

## Alternativas Consideradas

- **Guardar apenas `order_id` e renderizar o payload no momento do envio:** reduziria o tamanho da linha armazenada na outbox e sempre refletiria o estado "mais atual" do pedido, mas quebraria a semântica de "o que mudou nesta transição específica" sempre que o pedido sofresse mudanças adicionais entre a inserção do evento e o envio (ou entre tentativas de retry) — produzindo payloads inconsistentes entre tentativas do mesmo evento. Descartada ([09:51]-[09:52] Bruno/Larissa/Diego).
- **ID auto incremental para a tabela de outbox:** mais simples e levemente mais eficiente em índices, mas inconsistente com o padrão de identificadores já adotado em 100% das outras tabelas do projeto. Descartada em favor de UUID ([09:51] Larissa).

## Consequências

**Positivas:**
- Garante que todas as tentativas de entrega de um mesmo evento (inclusive em retries ao longo de horas) carreguem exatamente o mesmo conteúdo, condizente com o estado do pedido no instante da transição que gerou o evento.
- Mantém consistência de modelagem com o restante do schema do projeto (uso uniforme de UUID como chave primária).
- Simplifica o worker, que não precisa consultar novamente o estado do pedido no momento do envio — apenas lê e envia o payload já armazenado.

**Negativas:**
- Aumenta o tamanho de cada linha da tabela de outbox, por armazenar o payload completo (ainda que limitado a 64KB) em vez de apenas uma referência (`order_id`).
- Se for necessário, no futuro, alterar o formato do payload retroativamente (ex. adicionar um campo), eventos já inseridos na outbox antes da mudança manterão o formato antigo — exige atenção a versionamento de payload em evoluções futuras, não coberta por este ADR.
