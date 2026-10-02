# ADR-002: Retry com backoff exponencial e Dead Letter Queue em tabela separada

## Status

Aceito

## Contexto

Clientes de webhook podem estar temporariamente indisponíveis quando um evento é enviado. A equipe precisava decidir quantas tentativas realizar, com qual espaçamento, e o que fazer quando todas as tentativas se esgotam.

Na reunião, Diego propôs backoff exponencial com um teto de tentativas, após o qual o evento é considerado falha permanente ([09:15] Diego). Foi discutido o número de tentativas: Bruno sugeriu 3 ("mais agressivo", [09:16] Bruno), mas Diego argumentou que 3 tentativas esgotariam em cerca de 30 minutos, insuficiente para cobrir janelas de manutenção planejada de clientes (citou um caso real de indisponibilidade de duas horas, [09:16] Diego). Retry indefinido também foi rejeitado por deixar eventos "pendurados para sempre" ([09:15] Diego). A equipe fechou em 5 tentativas com progressão 1 min, 5 min, 30 min, 2 h, 12 h, totalizando quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego, confirmado por Marcos e Larissa).

Também foi decidido onde registrar falhas permanentes: Diego propôs uma tabela `webhook_dead_letter` separada da outbox principal, guardando payload, motivo da falha e timestamp, por ser "mais limpa a leitura da outbox principal" e servir de evidência para debug e reprocessamento ([09:18] Diego). O reprocessamento é manual, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).

## Decisão

- Política de retry: **5 tentativas no total**, com backoff exponencial de **1 min, 5 min, 30 min, 2 h e 12 h** entre tentativas sucessivas.
- Após a 5ª tentativa sem sucesso, o evento é movido para uma tabela dedicada **`webhook_dead_letter`**, contendo o payload original, o motivo da última falha e o timestamp.
- Reprocessamento de itens da DLQ é **manual**, via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito a usuários com role `ADMIN` (ver ADR-006 para reuso do `requireRole`), que recoloca o evento na outbox com status pendente.
- O endpoint de replay deve registrar em log (auditoria) qual usuário administrador executou o reprocessamento ([09:36] Sofia).

## Alternativas Consideradas

- **3 tentativas de retry:** mais agressivo e rápido para liberar recursos, mas insuficiente para cobrir indisponibilidades de clientes por algumas horas (ex. manutenção planejada), resultando em perda de notificações legítimas. Descartada ([09:16] Bruno/Diego).
- **Retry indefinido:** garantiria entrega eventual sem limite de tentativas, mas mantém eventos "pendurados" indefinidamente quando o cliente está permanentemente fora do ar, sem sinalização clara de falha. Descartada ([09:15] Diego).
- **Marcar falha permanente como status na própria tabela de outbox** (sem tabela separada): mais simples, porém mistura o fluxo operacional "quente" (pendentes/processando) com o estado terminal de falha, dificultando a leitura e auditoria da fila ativa. Descartada em favor de tabela separada ([09:18] Diego/Bruno).

## Consequências

**Positivas:**
- Cobre janelas de indisponibilidade de clientes de até ~15 horas sem perder eventos, reduzindo falsos positivos de "cliente não recebeu".
- Separar a DLQ da outbox principal mantém a tabela operacional enxuta e facilita queries de monitoramento sobre pendências reais.
- Reprocessamento manual com auditoria dá controle explícito sobre reenvios, evitando reenvio acidental em massa.

**Negativas:**
- Eventos que falham definitivamente exigem intervenção manual de um administrador; não há fallback automático (ex. e-mail) nesta fase — ponto adiado para fase futura ([09:37] Marcos/Larissa).
- A progressão de backoff de até 12 horas significa que, no pior caso, um cliente só é considerado "em DLQ" quase 15 horas após a primeira falha, o que pode atrasar a percepção de um problema real de integração do lado do cliente.
