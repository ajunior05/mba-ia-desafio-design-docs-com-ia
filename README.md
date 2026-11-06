# Sistema de Webhooks de Notificação de Pedidos — Design Docs

> O enunciado original do desafio está preservado em [`docs/ENUNCIADO.md`](docs/ENUNCIADO.md).

## Sobre o Desafio

Este repositório é a entrega do desafio "Da Reunião ao Documento: Design Docs Gerados por IA", do módulo de Design Docs com IA. O ponto de partida era uma transcrição literal de ~55 minutos de uma reunião técnica (`TRANSCRICAO.md`) em que tech lead, PM, dois engenheiros e uma engenheira de segurança decidem, ao vivo, como construir um sistema de webhooks de notificação de pedidos para um OMS já existente — e o código-fonte desse OMS, que não tinha nenhum mecanismo de evento ou notificação externa até então.

A tarefa não era desenhar a solução do zero: a decisão técnica já estava tomada na reunião. O trabalho era de **documentação rastreável**: ler a transcrição e o código com cuidado, separar o que foi decidido do que foi descartado ou adiado, e produzir um pacote de design docs (PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade) em que cada afirmação pudesse ser rastreada até uma fala específica da reunião ou a um arquivo real do código — sem inventar requisito, decisão ou restrição.

## Ferramentas de IA Utilizadas

- **Claude Code (Anthropic)** — ferramenta única usada neste desafio, em modo agente dentro do terminal, com acesso direto ao repositório clonado. Usada para ler a transcrição completa e os arquivos de código relevantes, extrair decisões/requisitos/restrições, estruturar e escrever todos os documentos (ADRs, RFC, FDD, PRD, Tracker), e para a revisão de consistência final.

## Workflow Adotado

A ordem seguida foi a sugerida pelo próprio desafio, por ser a que menos gera retrabalho:

1. **Fork e setup** do repositório base, clonado localmente.
2. **Contextualização**: leitura completa de `TRANSCRICAO.md` e dos arquivos de código mais relevantes (`order.service.ts`, `order.status.ts`, hierarquia de erros, middlewares, `schema.prisma`, padrão de rotas) antes de escrever qualquer documento.
3. **ADRs primeiro** — as 6 decisões arquiteturais principais da reunião (outbox, worker dedicado, retry/DLQ, HMAC, at-least-once, reuso de padrões) mais uma decisão secundária relevante (snapshot do payload), formalizadas como o esqueleto do resto da documentação.
4. **RFC**, consolidando a proposta em nível de arquitetura e linkando os ADRs já escritos.
5. **FDD**, detalhando contratos HTTP, fluxos, matriz de erros e os pontos exatos de integração com o código existente.
6. **PRD**, por último entre os documentos grandes — com RFC, FDD e ADRs prontos, a consolidação de alto nível (objetivos, métricas, riscos de negócio) ficou direta.
7. **Tracker**, varrendo os quatro documentos anteriores linha a linha.
8. **Revisão final** contra a checklist de critérios de aceite do desafio (ver seção de Iterações abaixo).

## Prompts Customizados

Os dois prompts abaixo foram os mais determinantes para a qualidade do resultado — ambos dirigidos, não genéricos, seguindo a recomendação do próprio desafio.

**1. Extração de decisões para os ADRs**, usado antes de escrever qualquer ADR:

```
Leia TRANSCRICAO.md e o código em src/modules/orders/order.service.ts,
src/modules/orders/order.status.ts e src/shared/errors/. Extraia apenas
as decisões arquiteturais FECHADAS na reunião (não pontos adiados ou
descartados). Para cada decisão, produza um ADR em formato MADR (Status,
Contexto, Decisão, Alternativas Consideradas com pelo menos 1 alternativa
real discutida, Consequências positivas e negativas com trade-off
explícito). Cite ao final de cada ADR o(s) timestamp(s) exato(s) e o
nome de quem participou da decisão, no formato [hh:mm] Nome. Pelo menos
um ADR deve referenciar explicitamente arquivos ou classes reais do
código existente.
```

**2. Filtragem do que NÃO entra como requisito**, usado antes de escrever o PRD e o RFC:

```
A partir de TRANSCRICAO.md, separe em quatro listas: (a) requisitos
funcionais fechados na reunião; (b) itens explicitamente descartados;
(c) itens explicitamente adiados para fase futura; (d) pontos levantados
mas não decididos. NÃO inclua itens de (b), (c) ou (d) como requisito
funcional em nenhum documento. Eles só podem aparecer na seção "Fora de
escopo" do PRD ou em "Questões em aberto" no RFC, cada um citando o
timestamp exato em que foi descartado ou adiado.
```

## Iterações e Ajustes

O processo não foi gerar tudo de uma vez — a revisão final contra a checklist de critérios de aceite encontrou e corrigiu pelo menos três problemas concretos:

1. **Contagem incorreta no Tracker.** A linha de resumo do `TRACKER.md` afirmava "11 itens com fonte CODIGO", mas a contagem manual das linhas da tabela resultou em 12. O número foi recalculado e corrigido (72 TRANSCRICAO / 12 CODIGO, de 84 linhas), evitando que o documento afirmasse uma métrica de cobertura errada sobre si mesmo.
2. **Contratos HTTP com request implícito.** Na primeira versão do FDD, endpoints como `DELETE`, `GET` e os `POST` sem corpo (rotacionar secret, replay de DLQ) mostravam apenas o `Response`, sem indicar explicitamente a ausência de corpo na requisição. Isso deixava ambíguo se o documento realmente cobria "request e response" para os 4+ endpoints exigidos pelo critério de aceite. Foi adicionada uma linha `Request:` explícita em cada um desses endpoints.
3. **Erros inventados vs. erros derivados.** Ao montar a matriz `WEBHOOK_*`, dois códigos (`WEBHOOK_EVENT_LIST_EMPTY`, `WEBHOOK_INVALID_EVENT_TYPE`) não têm uma fala literal na transcrição nomeando o código — eles foram incluídos porque são consequência direta de um requisito explícito (o webhook precisa ter uma lista de status de interesse, e ela precisa ser válida). Em vez de omitir essa origem ou inventar uma citação que não existe, o Tracker marca essas linhas com o trecho da reunião que embasa a regra de negócio, mantendo a rastreabilidade honesta mesmo quando o nome exato do código é uma decisão de implementação e não uma fala literal.

## Como Navegar a Entrega

Ordem de leitura sugerida (da decisão mais atômica para a mais alto nível, e por fim a rastreabilidade):

1. [`docs/adrs/`](docs/adrs/) — as 7 decisões arquiteturais isoladas (outbox, worker, retry/DLQ, HMAC, at-least-once, reuso de padrões, snapshot de payload).
2. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica consolidada, alternativas descartadas e questões em aberto.
3. [`docs/FDD.md`](docs/FDD.md) — o detalhamento de implementação: fluxos, contratos HTTP, matriz de erros, observabilidade e a seção de integração com o código existente.
4. [`docs/PRD.md`](docs/PRD.md) — a visão de produto/negócio: problema, escopo, métricas de sucesso e riscos.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — a tabela de rastreabilidade cruzando cada item dos quatro documentos acima com `TRANSCRICAO.md` ou com um arquivo real do código.
6. [`TRANSCRICAO.md`](TRANSCRICAO.md) — a fonte primária, preservada sem alterações.
