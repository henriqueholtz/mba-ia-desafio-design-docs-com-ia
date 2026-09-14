# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos de design (PRD, RFC, FDD, ADRs) à sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte (`CODIGO`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Restrição | OMS não tem mecanismo de notificação externa hoje | CODIGO | src/modules (ausência de módulo webhooks) |
| PRD-PROB-01 | docs/PRD.md | Requisito Funcional | Clientes B2B solicitaram notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Restrição | Atlas Comercial ameaça migrar de fornecedor se prazo não for cumprido | TRANSCRICAO | [09:01] Marcos |
| PRD-ESC-01 | docs/PRD.md | Decisão | Escopo é somente outbound (não bidirecional) | TRANSCRICAO | [09:02] Sofia / [09:03] Marcos |
| PRD-MET-01 | docs/PRD.md | Requisito Não Funcional | Meta de latência: até 10s para 95% dos eventos | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL + statuses de interesse) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerado automaticamente, retornado só na criação | TRANSCRICAO | [09:31] Marcos / [09:21] Sofia |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição (PATCH) de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção (DELETE) de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem (GET) de webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtragem por status de interesse na inserção do outbox | TRANSCRICAO | [09:33]-[09:34] Bruno/Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial | TRANSCRICAO | [09:14]-[09:17] |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Movimentação para DLQ após esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a ADMIN com auditoria | TRANSCRICAO | [09:18]-[09:19] / [09:35]-[09:36] |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 em cada entrega | TRANSCRICAO | [09:19]-[09:20] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21]-[09:22] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | X-Event-Id estável para deduplicação | TRANSCRICAO | [09:24]-[09:26] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Rejeição de URLs não-HTTPS | TRANSCRICAO | [09:22]-[09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, erro em vez de truncar | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once | TRANSCRICAO | [09:24]-[09:26] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Auditoria de replay de DLQ | TRANSCRICAO | [09:35]-[09:36] Sofia |
| PRD-ESCOPO-FORA-01 | docs/PRD.md | Restrição | Notificação por e-mail em falhas recorrentes — fora de escopo | TRANSCRICAO | [09:37]-[09:38] Larissa |
| PRD-ESCOPO-FORA-02 | docs/PRD.md | Restrição | Painel visual para clientes — fora de escopo | TRANSCRICAO | [09:39]-[09:40] Larissa |
| PRD-ESCOPO-FORA-03 | docs/PRD.md | Restrição | Rate limiting de saída — ponto em aberto, não implementado | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| PRD-ESCOPO-FORA-04 | docs/PRD.md | Restrição | Arquivamento de eventos antigos do outbox — fora de escopo | TRANSCRICAO | [09:07]-[09:08] Diego |
| PRD-ESCOPO-FORA-05 | docs/PRD.md | Restrição | Ordenação global entre múltiplos workers — fora de escopo | TRANSCRICAO | [09:12]-[09:14] |
| PRD-RISK-01 | docs/PRD.md | Trade-off | Risco de vazamento de secret HMAC | TRANSCRICAO | [09:21]-[09:22] / [09:45]-[09:47] |
| PRD-RISK-02 | docs/PRD.md | Trade-off | Risco de outage prolongado do cliente esgotar retries | TRANSCRICAO | [09:16]-[09:19] |
| PRD-DEP-01 | docs/PRD.md | Decisão | Dependência de OrderService.changeStatus como ponto único de disparo | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-01 | docs/RFC.md | Requisito Funcional | Clientes fazem polling hoje em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão | Proposta: Outbox + worker + retry/DLQ + HMAC + at-least-once | TRANSCRICAO | [09:03]-[09:26] |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Dispatch síncrono descartado (bloqueio de transação) | TRANSCRICAO | [09:03]-[09:06] Bruno/Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado (overengineering) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Triggers de banco descartados (sem NOTIFY/LISTEN no MySQL) | TRANSCRICAO | [09:09] Bruno/Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado (complexidade desproporcional) | TRANSCRICAO | [09:25] Diego/Sofia |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de saída | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: notificação de falha por e-mail | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Questão em aberto: escala do worker além de uma instância | TRANSCRICAO | [09:12]-[09:14] Diego |
| RFC-RISK-01 | docs/RFC.md | Trade-off | Risco: nova responsabilidade dentro da transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-RISK-02 | docs/RFC.md | Trade-off | Cronograma de 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:45]-[09:47] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Fluxo de criação do evento dentro da transação changeStatus | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Função pura publishWebhookEvent(tx, ...) | TRANSCRICAO | [09:41] Bruno/Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Worker processa lote ordenado por created_at | TRANSCRICAO | [09:12]-[09:13] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /webhooks — registro | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18]-[09:19] / [09:35] |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21]-[09:22] Sofia |
| FDD-ERR-01 | docs/FDD.md | Restrição | Código WEBHOOK_INVALID_URL (HTTPS obrigatório) | TRANSCRICAO | [09:22]-[09:23] Sofia |
| FDD-ERR-02 | docs/FDD.md | Restrição | Código WEBHOOK_PAYLOAD_TOO_LARGE (64KB) | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| FDD-RES-01 | docs/FDD.md | Requisito Não Funcional | Timeout HTTP de 10s por tentativa | TRANSCRICAO | [09:42] Diego/Sofia |
| FDD-RES-02 | docs/FDD.md | Requisito Não Funcional | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:16]-[09:17] Larissa/Diego |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Integração com order.service.ts (changeStatus) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Decisão | Reuso do enum OrderStatus em order.status.ts | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEG-03 | docs/FDD.md | Decisão | Extensão de AppError/http-errors.ts com códigos WEBHOOK_* | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | Reuso de requireRole em auth.middleware.ts | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Decisão | Middleware de erro não modificado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | Reuso do logger Pino | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Extensão aditiva do schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INTEG-08 | docs/FDD.md | Decisão | src/worker.ts espelha padrão de src/server.ts | CODIGO | src/server.ts |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Payload como snapshot, sem itens do pedido | TRANSCRICAO | [09:43] Diego |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44]-[09:45] Diego/Sofia |
| FDD-DADOS-03 | docs/FDD.md | Decisão | UUID como chave primária das novas tabelas | TRANSCRICAO | [09:51] Larissa/Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Padrão Outbox no MySQL | TRANSCRICAO | [09:03]-[09:08] |
| ADR-002 | docs/adrs/ADR-002-worker-dedicado-em-polling.md | Decisão | Worker dedicado, polling de 2s | TRANSCRICAO | [09:08]-[09:13] |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff + DLQ | TRANSCRICAO | [09:14]-[09:19] |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:19]-[09:24] |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24]-[09:26] |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso dos padrões existentes do projeto | TRANSCRICAO | [09:27]-[09:30] |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Referência a AppError, http-errors.ts, order.service.ts | CODIGO | src/shared/errors/app-error.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md | Decisão | Snapshot do payload na inserção, não no envio | TRANSCRICAO | [09:51]-[09:52] |
| CODIGO-ORDER-STATUS | docs/FDD.md | Restrição | Estados válidos: PENDING/PAID/PROCESSING/SHIPPED/DELIVERED/CANCELLED | CODIGO | prisma/schema.prisma |
| CODIGO-ORDER-TX | docs/FDD.md | Restrição | changeStatus já roda em transação Prisma (linhas 126-179) | CODIGO | src/modules/orders/order.service.ts |
| CODIGO-ERROR-PATTERN | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | InsufficientStockError/InvalidStatusTransitionError como modelo | CODIGO | src/shared/errors/http-errors.ts |
| CODIGO-PAGINACAO | docs/FDD.md | Restrição | Formato de paginação data/pagination reutilizado | CODIGO | src/shared/http/response.ts |
