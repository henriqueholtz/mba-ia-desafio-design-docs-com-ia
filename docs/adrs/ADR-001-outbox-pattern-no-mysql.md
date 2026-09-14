# ADR-001: Padrão Outbox no MySQL para publicação de eventos de pedido

## Status

Aceito

## Contexto

O OMS precisa notificar sistemas externos (clientes B2B: Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda, substituindo o polling atual em `GET /orders` ([09:00]-[09:02] Marcos). A mudança de status já ocorre dentro de uma transação Prisma única em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179), que atualiza `orders`, ajusta estoque e insere um registro em `order_status_history`.

Bruno levantou a hipótese de disparar a chamada HTTP para o cliente de forma síncrona, dentro dessa mesma transação ([09:03]-[09:04]). Diego apontou o problema: um cliente lento ou indisponível bloquearia a transação de status para pedidos completamente alheios, e não há uma estratégia de rollback sensata se a chamada falhar no meio do commit ([09:04], [09:06]: "Síncrono está fora de questão").

A alternativa levantada por Larissa foi usar uma fila dedicada, como Redis Streams ([09:07]). Diego descartou: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering" ([09:07]).

## Decisão

Adotar o padrão Outbox: dentro da mesma transação SQL que já atualiza `orders` e insere em `order_status_history`, inserir também uma linha na nova tabela `webhook_outbox`, contendo o payload do evento já serializado. Um worker separado (ver ADR-002) lê essa tabela de forma assíncrona e realiza a entrega HTTP.

Isso garante consistência transacional: se a transação de mudança de status commitar, o evento está garantidamente persistido; se ela sofrer rollback, o evento nunca existiu ([09:06] Diego).

A tabela usa índice em status (`pendente, processando, falhou, entregue`) e `created_at` para leitura eficiente pelo worker; linhas entregues são candidatas a arquivamento após 30 dias, mas esse arquivamento fica fora do escopo desta feature ([09:07]-[09:08] Diego).

## Alternativas Consideradas

- **Dispatch síncrono dentro de `changeStatus`**: descartado porque acopla a disponibilidade de clientes externos à disponibilidade da API de pedidos, sem estratégia de rollback viável ([09:03]-[09:06]).
- **Fila externa (Redis Streams)**: descartado por introduzir infraestrutura nova (Redis Cluster) para um time pequeno, quando o MySQL já existente resolve o problema com uma tabela adicional ([09:07] Diego).

## Consequências

**Positivas:**
- Garantia forte de consistência entre mudança de status e criação do evento — nunca há status mudado sem evento correspondente (e vice-versa).
- Não introduz nova infraestrutura; reaproveita o MySQL e o Prisma já existentes no projeto.
- Falha de entrega ao cliente externo nunca impacta a operação principal de pedidos.

**Negativas (trade-off explícito):**
- Introduz uma etapa de leitura assíncrona (polling, ver ADR-002), o que significa que a entrega não é instantânea — há uma latência mínima equivalente ao intervalo de polling.
- A tabela `webhook_outbox` cresce continuamente e exige rotina de limpeza/arquivamento no futuro (fora do escopo atual, risco documentado no FDD).
- Acopla o schema de pedidos (`prisma/schema.prisma`) a uma nova tabela de infraestrutura de mensageria, ainda que de forma desacoplada via worker.
