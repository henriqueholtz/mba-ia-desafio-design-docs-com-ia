# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

Ao expor dados de pedidos para endpoints HTTP externos, é necessário garantir autenticidade e integridade do payload, para que o cliente possa verificar que a notificação realmente veio do OMS e não foi adulterada em trânsito ([09:19] Sofia).

Sofia propôs o padrão de mercado: assinar o corpo da requisição com um segredo compartilhado (HMAC) e enviar a assinatura em um header (`X-Signature`), permitindo que o cliente verifique ([09:20]). Bruno perguntou qual algoritmo; Sofia definiu SHA-256: "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20]).

Um ponto crítico levantado por Sofia foi o uso de um segredo único por endpoint de cliente, em vez de um segredo global da plataforma: "Senão se vaza uma, vaza tudo" ([09:21]). Diego reforçou com um precedente real: um cliente já vazou um secret em log de aplicação própria anteriormente ([09:22]).

Sofia também exigiu suporte a rotação do secret, com período de graça de 24 horas em que o secret antigo continua válido em paralelo ao novo, dando tempo ao cliente para migrar ([09:21]-[09:22]).

Adicionalmente, foram fixados como requisitos não funcionais (não decisões arquiteturais separadas, conforme Larissa em [09:23]): TLS obrigatório para a URL do webhook (validação de schema, não decisão arquitetural) e limite de 64KB no tamanho do payload, com erro explícito em vez de truncamento caso excedido ([09:22]-[09:24] Sofia/Diego).

## Decisão

Cada registro de webhook (tabela de configuração, com `url`, `secret`, `customer_id`, flag de ativo) recebe um secret único gerado pela plataforma. Toda entrega HTTP inclui um header `X-Signature` contendo o HMAC-SHA256 do corpo da requisição, calculado com esse secret.

O secret pode ser rotacionado via endpoint dedicado; durante 24 horas após a rotação, tanto o secret antigo quanto o novo são aceitos para validar assinaturas, após o que o secret antigo expira.

URLs de webhook devem ser HTTPS (validação em schema Zod, rejeitando `http://`), e o payload de cada evento é limitado a 64KB, retornando erro na inserção do outbox se excedido.

## Alternativas Consideradas

- **Secret global da plataforma, compartilhado entre todos os clientes**: descartado por Sofia — um único vazamento comprometeria a autenticidade de notificações para todos os clientes simultaneamente ([09:21]).
- **Rotação de secret sem período de graça** (troca imediata, invalidando o secret antigo na hora): implicitamente descartado — Sofia exigiu 24h de sobreposição para não quebrar integrações de clientes durante a migração ([09:21]-[09:22]).

## Consequências

**Positivas:**
- Isolamento de blast radius: vazamento do secret de um cliente não compromete os demais.
- Verificação de integridade e autenticidade alinhada a padrão de mercado, facilitando integração por parte dos clientes.
- Rotação com grace period evita downtime de integração durante troca de credenciais.

**Negativas (trade-off explícito):**
- Necessário armazenar e gerenciar dois secrets válidos simultaneamente durante o período de rotação (complexidade adicional no schema e na lógica de validação).
- Geração, exibição e rotação de secrets são operações sensíveis que exigem cuidado extra de implementação e revisão de segurança dedicada antes do deploy (Sofia reservou ao menos 2 dias úteis para essa revisão, [09:45]-[09:47]).
