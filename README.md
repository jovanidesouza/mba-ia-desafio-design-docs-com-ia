# Sistema de Webhooks de Notificação de Pedidos — Design Docs

> O enunciado original do desafio foi preservado em [docs/DESAFIO.md](docs/DESAFIO.md). Este README documenta o processo de produção do pacote de documentação técnica.

## Sobre o desafio

Este repositório é a entrega de um desafio que consiste em transformar a transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) em um pacote completo de design docs para uma nova feature de um Order Management System: um sistema de webhooks que notifica clientes B2B sobre mudanças de status de pedidos. A aplicação Node.js + TypeScript + Prisma/MySQL já existente serve apenas como contexto e referência — nenhuma linha de código de `src/`, `prisma/` ou `tests/` foi alterada.

O ponto central do exercício não é "gerar documentação com IA", mas usar a IA como ferramenta de produção mantendo rigor: toda decisão, requisito ou restrição registrada precisa ser rastreável a um trecho específico da transcrição ou a um arquivo real do código, nunca inventada. O arquivo [docs/TRACKER.md](docs/TRACKER.md) existe exatamente para tornar essa rastreabilidade verificável.

## Ferramentas de IA utilizadas

- **GitHub Copilot Chat (agente, modelo Claude Sonnet 4.5/5)**, dentro do VS Code — ferramenta única e principal de produção deste pacote. Foi usada para: ler a transcrição e o código-fonte diretamente do workspace, extrair decisões/requisitos/exclusões com prompts dirigidos, redigir os 7 ADRs, o RFC, o FDD, o PRD e o Tracker, e montar este README. O uso em modo agente permitiu que a IA lesse arquivos reais do repositório (`order.service.ts`, `http-errors.ts`, `auth.middleware.ts`, `schema.prisma` etc.) antes de escrever qualquer seção que os referenciasse, em vez de descrever o código "de memória".

## Workflow adotado

A ordem de produção seguiu a recomendação do próprio enunciado, por ser a que gera menos retrabalho: as decisões formam a base de tudo o que vem depois.

1. **Grounding inicial:** antes de escrever qualquer documento, a IA foi instruída a ler a transcrição inteira e os arquivos de código mais citados na reunião (`order.service.ts`, `http-errors.ts`, `auth.middleware.ts`, `error.middleware.ts`, `logger/index.ts`, `schema.prisma`), e a montar um mapeamento de decisões, requisitos, exclusões e caminhos de código reais **antes** de redigir qualquer documento. Esse mapeamento foi mantido como anotação de trabalho para ser reaproveitado em todos os documentos subsequentes, evitando que a IA "esquecesse" ou reinterpretasse a transcrição de forma diferente a cada arquivo gerado.
2. **ADRs primeiro** (`docs/adrs/`): as 7 decisões arquiteturais foram escritas uma a uma, cada uma citando os timestamps exatos da fala que a originou.
3. **RFC** (`docs/RFC.md`): consolidado em cima dos ADRs já prontos, com link direto para cada um na seção "Decisões relacionadas".
4. **FDD** (`docs/FDD.md`): o documento mais extenso, escrito depois do RFC para poder aprofundar tecnicamente sem repetir o que já estava descrito em nível de arquitetura.
5. **PRD** (`docs/PRD.md`): escrito por último entre os documentos "grandes", como consolidação do que já havia sido decidido tecnicamente, focado em reformular tudo em linguagem de produto/negócio.
6. **Tracker** (`docs/TRACKER.md`): montado por último, varrendo cada um dos quatro documentos e os sete ADRs linha a linha para gerar uma entrada rastreável por item.
7. **Este README**: escrito ao final, descrevendo o processo já concluído.

## Prompts customizados

Dois exemplos representativos dos prompts usados para dirigir a filtragem de conteúdo (em vez de pedir geração genérica):

```
Leia TRANSCRICAO.md por completo. Separe o conteúdo em três grupos,
citando sempre o timestamp [hh:mm] e o nome de quem falou:

1. Decisões fechadas (algo que o time concordou em fazer de uma forma
   específica, não apenas "foi mencionado")
2. Requisitos funcionais explícitos (algo que o sistema PRECISA fazer)
3. Itens descartados ou adiados explicitamente (algo que foi cogitado
   mas o time decidiu NÃO fazer agora, ou fazer depois)

Não inclua nada que seja apenas uma pergunta não respondida ou uma
observação lateral sem decisão associada. Se não houver timestamp
claro para um item, não o inclua.
```

```
Antes de escrever a seção "Integração com o sistema existente" do FDD,
abra e leia o conteúdo real dos seguintes arquivos: order.service.ts
(método changeStatus), http-errors.ts, auth.middleware.ts,
error.middleware.ts, logger/index.ts e schema.prisma.

Para cada um, descreva a integração citando apenas comportamento que
você verificou existir no arquivo (ex: nome exato de classes, métodos
e middlewares). Se eu mencionar algo que não existe no código que você
leu, me avise em vez de assumir que existe.
```

## Iterações e ajustes

Três ciclos principais de revisão até o resultado final:

1. **Primeira versão do FDD era genérica demais na seção de integração.** A primeira tentativa descrevia "o módulo vai reaproveitar o tratamento de erros existente" sem nomear classes ou arquivos reais. Foi necessário pedir explicitamente que a IA lesse `http-errors.ts` e `auth.middleware.ts` antes de escrever a seção, e citasse nomes exatos (`AppError`, `requireRole`, `InsufficientStockError`) — só assim a seção passou a atender o critério de nomear ao menos 4 caminhos de arquivo reais.
2. **Risco de confundir itens decididos com itens apenas cogitados.** Na primeira passada, "rate limiting de saída" e "dashboard visual" quase entraram como requisito funcional do PRD, quando na verdade a transcrição mostra claramente que foram adiados ("observar e decidir depois", "não, agora não"). Foi preciso revisar a seção "Fora de escopo" linha a linha contra a transcrição, reforçando a regra de só registrar como requisito algo com decisão fechada associada a um timestamp.
3. **Tracker com cobertura insuficiente na primeira tentativa.** A primeira versão do tracker cobria bem o PRD e o FDD, mas deixava a maior parte das alternativas descartadas do RFC e das consequências negativas dos ADRs sem linha correspondente. Foi necessário varrer novamente RFC e ADRs seção por seção para elevar a cobertura acima do mínimo de 80% exigido, incluindo explicitamente linhas de risco e trade-off que haviam sido omitidas.

## Como navegar a entrega

Ordem sugerida de leitura, da visão de produto até o detalhe de implementação:

1. [docs/PRD.md](docs/PRD.md) — por quê e o quê: problema, público, escopo e métricas.
2. [docs/RFC.md](docs/RFC.md) — proposta técnica, alternativas descartadas e questões em aberto.
3. [docs/adrs/](docs/adrs/) — cada decisão arquitetural isolada (ADR-001 a ADR-007), com contexto e consequências.
4. [docs/FDD.md](docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
5. [docs/TRACKER.md](docs/TRACKER.md) — referência cruzada de todo item de qualquer documento acima com sua origem na transcrição ou no código.
6. [TRANSCRICAO.md](TRANSCRICAO.md) — fonte primária, para quem quiser conferir qualquer timestamp citado nos documentos.

