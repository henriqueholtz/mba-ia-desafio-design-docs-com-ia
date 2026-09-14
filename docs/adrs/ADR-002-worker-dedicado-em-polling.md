# ADR-002: Worker dedicado em processo separado, com polling de 2 segundos

## Status

Aceito

## Contexto

Com o padrão Outbox definido (ADR-001), é necessário decidir como o `webhook_outbox` é processado e entregue aos clientes. Diego propôs um loop de polling, verificando a tabela a cada 2 segundos, processando os eventos pendentes mais antigos primeiro ([09:09]).

Bruno sugeriu usar triggers do MySQL para reagir de forma mais reativa a novos eventos ([09:09]). Diego descartou: MySQL não tem um mecanismo equivalente ao `NOTIFY`/`LISTEN` do PostgreSQL; triggers só conseguem executar SQL, não notificar um processo externo, e contornos (escrever em arquivo, chamar um endpoint a partir do trigger) foram considerados inadequados ([09:09]).

O requisito de negócio de "tempo real" foi esclarecido por Marcos: os clientes consideram aceitável qualquer latência abaixo de 10 segundos ([09:02]). Um polling de 2 segundos atende esse requisito com folga ([09:09]-[09:10], confirmado por Marcos).

Diego também levantou que o worker precisa rodar como processo separado da API: se ficar embutido na mesma instância, um restart da API mata o worker junto ([09:11]).

## Decisão

Implementar um worker dedicado como processo Node.js independente, com entry point próprio (`src/worker.ts`, espelhando o padrão de `src/server.ts`) e script `npm run worker`. O worker executa um loop de polling a cada 2 segundos, buscando lotes pequenos de eventos pendentes em `webhook_outbox` ordenados por `created_at`.

O worker usa sua própria instância de `PrismaClient` (não compartilhada com a API), pois `PrismaClient` é por processo — é uma instância nova apontando para o mesmo `DATABASE_URL`, não uma conexão compartilhada em memória ([09:29]-[09:30] Bruno/Diego).

Com um único worker, o processamento segue a ordem de `created_at`, preservando ordenação por `order_id` na prática — mas essa garantia não é global e deixa de existir se o worker for escalado horizontalmente no futuro ([09:12]-[09:14] Diego/Larissa/Marcos).

## Alternativas Consideradas

- **Triggers de banco de dados para acionar o processamento reativamente**: descartado porque MySQL não oferece mecanismo de notificação a processos externos (ao contrário do Postgres NOTIFY/LISTEN); a alternativa de workaround (trigger escrevendo em arquivo ou chamando HTTP) foi julgada inadequada e frágil ([09:09] Bruno/Diego).
- **Worker embutido no processo da API**: implicitamente descartado — Diego argumenta que precisa ser processo separado para não acoplar o ciclo de vida do worker ao da API ([09:11]).

## Consequências

**Positivas:**
- Simplicidade operacional: sem infraestrutura de mensageria adicional, sem necessidade de recursos como NOTIFY/LISTEN.
- Latência de 2s está muito abaixo do SLA de 10s aceito pelos clientes, com folga confortável.
- Processo isolado da API: reinício ou instabilidade da API não afeta o processamento de eventos, e vice-versa.

**Negativas (trade-off explícito):**
- Não há garantia de ordenação global de eventos entre pedidos diferentes caso o worker seja escalado para múltiplas instâncias no futuro — limitação documentada e aceita para esta fase ([09:13] Diego: "problema do futuro, não agora").
- Polling gera carga constante de leitura no banco mesmo quando não há eventos pendentes (mitigado pelo intervalo de 2s e índices adequados).
- Exige deploy e monitoramento de um processo adicional além da API.
