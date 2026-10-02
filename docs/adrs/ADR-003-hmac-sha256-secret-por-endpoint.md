# ADR-003: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

Os webhooks expõem dados de pedidos para sistemas fora da infraestrutura da empresa. Sofia (engenharia de segurança) levantou a necessidade de o cliente conseguir validar que a requisição realmente partiu da plataforma e que o payload não foi adulterado em trânsito ([09:19] Sofia).

A proposta foi assinar o corpo da requisição com uma chave secreta compartilhada entre a plataforma e o cliente, usando HMAC, e enviar a assinatura em um header (`X-Signature`) para o cliente verificar do lado dele ([09:20] Sofia). O algoritmo escolhido foi SHA-256 por ser "padrão de mercado", com suporte amplo em bibliotecas client-side ([09:20] Sofia).

Também foi decidido que a secret não pode ser global da plataforma, e sim única por endpoint de webhook cadastrado — "se vaza uma, vaza tudo" ([09:21] Sofia) — e que a tabela de configuração do webhook deve armazenar `url + secret + customer_id + estado ativo` ([09:21] Bruno). Diego reforçou a importância citando um incidente real: um cliente já vazou uma secret em log de aplicação no passado ([09:22] Diego). Por isso, a secret precisa ser rotacionável via endpoint de API, com a secret antiga permanecendo válida por 24 horas em paralelo com a nova, dando tempo do cliente migrar seus sistemas ([09:21] Sofia).

## Decisão

- Toda requisição de webhook é assinada com **HMAC-SHA256** sobre o corpo (body) da requisição, enviado no header `X-Signature`.
- Cada **endpoint de webhook cadastrado possui sua própria secret**, gerada pela plataforma e devolvida ao cliente apenas no momento da criação do cadastro ([09:31] Marcos). Não existe secret global compartilhada entre endpoints ou customers.
- A plataforma expõe um endpoint de **rotação de secret** (`POST /webhooks/:id/secret/rotate`). Ao rotacionar, a secret anterior permanece válida por **24 horas** (grace period) em paralelo com a nova, após o que é invalidada.
- URLs de webhook devem ser obrigatoriamente **HTTPS**; URLs `http://` são rejeitadas na validação do schema Zod de cadastro ([09:23] Sofia). Esta validação é tratada como requisito não funcional de implementação, não uma decisão arquitetural separada.

## Alternativas Consideradas

- **Secret única global por cliente (customer), não por endpoint:** mais simples de gerenciar, mas um vazamento comprometeria todos os endpoints daquele cliente simultaneamente. Descartada em favor de isolamento por endpoint ([09:21] Sofia).
- **Rotação de secret sem grace period** (substituição imediata): mais simples de implementar, mas quebraria a integração do cliente no instante da rotação, sem janela para atualização coordenada. Descartada em favor de grace period de 24h ([09:21] Sofia).
- **Outro algoritmo de assinatura (ex. HMAC-SHA1):** descartado por ser considerado criptograficamente mais fraco e menos padrão de mercado que SHA-256 ([09:20] Sofia).

## Consequências

**Positivas:**
- Isolamento de blast radius: o vazamento da secret de um endpoint não compromete os demais endpoints do mesmo cliente ou de outros clientes.
- Rotação sem downtime: o grace period de 24h permite ao cliente atualizar sua integração sem perder eventos durante a transição.
- Uso de um padrão amplamente adotado no mercado (HMAC-SHA256) reduz a barreira de integração para os clientes B2B.

**Negativas:**
- Aumenta a complexidade do modelo de dados: é necessário armazenar e gerenciar potencialmente duas secrets ativas por endpoint durante o período de rotação.
- A responsabilidade de validar a assinatura corretamente recai sobre o cliente; implementações client-side incorretas (ex. comparação não constante-no-tempo) estão fora do controle da plataforma.
- Exige rota e fluxo de API adicionais (rotação) que precisam de revisão de segurança dedicada antes do deploy, conforme solicitado por Sofia ([09:46] Sofia: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança").
