# ADR-003: Retry com backoff exponencial e Dead Letter Queue (DLQ)

## Status

Aceito

## Contexto

Clientes externos podem estar temporariamente indisponíveis quando um evento é processado pelo worker. Larissa levantou a pergunta de comportamento nesse cenário ([09:14]-[09:15]). Diego propôs backoff exponencial com número máximo de tentativas.

Houve debate sobre a quantidade de tentativas: Bruno sugeriu 3 (mais agressivo), mas Diego contrapôs citando um precedente real de um cliente com 2 horas de manutenção planejada, que teria esgotado 3 tentativas em 30 minutos ([09:16]). Larissa decidiu por 5 tentativas, com a seguinte progressão de backoff: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas — cobrindo aproximadamente 15 horas entre a primeira falha e a última tentativa ([09:16]-[09:17]). Marcos validou o prazo do ponto de vista de negócio: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17]).

Restava decidir o que acontece após esgotar as tentativas. Larissa perguntou se a falha definitiva vira apenas um status no próprio outbox ou uma estrutura separada ([09:17]-[09:18]). Diego optou por uma tabela dedicada `webhook_dead_letter`, guardando payload, motivo da falha e timestamp, mantendo a leitura do outbox limpa e criando uma trilha de evidência/reprocessamento ([09:18]).

Bruno perguntou quem reprocessa esses eventos; Diego definiu um endpoint administrativo, `POST /admin/webhooks/dead-letter/:id/replay`, que re-enfileira o evento como pendente no outbox ([09:18]-[09:19]).

## Decisão

Após falha de entrega, o worker reagenda o evento com backoff exponencial de 5 tentativas: 1m, 5m, 30m, 2h, 12h. Esgotadas as 5 tentativas, o evento é movido para a tabela `webhook_dead_letter` (payload, motivo da falha, timestamp), saindo do fluxo normal do outbox.

Reprocessamento de eventos em DLQ é manual, via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a usuários com role `ADMIN` (reutilizando `requireRole` de `src/middlewares/auth.middleware.ts`), com log de auditoria de quem executou o replay ([09:35]-[09:36] Sofia/Larissa).

## Alternativas Consideradas

- **3 tentativas de retry**: descartado por ser agressivo demais — esgotaria o backoff em cerca de 30 minutos, insuficiente para cobrir janelas reais de manutenção de clientes ([09:16] Bruno/Diego).
- **Retries indefinidos**: mencionado por Diego como algo que "algumas pessoas defendem", mas descartado por risco de eventos ficarem pendentes indefinidamente caso o cliente nunca volte a responder ([09:15]).
- **Marcar falha definitiva como um status no próprio `webhook_outbox`** em vez de tabela separada: descartado em favor de uma tabela dedicada, por manter a leitura do outbox mais simples e oferecer melhor trilha de auditoria/reprocessamento ([09:17]-[09:18] Diego).

## Consequências

**Positivas:**
- Tolerância a indisponibilidades temporárias de clientes, incluindo janelas de manutenção de horas, sem intervenção manual.
- Separação clara entre eventos "em andamento" (outbox) e "falha definitiva" (DLQ), facilitando observabilidade e queries.
- Reprocessamento auditável e restrito a administradores reduz risco operacional.

**Negativas (trade-off explícito):**
- Eventos podem demorar até ~15 horas para serem definitivamente marcados como falha, o que atrasa a visibilidade do problema para o time de suporte se não houver alerta específico.
- Reprocessamento manual via endpoint administrativo não escala para grandes volumes de eventos em DLQ simultâneos — aceitável na fase atual, mas é um risco a monitorar.
