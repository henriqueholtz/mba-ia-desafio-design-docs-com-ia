# ADR-007: Snapshot do payload no momento da inserção do evento (não no envio)

## Status

Aceito

## Contexto

Ao final da reunião, Larissa, Bruno e Diego discutiram um detalhe de implementação com impacto direto na consistência dos dados entregues aos clientes: o payload do evento deve ser montado (snapshot) no momento em que o evento é inserido no outbox, ou re-renderizado a partir do estado atual do pedido no momento em que o worker efetivamente envia a notificação? ([09:51]-[09:52]).

Se o payload fosse gerado no momento do envio, uma mudança de status subsequente no mesmo pedido (por exemplo, se o pedido mudasse de status novamente antes do worker processar o evento anterior) poderia fazer o payload não refletir mais o estado que gerou aquele evento específico, quebrando a semântica "isso mudou de X para Y". O grupo decidiu que o payload deve ser um snapshot imutável, serializado e persistido já no momento da inserção na `webhook_outbox`, dentro da mesma transação de `changeStatus` ([09:51]-[09:52] Larissa/Bruno/Diego: "Beleza, snapshot. Decidido.").

Essa decisão também reforça a convenção de identificadores já usada no projeto: todas as novas tabelas (`webhook_outbox`, `webhook_dead_letter`, tabela de configuração de webhooks) usam UUID como chave primária, seguindo o padrão de todo o schema Prisma existente — "Tudo é uuid" ([09:51] Larissa/Diego).

## Decisão

O payload de cada evento de webhook é serializado por completo (incluindo `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`) e persistido como snapshot imutável na linha do `webhook_outbox` no momento da inserção, dentro da mesma transação que altera o status do pedido. O worker apenas lê e envia esse payload já pronto, sem consultar novamente o estado atual do pedido no banco. Todas as tabelas novas usam `id` do tipo UUID (`@default(uuid()) @db.Char(36)`), consistente com o restante do `prisma/schema.prisma`.

## Alternativas Consideradas

- **Re-renderizar o payload no momento do envio, a partir do estado atual do pedido**: descartada — quebraria a semântica do evento se o pedido mudasse de estado novamente entre a inserção e o envio, fazendo o payload não corresponder mais à transição que originou aquele evento específico ([09:51]-[09:52]).

## Consequências

**Positivas:**
- Garante que cada evento entregue reflete fielmente a transição de status que o originou, mesmo que o pedido tenha mudado de estado novamente antes do envio.
- Reduz acoplamento do worker ao schema de `orders` — o worker não precisa fazer joins ou consultas adicionais para montar o payload, apenas ler e enviar.
- Consistente com a convenção de UUID já usada em 100% das tabelas do projeto, reduzindo decisões ad-hoc de modelagem.

**Negativas (trade-off explícito):**
- O payload armazenado pode ficar "desatualizado" em relação ao estado mais recente do pedido no momento da entrega — aceitável, pois o cliente pode sempre consultar `GET /orders/:id` para o estado atual, e o payload não inclui itens do pedido para evitar acoplamento excessivo ([09:43] Diego).
- Aumenta o tamanho de cada linha do outbox (payload completo persistido), reforçando a necessidade do limite de 64KB (ADR-004) e de rotina futura de arquivamento (ADR-001).
