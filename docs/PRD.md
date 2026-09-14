# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O OMS (Order Management System) hoje não possui nenhum mecanismo de notificação externa: clientes B2B que precisam acompanhar o status de seus pedidos são obrigados a fazer polling repetido em `GET /orders`. Esta feature introduz um sistema de webhooks outbound que notifica clientes automaticamente, via HTTP, sempre que o status de um pedido muda, eliminando a necessidade de polling.

## Problema e Motivação

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) formalizaram a solicitação de notificações em tempo real sobre o status de seus pedidos junto ao time de produto ([09:00]-[09:01] Marcos). O modelo atual de polling é ineficiente e caro para as integrações desses clientes. Atlas Comercial sinalizou risco concreto de churn — ameaça de migrar para um concorrente — caso a funcionalidade não seja entregue até o fim do trimestre ([09:01] Marcos).

## Público-Alvo e Cenários de Uso

- **Clientes B2B integradores** (ex.: Atlas Comercial, MaxDistribuição, Nova Cargo): sistemas externos que consomem a API do OMS e precisam reagir a mudanças de status de pedido (ex.: pedido pago, enviado, entregue) sem consultar a API repetidamente.
- **Administradores do OMS** (role `ADMIN`): responsáveis por reprocessar manualmente eventos que falharam definitivamente (DLQ) ([09:18]-[09:19], [09:35]-[09:36]).
- **Usuários autenticados do OMS** (qualquer role, por enquanto): podem cadastrar, editar e consultar webhooks em nome de um cliente através da API já autenticada por JWT ([09:36]-[09:37] Sofia: "Por enquanto sim").

## Objetivos e Métricas de Sucesso

- **Eliminar o polling manual** como forma primária de os clientes acompanharem status de pedidos.
- **Latência de entrega:** notificar clientes em até 10 segundos após a mudança de status, para pelo menos 95% dos eventos — meta derivada diretamente do critério de aceite informal definido por Marcos ("tempo real" = abaixo de 10s, [09:02]) e da escolha de polling de 2s do worker, que oferece folga sobre essa meta.
- **Confiabilidade de entrega:** garantir entrega at-least-once, com no máximo 5 tentativas de retry e janela de até ~15h antes de um evento ser considerado falha definitiva (DLQ) ([09:16]-[09:17]).
- **Retenção do cliente Atlas Comercial:** entregar a feature até o fim de novembro, prazo comunicado por Marcos como crítico para retenção do cliente ([09:45]-[09:47] Larissa).

## Escopo

### Incluso

- Cadastro, edição, listagem e remoção de webhooks por cliente (CRUD completo) ([09:31]-[09:33]).
- Filtragem por status de interesse por endpoint (ex.: só notificar em `SHIPPED` e `DELIVERED`) ([09:33]-[09:34]).
- Entrega assíncrona via padrão Outbox + worker dedicado em polling ([09:03]-[09:13]).
- Retry com backoff exponencial e Dead Letter Queue (DLQ) para falhas persistentes ([09:14]-[09:19]).
- Reprocessamento manual de eventos em DLQ, restrito a administradores ([09:18]-[09:19], [09:35]-[09:36]).
- Histórico de entregas por webhook (últimas 100), consultável pelo cliente ([09:34]).
- Autenticação de payload via HMAC-SHA256, secret único por endpoint, com rotação e grace period de 24h ([09:19]-[09:22]).
- Entrega at-least-once com deduplicação via `X-Event-Id` do lado do cliente ([09:24]-[09:26]).
- TLS obrigatório para URLs de webhook e limite de 64KB por payload ([09:22]-[09:24]).

### Fora de escopo

- **Notificação por e-mail em falhas recorrentes de entrega**: descartada explicitamente para esta fase por Larissa — "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" ([09:37]-[09:38]).
- **Painel visual (dashboard) para clientes gerenciarem seus webhooks**: descartado — "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" ([09:39]-[09:40] Larissa).
- **Rate limiting de saída para os clientes**: não implementado nesta fase; ponto explicitamente deixado em aberto para observação futura ([09:38]-[09:39] Diego/Larissa).
- **Arquivamento/purga de eventos antigos do outbox**: mencionado como necessário no futuro, mas fora do escopo desta feature ([09:07]-[09:08] Diego).
- **Garantia de ordenação global de eventos entre múltiplos workers**: válido apenas para um único worker nesta fase; escalar horizontalmente exigiria decisão futura sobre particionamento ([09:12]-[09:14]).
- **Webhooks de entrada (bidirecionais)**: escopo é estritamente outbound — "Só saindo da gente pra eles" ([09:02]-[09:03] Marcos/Sofia).

## Requisitos Funcionais

1. O sistema deve permitir que um usuário autenticado cadastre um webhook para um cliente, informando URL e lista de status de interesse ([09:31] Marcos).
2. O sistema deve gerar automaticamente um secret para cada webhook cadastrado, retornado apenas no momento da criação ([09:31] Marcos, [09:21] Sofia).
3. O sistema deve permitir editar (`PATCH`) um webhook existente (URL, status de interesse, ativo/inativo) ([09:33] Bruno).
4. O sistema deve permitir remover (`DELETE`) um webhook ([09:33] Bruno).
5. O sistema deve permitir listar (`GET`) os webhooks cadastrados de um cliente ([09:33] Bruno).
6. O sistema deve filtrar o envio de eventos pelos status de interesse configurados em cada webhook, não inserindo evento no outbox quando nenhum webhook do cliente estiver interessado no novo status ([09:33]-[09:34] Bruno/Diego).
7. O sistema deve expor o histórico das últimas 100 entregas de um webhook, incluindo sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).
8. O sistema deve reenviar automaticamente eventos que falharem, seguindo backoff exponencial de 5 tentativas (1m, 5m, 30m, 2h, 12h) ([09:14]-[09:17]).
9. O sistema deve mover eventos que esgotarem as tentativas de retry para uma fila de falhas definitivas (Dead Letter Queue) ([09:18] Diego).
10. O sistema deve permitir que um administrador reprocesse manualmente um evento em DLQ via endpoint dedicado, restrito à role `ADMIN` e com log de auditoria de quem executou a ação ([09:18]-[09:19], [09:35]-[09:36]).
11. O sistema deve assinar cada notificação enviada com HMAC-SHA256, usando o secret do endpoint correspondente, incluído no header `X-Signature` ([09:19]-[09:20] Sofia).
12. O sistema deve permitir a rotação do secret de um webhook, mantendo o secret anterior válido por 24 horas após a rotação ([09:21]-[09:22] Sofia).
13. O sistema deve incluir um identificador único de evento (`X-Event-Id`) em toda tentativa de entrega, estável entre reentregas, para permitir deduplicação pelo cliente ([09:24]-[09:26] Diego).
14. O sistema deve rejeitar o cadastro de webhooks com URL que não utilize HTTPS ([09:22]-[09:23] Sofia).

## Requisitos Não Funcionais

- **Latência:** notificação entregue em até 10 segundos após a mudança de status, na grande maioria dos casos, via polling de 2s do worker ([09:02], [09:09]-[09:10]).
- **Segurança:** TLS obrigatório para URLs de webhook; assinatura HMAC-SHA256 por payload; secret exclusivo por endpoint (não global); rotação de secret com grace period de 24h ([09:19]-[09:24]).
- **Limite de payload:** eventos maiores que 64KB são rejeitados na origem (erro), não truncados ([09:23]-[09:24]).
- **Confiabilidade:** garantia de entrega at-least-once; consistência transacional entre mudança de status e criação do evento (padrão Outbox) ([09:03]-[09:08], [09:24]-[09:26]).
- **Auditabilidade:** toda ação de replay administrativo de DLQ deve ser registrada em log com o identificador do usuário que a executou ([09:35]-[09:36] Sofia).
- **Consistência de convenções:** o módulo deve seguir os padrões arquiteturais já estabelecidos no projeto (estrutura de módulos, tratamento de erros via `AppError`, logger Pino) ([09:27]-[09:30]).

## Decisões e Trade-offs Principais

Detalhadas individualmente nos ADRs (`docs/adrs/`). Resumo:

- **Outbox transacional em vez de dispatch síncrono ou fila externa**: prioriza consistência e simplicidade operacional sobre latência mínima teórica ([ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)).
- **Worker dedicado em polling de 2s em vez de mecanismo reativo**: aceita uma latência mínima (até 2s) em troca de não depender de infraestrutura ou recursos que o MySQL não oferece ([ADR-002](adrs/ADR-002-worker-dedicado-em-polling.md)).
- **5 tentativas de retry com backoff de até 12h, e não 3**: prioriza tolerância a outages reais de clientes sobre velocidade de detecção de falha definitiva ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
- **At-least-once em vez de exactly-once**: prioriza simplicidade de implementação, transferindo a responsabilidade de deduplicação ao cliente, seguindo padrão de mercado ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).

## Dependências

- Máquina de estados de pedidos já existente (`src/modules/orders/order.status.ts`) e o método `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que passam a ser o ponto único de disparo de eventos.
- Modelos `Customer` e `Order` do schema Prisma existente, para associar webhooks a clientes e montar o payload dos eventos.
- Middleware de autenticação e de autorização por role já existentes (`src/middlewares/auth.middleware.ts`).
- Disponibilidade da Sofia (Security Engineer) para revisão de segurança dedicada antes do deploy — pelo menos 2 dias úteis reservados ([09:45]-[09:47]).

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Vazamento de secret HMAC de um cliente compromete a autenticidade das notificações | Média | Alto | Secret exclusivo por endpoint (não global), suporte a rotação com grace period, revisão de segurança dedicada antes do deploy ([09:21]-[09:22], [09:45]-[09:47]) |
| Cliente com outage prolongado esgota as tentativas de retry e perde eventos críticos | Média | Médio | Janela de retry de ~15h (5 tentativas) cobre a maioria das janelas de manutenção reais; eventos não são descartados, vão para DLQ e podem ser reprocessados manualmente ([09:16]-[09:19]) |
| Outbox cresce sem controle e degrada performance de leitura do worker | Baixa (no prazo desta feature) | Médio | Índices em status/timestamp, limite de 64KB por payload; arquivamento identificado como dívida técnica para fase futura ([09:07]-[09:08]) |
| Prazo de fim de novembro não é cumprido, gerando risco de churn do cliente Atlas Comercial | Baixa a Média | Alto | Estimativa de 3 sprints já inclui a janela de revisão de segurança da Sofia; escopo foi deliberadamente reduzido (sem e-mail, sem painel, sem rate limiting) para viabilizar o prazo ([09:45]-[09:47]) |

## Critérios de Aceitação

- Um cliente com webhook ativo para o status `SHIPPED` recebe uma notificação HTTP assinada em até ~10 segundos após um pedido mudar para `SHIPPED`.
- Uma tentativa de entrega falha é reenviada automaticamente seguindo o cronograma de backoff, sem intervenção manual.
- Após 5 falhas consecutivas, o evento aparece na Dead Letter Queue e pode ser reprocessado por um administrador.
- Rotação de secret não interrompe a validação de assinaturas por 24 horas após a troca.
- Nenhum evento é inserido no outbox para um status que nenhum webhook ativo do cliente tenha assinado.
- Um usuário sem role `ADMIN` recebe erro de autorização ao tentar reprocessar eventos em DLQ.

## Estratégia de Testes e Validação

- **Testes de integração** cobrindo o fluxo completo de `changeStatus` → inserção no outbox → processamento pelo worker → entrega HTTP (mockada), incluindo cenário de rollback (evento não deve ser criado se a transação falhar).
- **Testes unitários** da lógica de backoff (cálculo de `nextAttemptAt` por tentativa) e da geração/validação de assinatura HMAC-SHA256.
- **Testes de contrato** dos endpoints HTTP (CRUD de webhooks, listagem de entregas, replay de DLQ), validando status codes e payloads de erro (`WEBHOOK_*`).
- **Revisão de segurança dedicada** da Sofia antes do deploy, com foco em geração/armazenamento/rotação de secrets ([09:45]-[09:47]).
- **Validação manual** do fluxo ponta a ponta com um endpoint de teste (ex.: webhook.site ou serviço equivalente) simulando o cliente externo antes do go-live com Atlas Comercial.
