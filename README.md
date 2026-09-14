# Sistema de Webhooks de Notificação de Pedidos — Design Docs

Documentação técnica produzida para o desafio "Da Reunião ao Documento: Design Docs Gerados por IA" do MBA.

## Sobre o desafio

O desafio pede para transformar a transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) em um pacote completo de design docs — PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade — para uma feature que nunca chegou a ser implementada nem documentada: um sistema de webhooks de notificação de pedidos para um OMS (Order Management System) já existente em produção.

O ponto central do exercício não é escrever documentação bonita, e sim manter rastreabilidade absoluta: toda decisão, requisito ou restrição registrada precisa ter origem identificável na transcrição (com timestamp e falante) ou no código-fonte real do projeto. Nada pode ser inventado — inclusive o que **não** entra no escopo (itens descartados ou adiados na reunião) precisa ser identificado com o mesmo rigor do que entra.

## Ferramentas de IA utilizadas

- **Claude Code (Claude Sonnet 5)** — ferramenta única usada neste desafio, em modo agente com acesso de leitura ao repositório (código-fonte, `prisma/schema.prisma`, `TRANSCRICAO.md`). Usada para: (1) explorar o código existente e extrair os padrões reais de módulos/erros/middlewares; (2) ler a transcrição inteira e mapear cada decisão, requisito, alternativa descartada e ponto adiado ao timestamp exato; (3) redigir PRD, RFC, FDD, ADRs, Tracker e este README a partir desse levantamento; (4) revisão cruzada dos documentos entre si para evitar duplicação de conteúdo entre RFC e FDD.

## Workflow adotado

1. **Modo de planejamento** — antes de gerar qualquer documento, usei o modo de planejamento do Claude Code para forçar uma etapa de exploração dedicada do repositório e da transcrição, evitando que a IA partisse direto para a redação sem entender o código e o contexto completo da reunião.
2. **Exploração e levantamento de fatos** — um subagente de exploração leu a íntegra de `TRANSCRICAO.md` e os arquivos relevantes de `src/` e `prisma/schema.prisma`, produzindo um relatório estruturado com timestamps exatos para cada decisão, requisito, alternativa descartada e item adiado, além dos caminhos reais de arquivos a citar. Esse relatório virou a única fonte de verdade para todos os documentos seguintes.
3. **ADRs primeiro** — as 6 decisões arquiteturais centrais da reunião (mais uma decisão secundária relevante, o snapshot do payload) foram registradas como ADRs individuais antes de qualquer outro documento, formando o "esqueleto" técnico do restante.
4. **RFC em seguida** — consolidação da proposta técnica em alto nível, linkando os ADRs já escritos, com alternativas descartadas e questões em aberto extraídas diretamente da transcrição.
5. **FDD depois do RFC** — detalhamento de implementação (fluxos, contratos HTTP, matriz de erros, integração com arquivos reais do código), aprofundando o que o RFC deixou em nível de arquitetura.
6. **PRD por último entre os três grandes documentos** — com RFC, FDD e ADRs prontos, o PRD foi essencialmente uma consolidação em nível de produto/negócio.
7. **Tracker** — montado varrendo os quatro documentos e os sete ADRs, mapeando cada item identificável a um timestamp da transcrição ou a um caminho de arquivo do código.
8. **README** — escrito por último, já com o processo completo para poder descrevê-lo com precisão.

## Prompts customizados

Prompt usado para a etapa de exploração/levantamento de fatos (adaptado para exigir timestamps exatos, essencial para o Tracker):

```
Investigate and report:
1. Repo structure — modules in src/, whether docs/ and docs/adrs/ already exist.
2. Order module details — status state machine, changeStatus transaction,
   existing error classes/patterns, audit/logging of status changes,
   Prisma schema relevant to orders.
3. TRANSCRICAO.md — read the FULL file. Summarize in detail:
   - participants (names and roles)
   - all functional requirements discussed with timestamps
   - the 6 main technical decisions, quoted/paraphrased with timestamp
   - alternatives discussed and discarded (with reasoning/trade-off and timestamp)
   - things explicitly postponed to future phases (with timestamp)
   - non-functional requirements mentioned
   - secondary technical details (payload format, timeouts, headers, endpoint naming)

Report back in a structured, thorough way — exact timestamps and speaker
names are required for traceability purposes since every requirement/decision
needs to link to its transcript timestamp or source code file.
```

Prompt usado para orientar a redação do FDD, especificamente a seção obrigatória de integração com o código existente:

```
Escreva a seção "Integração com o sistema existente" do FDD citando pelo
menos 4 caminhos de arquivo REAIS do repositório (não invente nomes).
Para cada arquivo, explique especificamente COMO o módulo de webhooks se
integra a ele: por exemplo, qual método existente é estendido, qual classe
de erro existente é reutilizada como base, qual padrão de código é seguido
sem modificação. Se não for possível apontar um caminho real e uma forma
concreta de integração, não inclua o item.
```

## Iterações e ajustes

Ao longo da produção, os seguintes ajustes foram necessários em relação ao primeiro levantamento:

1. **Separação de decisão arquitetural vs. requisito não funcional**: a primeira leitura da transcrição misturava itens como "TLS obrigatório" e "limite de payload de 64KB" junto às 6 decisões arquiteturais principais. Ao reler a fala da própria Larissa em [09:23] ("não vejo como decisão arquitetural separada, é só requisito não funcional"), esses itens foram retirados da lista de candidatos a ADR e realocados para as seções de requisitos não funcionais do PRD e do FDD — evitando ADRs artificialmente inflados com decisões de validação, não de arquitetura.
2. **Risco de duplicação entre RFC e FDD**: a primeira versão mental do RFC tendia a descrever os fluxos técnicos com o mesmo nível de detalhe do FDD (ex.: tabela de backoff, payload completo). Foi necessário reescrever a seção "Proposta Técnica" do RFC para ficar em nível de arquitetura ("o quê e por quê"), movendo toda a tabela de retries, os contratos de endpoint e os campos de payload exclusivamente para o FDD ("como").
3. **Itens descartados vs. itens adiados**: a transcrição contém tanto ideias descartadas de forma definitiva (painel visual — "projeto separado do time de frontend") quanto pontos deixados conscientemente em aberto para decisão futura (rate limiting — "observar e decidir depois"). Esses dois tipos foram tratados de forma diferente: o painel entrou apenas em "Fora de escopo" do PRD; o rate limiting entrou tanto em "Fora de escopo" quanto em "Questões em Aberto" do RFC, já que é uma lacuna reconhecida e não uma decisão fechada de exclusão.
4. **Cobertura do Tracker**: a primeira passada do Tracker cobria bem os requisitos funcionais, mas tinha poucas linhas com Fonte = CODIGO. Foi necessário revisar o FDD e os ADRs especificamente em busca de referências a arquivos reais (`order.service.ts`, `order.status.ts`, `http-errors.ts`, `auth.middleware.ts`, `error.middleware.ts`, `logger/index.ts`, `schema.prisma`, `response.ts`) para elevar a cobertura de código acima do mínimo de 5 linhas exigido.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md) — problema, escopo e métricas de sucesso (visão de produto).
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica de alto nível, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — cada decisão arquitetural isolada, em ordem (`ADR-001` a `ADR-007`).
4. [`docs/FDD.md`](docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — referência cruzada de todo item a seu timestamp na transcrição ou arquivo de código.
6. [`TRANSCRICAO.md`](TRANSCRICAO.md) — fonte primária, para conferência pontual de qualquer citação.

O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado nesta entrega — serviu exclusivamente como referência e contexto para os documentos acima.
