# ADR-004: Garantia at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dado o modelo de outbox + worker com retry (ADR-001, ADR-002), é possível que um evento seja entregue mais de uma vez ao mesmo endpoint — por exemplo, se o worker processa o envio, o cliente recebe e processa com sucesso, mas a confirmação (resposta HTTP) se perde antes do worker marcá-lo como entregue, levando a uma nova tentativa.

Diego colocou explicitamente que a plataforma vai garantir **at-least-once**, não exactly-once: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" ([09:24] Diego). Para permitir que o cliente identifique duplicatas, cada evento recebe um identificador único — um UUID gerado no momento em que o evento entra na outbox — enviado no header `X-Event-Id` ([09:25] Diego). Sofia observou que isso transfere a responsabilidade de deduplicação para o lado do cliente ([09:25] Sofia), ao que Diego respondeu que esse é o padrão adotado por players de mercado como Stripe e GitHub, e que garantir exactly-once exigiria coordenação entre os dois lados, aumentando substancialmente a complexidade para resolver um problema que o event_id já resolve "em 99% dos casos" ([09:25] Diego). Marcos se comprometeu a documentar esse comportamento de forma destacada no portal do desenvolvedor ([09:26] Marcos).

## Decisão

- A plataforma garante semântica de entrega **at-least-once**: um evento pode, em cenários de falha de confirmação, ser entregue mais de uma vez ao mesmo endpoint.
- Cada evento recebe um **UUID único (`event_id`)**, gerado no momento da inserção na outbox, e esse mesmo id é enviado em toda tentativa de entrega (inclusive em retries) no header **`X-Event-Id`**.
- A responsabilidade de deduplicação do lado do recebimento é do cliente, que deve usar o `X-Event-Id` para detectar e ignorar entregas repetidas.
- Este comportamento deve ser documentado de forma explícita e destacada na documentação voltada a clientes (portal do desenvolvedor), fora do escopo dos design docs técnicos desta feature.

## Alternativas Consideradas

- **Garantia exactly-once:** eliminaria a necessidade de deduplicação no cliente, mas exigiria mecanismos de coordenação transacional entre plataforma e cliente (ex. confirmação transacional bidirecional), com complexidade de implementação e operação significativamente maior. Descartada por custo/benefício desfavorável ([09:25] Diego).
- **At-most-once (sem retry em caso de dúvida de entrega):** evitaria duplicatas, mas arriscaria perder eventos legítimos sempre que houvesse incerteza sobre o sucesso da entrega — inaceitável dado o requisito de notificação confiável dos clientes B2B. Não foi proposta como alternativa viável na reunião, mas é a consequência natural de não aceitar at-least-once.

## Consequências

**Positivas:**
- Modelo simples e alinhado a padrões consolidados de mercado (Stripe, GitHub), reduzindo a curva de aprendizado dos clientes B2B na integração.
- Evita a complexidade de coordenação distribuída necessária para exactly-once.
- O `X-Event-Id` também serve para correlação em logs e no histórico de entregas (`GET /webhooks/:id/deliveries`).

**Negativas:**
- Transfere trabalho de implementação (deduplicação) para os sistemas dos clientes; integrações mal implementadas do lado do cliente podem processar efeitos colaterais duplicados (ex. atualizar status duas vezes).
- Requer comunicação e documentação externa cuidadosa para que os clientes entendam e implementem a deduplicação corretamente — risco de suporte caso não fique claro.
