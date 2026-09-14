# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

O RFC (`docs/RFC.md`) definiu a arquitetura de alto nível: padrão Outbox, worker dedicado em polling, retry com DLQ, autenticação HMAC-SHA256 e entrega at-least-once. Este documento especifica **como implementar** cada peça, em nível suficiente para um desenvolvedor começar a codar sem decisões de design pendentes.

O ponto de integração central é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179), que já executa, dentro de uma transação Prisma (`this.prisma.$transaction`): busca do pedido, validação de transição via `canTransition` (`src/modules/orders/order.status.ts`), débito/reposição de estoque, `tx.order.update()` e `tx.orderStatusHistory.create()`. A feature adiciona um passo a essa transação: inserir um evento em `webhook_outbox`.

## Objetivos Técnicos

- Garantir que todo evento de mudança de status gere, de forma transacional, uma linha em `webhook_outbox` (nunca perder ou duplicar na origem).
- Entregar eventos a clientes externos com latência ponta a ponta abaixo de 10 segundos na grande maioria dos casos.
- Tolerar indisponibilidade temporária de clientes (até ~15h) sem perda de eventos, via retry + DLQ.
- Garantir autenticidade e integridade de cada entrega via HMAC-SHA256.
- Manter o módulo alinhado 100% às convenções de código já existentes no projeto (módulos, erros, logger).

## Escopo e Exclusões

**Dentro do escopo:**
- Modelagem de `webhook_outbox`, `webhook_dead_letter` e tabela de configuração de webhooks (`webhook_endpoints`).
- Extensão de `OrderService.changeStatus` para publicar eventos.
- Worker dedicado (`src/worker.ts`) com polling, retry/backoff e movimentação para DLQ.
- Endpoints HTTP de CRUD de configuração de webhook, listagem de entregas e replay administrativo de DLQ.
- Assinatura HMAC-SHA256, rotação de secret com grace period de 24h.

**Fora do escopo (ver PRD, seção "Fora de escopo", e RFC, seção "Questões em Aberto"):**
- Notificação por e-mail em falhas recorrentes de entrega ([09:37]-[09:38]).
- Rate limiting de saída para clientes ([09:38]-[09:39]).
- Painel visual para clientes gerenciarem webhooks ([09:39]-[09:40]).
- Arquivamento/purga de eventos antigos do outbox ([09:07]-[09:08]).
- Escala horizontal do worker (múltiplas instâncias) e garantias de ordenação global ([09:12]-[09:14]).

## Modelagem de Dados (novas tabelas)

Seguindo a convenção existente em `prisma/schema.prisma` (UUID em `Char(36)`, `@default(uuid())`, timestamps `createdAt`/`updatedAt`, `@@map` em snake_case):

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model WebhookEndpoint {
  id           String   @id @default(uuid()) @db.Char(36)
  customerId   String   @db.Char(36)
  url          String   @db.VarChar(500)
  secret       String   @db.VarChar(255)
  previousSecret String? @db.VarChar(255)
  secretRotatedAt DateTime?
  statuses     Json
  active       Boolean  @default(true)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  outboxItems WebhookOutbox[]

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36)
  webhookEndpointId String          @db.Char(36)
  orderId       String              @db.Char(36)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id])

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id              String   @id @default(uuid()) @db.Char(36)
  eventId         String   @db.Char(36)
  webhookEndpointId String @db.Char(36)
  payload         Json
  failureReason   String   @db.Text
  attempts        Int
  movedAt         DateTime @default(now())

  @@index([webhookEndpointId])
  @@map("webhook_dead_letter")
}
```

## Fluxos Detalhados

### 1. Criação do evento na Outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` executa sua transação atual (validação de transição, débito/reposição de estoque, `tx.order.update()`, `tx.orderStatusHistory.create()`) sem alterações.
2. Antes do commit, chama uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (nova, em `src/modules/webhooks/webhook.service.ts` ou módulo equivalente), recebendo o client de transação ativo (`tx`) em vez de um repository injetado ([09:41] Bruno/Diego).
3. `publishWebhookEvent`:
   a. Busca (via `tx`) os `WebhookEndpoint` ativos do `customerId` do pedido cujo array `statuses` contenha o novo `toStatus`. Se nenhum endpoint corresponder, não insere nada (otimização decidida em [09:33]-[09:34] Bruno/Diego: filtragem ocorre na inserção, não no envio).
   b. Para cada endpoint correspondente, monta o payload (snapshot completo, ver ADR-007) e insere uma linha em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
4. Se a inserção falhar, a transação inteira sofre rollback — mudança de status e evento nascem ou morrem juntos ([09:40]-[09:41] Bruno/Diego).

### 2. Processamento pelo Worker

1. `src/worker.ts` inicia um loop com `setInterval`/loop assíncrono de 2 segundos.
2. A cada iteração, busca um lote (ex.: até 50) de `WebhookOutbox` com `status IN (PENDING, FAILED)` e `nextAttemptAt <= now()`, ordenado por `createdAt` ascendente.
3. Marca cada item como `PROCESSING` (evita processamento duplicado se o worker escalar no futuro).
4. Para cada item: monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`), faz `POST` para a `url` do endpoint com timeout de 10 segundos.
5. Sucesso (2xx): marca `status = DELIVERED`, registra em log de entrega (ver seção Observabilidade).
6. Falha (timeout, erro de rede, status >= 400): incrementa `attempts`; se `attempts < 5`, marca `status = FAILED` e define `nextAttemptAt = now() + backoff[attempts]` (backoff: 1m, 5m, 30m, 2h, 12h); se `attempts >= 5`, move a linha para `WebhookDeadLetter` (copiando payload, motivo da falha, `attempts`) e remove/marca a linha original do outbox como finalizada.

### 3. Retry com Backoff

Tabela de backoff por tentativa (ADR-003):

| Tentativa | Delay até a próxima |
|---|---|
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 (última) | 12 horas |

Após a 5ª falha, o evento vai para DLQ.

### 4. Dead Letter Queue (DLQ) e Replay

- Eventos em `webhook_dead_letter` ficam disponíveis para consulta via `GET /admin/webhooks/dead-letter` (autenticado, role `ADMIN`).
- Replay manual: `POST /admin/webhooks/dead-letter/:id/replay` — recria uma linha em `webhook_outbox` com `status = PENDING`, `attempts = 0`, e remove (ou marca como reprocessada) a linha em `webhook_dead_letter`. Requer role `ADMIN` (reuso de `requireRole` de `src/middlewares/auth.middleware.ts`) e registra em log estruturado (Pino) o `userId` de quem executou o replay, para auditoria ([09:35]-[09:36] Sofia).

## Contratos Públicos

Todos os endpoints exigem autenticação via `authenticate` (JWT Bearer), reaproveitando `src/middlewares/auth.middleware.ts`. Respostas de listagem seguem o padrão de paginação existente em `src/shared/http/response.ts` (`{ data: [...], pagination: {...} }`).

### 1. `POST /webhooks` — Registrar um webhook

Cria uma configuração de webhook para um cliente. O secret é gerado pela plataforma e retornado **apenas nesta resposta**.

**Request:**
```json
{
  "customerId": "8f14e45f-ceea-4b7c-8b7b-3b5d7c5e21a1",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"]
}
```

**Response (201 Created):**
```json
{
  "id": "3e2f9a10-6d21-4e2a-9a11-2f0e8b6c9d10",
  "customerId": "8f14e45f-ceea-4b7c-8b7b-3b5d7c5e21a1",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "secret": "whsec_9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-09-14T13:00:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL` (URL não é HTTPS), `400 VALIDATION_ERROR` (statuses vazio/inválido), `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### 2. `GET /webhooks?customerId=...` — Listar webhooks de um cliente

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "3e2f9a10-6d21-4e2a-9a11-2f0e8b6c9d10",
      "customerId": "8f14e45f-ceea-4b7c-8b7b-3b5d7c5e21a1",
      "url": "https://api.atlascomercial.com.br/webhooks/oms",
      "statuses": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-14T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Nota: o `secret` nunca é retornado em `GET` — apenas na criação e na rotação.

### 3. `PATCH /webhooks/:id` — Atualizar configuração (URL, statuses, active)

**Request:**
```json
{
  "statuses": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true
}
```

**Response (200 OK):** mesmo formato do `GET`, com os campos atualizados.

Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 VALIDATION_ERROR`.

### 4. `DELETE /webhooks/:id` — Remover um webhook

**Response:** `204 No Content`.

Erros: `404 WEBHOOK_NOT_FOUND`.

### 5. `POST /webhooks/:id/rotate-secret` — Rotacionar secret

Gera um novo secret; o secret anterior permanece válido por 24 horas (ADR-004).

**Response (200 OK):**
```json
{
  "id": "3e2f9a10-6d21-4e2a-9a11-2f0e8b6c9d10",
  "secret": "whsec_1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d",
  "previousSecretValidUntil": "2026-09-15T13:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6. `GET /webhooks/:id/deliveries` — Histórico de entregas

Retorna as últimas 100 tentativas de entrega (sucesso/falha, payload, resposta, tempo de resposta) ([09:34] Marcos).

**Response (200 OK):**
```json
{
  "data": [
    {
      "eventId": "6c3a1e2d-9f8b-4c7a-8e1d-5b2c3a4d5e6f",
      "status": "DELIVERED",
      "attempts": 1,
      "httpStatus": 200,
      "responseTimeMs": 184,
      "deliveredAt": "2026-09-14T13:05:02.000Z"
    },
    {
      "eventId": "7d4b2f3e-0a9c-4d8b-9f2e-6c3d4e5f6a7b",
      "status": "FAILED",
      "attempts": 2,
      "httpStatus": 503,
      "responseTimeMs": 10000,
      "deliveredAt": null
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

### 7. `GET /admin/webhooks/dead-letter` — Listar eventos em DLQ (ADMIN)

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "9a8b7c6d-5e4f-3a2b-1c0d-9e8f7a6b5c4d",
      "eventId": "7d4b2f3e-0a9c-4d8b-9f2e-6c3d4e5f6a7b",
      "webhookEndpointId": "3e2f9a10-6d21-4e2a-9a11-2f0e8b6c9d10",
      "failureReason": "HTTP 503 after 5 attempts",
      "attempts": 5,
      "movedAt": "2026-09-15T01:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Erros: `403 FORBIDDEN` (role diferente de ADMIN).

### 8. `POST /admin/webhooks/dead-letter/:id/replay` — Reprocessar evento (ADMIN)

**Response (200 OK):**
```json
{
  "outboxId": "1f2e3d4c-5b6a-7c8d-9e0f-1a2b3c4d5e6f",
  "status": "PENDING",
  "requeuedAt": "2026-09-15T09:00:00.000Z",
  "requeuedBy": "4a5b6c7d-8e9f-0a1b-2c3d-4e5f6a7b8c9d"
}
```

Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN`.

## Matriz de Erros (`WEBHOOK_*`)

| Código | HTTP Status | Descrição | Classe base reutilizada |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook não encontrado | `NotFoundError` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Cliente informado não existe | `NotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | URL não é HTTPS ou é malformada | `BadRequestError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação exige secret e nenhum foi encontrado | `BadRequestError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB | `UnprocessableEntityError` |
| `WEBHOOK_INACTIVE` | 409 | Webhook está desativado e não pode ser usado | `ConflictError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Evento em DLQ não encontrado para replay | `NotFoundError` |
| `WEBHOOK_DUPLICATE_ENDPOINT` | 409 | Já existe webhook ativo com a mesma URL para o cliente | `ConflictError` |

Todas as classes seguem o padrão de `src/shared/errors/http-errors.ts` (`AppError` como base), sem exigir mudanças no middleware de erro (`src/middlewares/error.middleware.ts`), que já trata `AppError` genericamente.

## Estratégias de Resiliência

- **Timeout de entrega HTTP:** 10 segundos por tentativa; excedido, tratado como falha e segue para retry ([09:42] Diego/Sofia).
- **Retry com backoff exponencial:** 5 tentativas (1m/5m/30m/2h/12h) — ver ADR-003.
- **DLQ:** eventos que esgotam retries não são perdidos, ficam disponíveis para replay manual.
- **Limite de payload:** 64KB; eventos maiores falham na inserção do outbox com `WEBHOOK_PAYLOAD_TOO_LARGE`, não são truncados ([09:23]-[09:24]).
- **Fallback:** não há fallback automático de canal (ex.: e-mail) nesta fase — fora de escopo ([09:37]-[09:38]).
- **Idempotência do lado do cliente:** garantida via `X-Event-Id` estável entre tentativas (ADR-005), não pelo servidor.

## Observabilidade

- **Métricas** (via instrumentação a ser adicionada, ex.: contadores/histogramas expostos pelo processo do worker): taxa de eventos pendentes no outbox, taxa de sucesso/falha de entrega, latência de entrega (do `createdAt` do evento até `DELIVERED`), tamanho da fila de DLQ, tempo de resposta HTTP por tentativa.
- **Logs:** uso do logger Pino já existente (`src/shared/logger/index.ts`), com campos estruturados por evento (`eventId`, `webhookEndpointId`, `orderId`, `attempts`, `httpStatus`, `durationMs`). Redação de `secret`/`X-Signature` nos logs, seguindo a configuração de redaction já presente no logger.
- **Tracing:** cada tentativa de entrega deve propagar/logar o `eventId` como correlation id, permitindo rastrear uma entrega específica através de múltiplas tentativas de retry nos logs.

## Dependências e Compatibilidade

- Depende do schema Prisma existente (`Customer`, `Order`) para relacionar `WebhookEndpoint.customerId` e montar o payload do evento.
- Depende de `OrderService.changeStatus` continuar sendo o único ponto de mutação de status de pedido (nenhuma outra rota ou service deve alterar `order.status` diretamente sem passar pela mesma transação).
- Requer biblioteca HTTP client com suporte a timeout configurável no worker (ex.: `fetch` nativo do Node com `AbortController`, ou `axios`/`undici` já presentes nas dependências do projeto, a confirmar em `package.json`).
- Não introduz mudanças de schema em tabelas existentes — apenas tabelas novas.

## Integração com o Sistema Existente

Esta seção detalha, arquivo a arquivo, como o módulo de webhooks se integra ao código já existente:

1. **`src/modules/orders/order.service.ts`** (linhas 126-179, método `changeStatus`): dentro da transação `this.prisma.$transaction(async (tx) => {...})` já existente, após `tx.orderStatusHistory.create()`, adiciona-se uma chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Nenhuma outra lógica do método é alterada; a função recebe o `tx` ativo em vez de abrir uma transação própria, preservando a atomicidade (ADR-001, decidido em [09:40]-[09:41]).

2. **`src/modules/orders/order.status.ts`**: reaproveitado apenas para leitura — o módulo de webhooks não modifica a máquina de estados, mas usa os mesmos valores do enum `OrderStatus` (via `@prisma/client`) para validar o array `statuses` na configuração de cada `WebhookEndpoint`, garantindo que só sejam aceitos status válidos do domínio de pedidos.

3. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`**: todas as exceções do módulo de webhooks (`WebhookNotFoundError`, `InvalidWebhookUrlError`, `PayloadTooLargeError`, etc.) estendem as classes HTTP já existentes (`NotFoundError`, `BadRequestError`, `UnprocessableEntityError`, `ConflictError`), seguindo exatamente o padrão de `InvalidStatusTransitionError`/`InsufficientStockError`, apenas com códigos prefixados `WEBHOOK_*` (ADR-006, [09:28]-[09:29]).

4. **`src/middlewares/auth.middleware.ts`**: os endpoints administrativos (`GET /admin/webhooks/dead-letter`, `POST /admin/webhooks/dead-letter/:id/replay`) reutilizam o middleware `requireRole('ADMIN')` sem nenhuma modificação nesse arquivo. Os demais endpoints do módulo usam apenas `authenticate`, também sem alterações ([09:35]-[09:36]).

5. **`src/middlewares/error.middleware.ts`**: nenhuma alteração necessária — como todas as exceções do módulo estendem `AppError`, o middleware já tratado genericamente para `AppError`, `ZodError` e erros do Prisma cobre o novo módulo automaticamente ([09:29] Bruno).

6. **`src/shared/logger/index.ts`**: reutilizado sem alterações pelo worker e pelos services do módulo de webhooks para logging estruturado (incluindo o log de auditoria de replay de DLQ), aproveitando a configuração de redaction de segredos já existente.

7. **`prisma/schema.prisma`**: extensão aditiva com os modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter` e o enum `WebhookOutboxStatus`, seguindo a convenção de UUID (`@default(uuid()) @db.Char(36)`), timestamps e `@@map` em snake_case já usada por todos os modelos existentes (`Order`, `Customer`, `OrderStatusHistory`, etc.).

8. **`src/server.ts`**: usado como referência de padrão (não modificado) para a criação do novo entry point `src/worker.ts`, que segue a mesma forma de inicialização de configuração/logger/Prisma, mas sem subir o servidor HTTP Express — apenas o loop de polling.

## Critérios de Aceite Técnicos

- Uma mudança de status via `PATCH /orders/:id/status` que corresponda a um webhook ativo do cliente gera uma linha em `webhook_outbox` na mesma transação, visível imediatamente após o commit.
- Se a transação de `changeStatus` sofrer rollback por qualquer motivo, nenhuma linha de outbox é criada.
- O worker processa e entrega um evento pendente em, no máximo, ~2 segundos de latência de leitura, mais o tempo de resposta do endpoint do cliente.
- Uma entrega falha aciona retry conforme a tabela de backoff (ADR-003), e após 5 falhas o evento aparece em `webhook_dead_letter`.
- Toda entrega inclui headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
- A assinatura em `X-Signature` é validável pelo cliente usando HMAC-SHA256 sobre o corpo bruto da requisição com o secret correto (atual ou anterior, se dentro do grace period de 24h).
- Endpoints administrativos de DLQ retornam `403 FORBIDDEN` para usuários sem role `ADMIN`.

## Riscos e Mitigação

- **Risco:** payload do outbox cresce sem controle, degradando performance de leitura do worker. **Mitigação:** índice composto `(status, nextAttemptAt)`, limite de 64KB por payload, arquivamento futuro fora do escopo atual mas identificado como dívida técnica (ADR-001).
- **Risco:** vazamento de secret HMAC compromete a autenticidade das notificações de um cliente. **Mitigação:** secret único por endpoint (não global), suporte a rotação com grace period de 24h, revisão de segurança dedicada antes do deploy ([09:45]-[09:47] Sofia).
- **Risco:** cliente satura a própria integração se o OMS gerar muitos eventos em rajada. **Mitigação:** nenhuma nesta fase — ponto em aberto documentado no RFC, a ser monitorado ([09:38]-[09:39]).

## Fontes

Toda decisão e requisito citados neste documento têm origem em `TRANSCRICAO.md` (timestamps indicados) ou nos arquivos de código apontados na seção "Integração com o sistema existente". Ver `docs/TRACKER.md` para o mapeamento completo item a item.
