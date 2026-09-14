# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

- **Autor:** Time de Engenharia (Orders + Platform)
- **Status:** Em revisão
- **Data:** 2026-09-14 (baseado na reunião técnica realizada em [09:00])
- **Revisores:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Orders), Diego (Eng. Sênior, Platform), Sofia (Security Engineer)

## Resumo Executivo (TL;DR)

Propomos um sistema de webhooks outbound para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) em até 10 segundos sempre que o status de um pedido mudar, eliminando o polling atual em `GET /orders`. A solução usa o padrão **Outbox** (inserção do evento na mesma transação MySQL que muda o status do pedido), um **worker dedicado em processo separado** fazendo polling a cada 2 segundos, **retry com backoff exponencial e DLQ** para tolerar indisponibilidade de clientes, e **autenticação HMAC-SHA256 com secret por endpoint** para garantir autenticidade. A entrega é **at-least-once**, com deduplicação do lado do cliente via `X-Event-Id`. Toda a solução reaproveita os padrões já existentes no projeto (módulos, `AppError`, Pino, middleware de erro).

## Contexto e Problema

Três clientes B2B do OMS (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje monitoram o status de seus pedidos fazendo polling repetido em `GET /orders`, o que é ineficiente e caro para as integrações deles. Atlas Comercial sinalizou risco de migrar para um concorrente caso a notificação em tempo real não seja entregue até o fim do trimestre ([09:00]-[09:01] Marcos). "Tempo real" foi definido como latência aceitável abaixo de 10 segundos — o requisito central é eliminar o polling manual, não latência sub-segundo ([09:02] Marcos).

O OMS atualmente não possui nenhum mecanismo de notificação externa, eventos ou filas — este é um domínio novo dentro da aplicação (`src/modules/orders/` hoje só expõe CRUD e mudança de status síncrona via `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`, linhas 126-179).

O escopo é exclusivamente outbound: o OMS envia notificações para os clientes; não há webhooks de entrada (bidirecional foi descartado logo no início, [09:02]-[09:03] Sofia/Marcos).

## Proposta Técnica

A solução se apoia em cinco pilares, cada um formalizado em um ADR (ver seção "Decisões Relacionadas"):

1. **Outbox transacional**: a mudança de status do pedido (`OrderService.changeStatus`) passa a, dentro da mesma transação já existente, inserir um evento na nova tabela `webhook_outbox`. Isso garante que status e evento nascem ou morrem juntos — nunca há mudança de status sem evento correspondente, nem evento "fantasma" de uma transação que sofreu rollback.

2. **Worker assíncrono dedicado**: um processo Node.js separado (`src/worker.ts`, novo entry point análogo a `src/server.ts`) faz polling na tabela `webhook_outbox` a cada 2 segundos, processa eventos pendentes em lote e realiza a entrega HTTP para a URL configurada pelo cliente.

3. **Resiliência via retry + DLQ**: falhas de entrega (timeout, erro HTTP, indisponibilidade) acionam retry com backoff exponencial (1m, 5m, 30m, 2h, 12h — 5 tentativas). Esgotadas as tentativas, o evento migra para uma tabela `webhook_dead_letter`, de onde pode ser reprocessado manualmente por um administrador via endpoint dedicado.

4. **Segurança via HMAC-SHA256**: cada endpoint de webhook cadastrado por um cliente recebe um secret único, usado para assinar o corpo de cada requisição (header `X-Signature`). URLs devem ser HTTPS. Secrets são rotacionáveis, com período de graça de 24h. Payloads são limitados a 64KB.

5. **Entrega at-least-once com deduplicação**: cada evento carrega um `X-Event-Id` (UUID) estável entre tentativas, permitindo que o cliente deduplique entregas repetidas — mesmo modelo adotado por Stripe e GitHub.

O módulo é implementado em `src/modules/webhooks/`, seguindo a mesma estrutura de controller/service/repository/routes/schemas dos módulos já existentes (`src/modules/orders/`, `src/modules/customers/`), reutilizando `AppError` (com códigos `WEBHOOK_*`), o logger Pino e o middleware de erro centralizado sem modificações.

O detalhamento de contratos HTTP, matriz de erros, fluxos passo a passo e integração ponto a ponto com o código existente está no **FDD** (`docs/FDD.md`) — este RFC propositalmente não desce a esse nível.

## Alternativas Consideradas

### 1. Dispatch síncrono de webhook dentro da transação de `changeStatus`

Disparar a chamada HTTP ao cliente diretamente na transação que muda o status do pedido, sem outbox nem worker.

**Trade-off que motivou o descarte:** um cliente lento ou fora do ar bloquearia a transação de status para pedidos completamente não relacionados, e não havia estratégia de rollback sensata para o caso de falha na chamada HTTP no meio de uma transação de banco ([09:03]-[09:06] Bruno/Diego). Formalizado em [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md).

### 2. Fila externa dedicada (Redis Streams)

Usar uma fila de mensageria externa, como Redis Streams, em vez de uma tabela outbox no próprio MySQL.

**Trade-off que motivou o descarte:** exigiria subir e operar infraestrutura nova (ex.: Redis Cluster) para um time pequeno, quando o MySQL já existente resolve o problema com uma tabela adicional — considerado overengineering para o estágio atual do produto ([09:07] Diego). Formalizado em [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md).

### 3. Worker reativo via triggers de banco de dados

Usar triggers do MySQL para acionar o processamento de eventos de forma reativa, em vez de polling periódico.

**Trade-off que motivou o descarte:** MySQL não oferece um mecanismo equivalente ao NOTIFY/LISTEN do PostgreSQL; triggers só executam SQL e não conseguem notificar um processo externo. Workarounds cogitados (escrever em arquivo, chamar endpoint a partir do trigger) foram considerados frágeis e inadequados. O polling de 2s já atende com folga o SLA de 10s aceito pelos clientes ([09:09] Bruno/Diego). Formalizado em [ADR-002](adrs/ADR-002-worker-dedicado-em-polling.md).

### 4. Garantia de entrega exactly-once

Garantir que cada evento seja entregue exatamente uma vez, eliminando a necessidade de deduplicação do lado do cliente.

**Trade-off que motivou o descarte:** exigiria coordenação bidirecional (confirmação transacional de recebimento) com complexidade desproporcional ao ganho; at-least-once combinado com `X-Event-Id` resolve a "esmagadora maioria dos casos práticos", seguindo o mesmo padrão adotado por Stripe e GitHub ([09:25] Diego/Sofia). Formalizado em [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

## Questões em Aberto

1. **Rate limiting de saída para clientes**: Diego levantou o risco de um cliente ser "estourado" com muitas chamadas em pouco tempo (ex.: 50 mudanças de status em um minuto no mesmo pedido/cliente). Não foi tomada uma decisão de implementar rate limiting nesta fase — a orientação de Larissa foi "observar e decidir depois" ([09:38]-[09:39]). Precisa ser monitorado em produção e revisitado se virar problema real.

2. **Notificação de falha recorrente ao cliente (e-mail)**: Marcos perguntou se o cliente deveria ser avisado por e-mail após falhas consecutivas de entrega (ex.: 3 falhas seguidas). Larissa determinou que está fora de escopo desta fase, a ser avaliado em fase futura após medição de impacto ([09:37]-[09:38]). Não há decisão sobre o mecanismo (e-mail, painel, outro canal).

3. **Escala do worker além de uma única instância**: a ordenação de eventos por `order_id` só é garantida com um único worker. Diego mencionou partição por `order_id` ou locking pessimista como possíveis soluções futuras, mas nenhuma foi decidida — "problema do futuro, não agora" ([09:13]).

## Impacto e Riscos

- **Impacto no fluxo crítico de pedidos**: `OrderService.changeStatus` passa a ter uma responsabilidade adicional (inserir no outbox) dentro de sua transação existente. Erro na lógica de inserção do evento pode, em tese, impactar a operação principal de mudança de status — mitigado por ser uma função pura (`publishWebhookEvent(tx, ...)`) chamada dentro da transação já existente, sem I/O externo.
- **Nova superfície de dados sensíveis**: secrets HMAC por cliente precisam de geração, armazenamento e rotação seguros — Sofia solicitou pelo menos 2 dias úteis de revisão de segurança dedicada antes do deploy, com foco em HMAC e geração de secrets ([09:45]-[09:47]).
- **Operação de um processo adicional**: o worker precisa ser deployado, monitorado e alertado separadamente da API — aumenta a superfície operacional.
- **Cronograma**: estimativa de Larissa é de 3 sprints, incluindo a janela de revisão de segurança da Sofia, para atender o prazo de fim de novembro solicitado pela Atlas Comercial ([09:45]-[09:47]).

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002: Worker dedicado em polling](adrs/ADR-002-worker-dedicado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004: Autenticação HMAC-SHA256 por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)
- [ADR-005: Entrega at-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007: Snapshot do payload na inserção do evento](adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md)
