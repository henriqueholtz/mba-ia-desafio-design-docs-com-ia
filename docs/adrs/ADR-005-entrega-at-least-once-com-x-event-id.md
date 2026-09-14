# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Diego definiu que a garantia de entrega dos webhooks é at-least-once (pelo menos uma vez), o que significa que a entrega duplicada de um mesmo evento é possível e deve ser tratada pelo cliente ([09:24]).

Bruno questionou como o cliente diferenciaria eventos duplicados de eventos novos ([09:25]). Diego propôs um header `X-Event-Id`, carregando um UUID gerado no momento em que o evento é inserido no outbox, único por evento — o cliente usa esse identificador para deduplicar do seu lado ([09:25]).

Sofia observou que essa abordagem transfere a responsabilidade de deduplicação para o cliente ([09:25]). Diego defendeu como padrão de mercado, citando Stripe e GitHub como exemplos de provedores que adotam a mesma estratégia, e argumentou que uma garantia exactly-once exigiria coordenação bidirecional muito mais complexa para resolver um problema que at-least-once + `event_id` já resolve em "99% dos casos" ([09:25]). Marcos se comprometeu a documentar isso de forma proeminente no portal de desenvolvedores ([09:26]).

## Decisão

O sistema garante entrega at-least-once: cada evento pode, em cenários de falha/retry, ser entregue mais de uma vez ao mesmo endpoint. Cada evento carrega um `event_id` (UUID) gerado na inserção no outbox, enviado no header `X-Event-Id` em toda tentativa de entrega (incluindo reentregas). Cabe ao cliente implementar deduplicação usando esse identificador.

Essa decisão deve ser documentada de forma explícita e proeminente na documentação pública dos webhooks (portal de desenvolvedores / FDD), para que integradores saibam que precisam tratar duplicidade.

## Alternativas Consideradas

- **Garantia exactly-once**: descartada por exigir coordenação bidirecional entre plataforma e cliente (ex.: confirmação transacional de recebimento), complexidade desproporcional ao ganho, dado que at-least-once com deduplicação por `event_id` resolve a esmagadora maioria dos casos práticos — mesma abordagem adotada por Stripe e GitHub ([09:25] Diego).

## Consequências

**Positivas:**
- Simplicidade de implementação no lado do servidor: não é necessário rastrear confirmações de recebimento do cliente.
- Alinhado a padrão de mercado amplamente adotado (Stripe, GitHub), reduzindo a curva de aprendizado para integradores.
- Compatível com o modelo de retry e DLQ (ADR-003), já que reentregas não quebram a garantia de consistência.

**Negativas (trade-off explícito):**
- Transfere a responsabilidade de deduplicação para o cliente, exigindo documentação clara (portal de desenvolvedores) e possível fricção de suporte se clientes não implementarem a deduplicação corretamente.
- Sem uma tabela de confirmação de entrega do lado do servidor, o sistema não tem visibilidade se o cliente de fato processou (idempotentemente) um evento duplicado.
