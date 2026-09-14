# ADR-006: Reuso máximo dos padrões arquiteturais já existentes no projeto

## Status

Aceito

## Contexto

Ao desenhar o novo módulo de webhooks, Bruno e Diego discutiram como encaixá-lo na estrutura já existente do projeto em vez de introduzir convenções novas ([09:27]-[09:30]).

O projeto já segue um padrão de um módulo por domínio em `src/modules/<dominio>/`, com arquivos `controller`, `service`, `repository`, `routes` e `schemas` (ver, por exemplo, `src/modules/orders/`). Bruno propôs seguir a mesma estrutura para `src/modules/webhooks/` ([09:27]-[09:28]).

Sobre tratamento de erros, o projeto já possui uma classe base `AppError` (`src/shared/errors/app-error.ts`) e subclasses específicas como `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), cada uma com um código de erro (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`). Bruno propôs replicar o padrão para o módulo de webhooks, com códigos prefixados `WEBHOOK_*` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28]-[09:29]). Larissa confirmou o prefixo `WEBHOOK_` como padrão para todos os códigos de erro do módulo ([09:29]).

Bruno também notou que o logger (Pino, `src/shared/logger/index.ts`) já é usado no projeto todo e não precisa de nada novo, e que o middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros do Prisma sem precisar de alterações ([09:29]).

Sobre infraestrutura de dados, Diego perguntou se o worker abriria a mesma instância de `PrismaClient` da API ou uma separada; Bruno esclareceu que, como `PrismaClient` é por processo, o worker precisa de uma instância nova (mesmo `DATABASE_URL`, processo Node diferente) ([09:29]-[09:30]).

Larissa fechou a discussão resumindo a decisão: reuso máximo do que já existe — `AppError`, Pino, middleware de erro, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro — o módulo de webhooks se encaixa como qualquer outro módulo do projeto ([09:30]).

## Decisão

O módulo de webhooks é implementado em `src/modules/webhooks/`, seguindo exatamente a mesma estrutura de arquivos dos módulos existentes (`controller`, `service`, `repository`, `routes`, `schemas`). Toda validação de entrada usa Zod, no mesmo padrão de `order.schemas.ts`. Erros do domínio de webhooks estendem `AppError`/subclasses HTTP existentes (`ConflictError`, `NotFoundError`, `UnprocessableEntityError`, etc.) com códigos prefixados `WEBHOOK_*`. Logging usa a instância Pino já existente, sem configuração adicional. O middleware de erro centralizado não é modificado — `AppError` lançado pelo módulo de webhooks já é tratado automaticamente. O worker (`src/worker.ts`, ver ADR-002) instancia seu próprio `PrismaClient`, apontando para o mesmo `DATABASE_URL` da API.

## Alternativas Consideradas

- **Criar uma estrutura de pastas ou convenção de erros nova e específica para webhooks** (por exemplo, um pacote isolado fora de `src/modules`, ou uma hierarquia de exceções própria): descartada — aumentaria a carga cognitiva do time sem ganho real, indo contra a decisão explícita de Larissa de reusar o que já existe ([09:30]).

## Consequências

**Positivas:**
- Menor curva de aprendizado para o time, que já conhece a estrutura de módulos, o padrão de erros e o logger.
- Menor superfície de código novo: nenhuma mudança é necessária no middleware de erro, no logger ou na configuração de banco existente.
- Consistência do código-base facilita revisão e manutenção futura.

**Negativas (trade-off explícito):**
- Amarra o módulo de webhooks às limitações do padrão atual (por exemplo, um `PrismaClient` por processo em vez de um pool compartilhado mais sofisticado), o que pode exigir refatoração se o padrão de módulos mudar no futuro.
- Códigos de erro `WEBHOOK_*` precisam ser mantidos manualmente em sincronia com a convenção existente (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`), sem enforcement automatizado (ex.: lint de nomenclatura).
