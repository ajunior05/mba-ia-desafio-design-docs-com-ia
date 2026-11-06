# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | Reunião técnica de kickoff (ver `TRANSCRICAO.md`) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Engenharia de Segurança) |

## TL;DR

Vamos notificar clientes B2B sobre mudanças de status de pedido via webhooks outbound, usando o padrão **Outbox** no MySQL já existente (inserido na mesma transação de `OrderService.changeStatus`) e um **worker dedicado** que faz polling a cada 2 segundos e entrega via HTTP com **retry exponencial + DLQ**. Entrega é **at-least-once**, autenticada por **HMAC-SHA256 com secret por endpoint**. O módulo reaproveita integralmente os padrões já existentes do projeto (estrutura de módulos, erros, logging). Nenhuma infraestrutura nova é introduzida.

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) hoje fazem polling manual em `GET /orders` para detectar mudanças de status, o que torna a integração lenta e cara do lado deles. Eles pediram notificação em até 10 segundos após a mudança de status, sob risco de a Atlas Comercial migrar para um concorrente caso a feature não saia até o fim do trimestre.

A aplicação não possui hoje nenhum mecanismo de evento, fila ou notificação externa — este vácuo é o problema a resolver.

## Proposta Técnica

A mudança de status de um pedido (`OrderService.changeStatus`) passa a, dentro da mesma transação SQL já existente, inserir um registro de evento em uma nova tabela de outbox (`webhook_outbox`), com o payload já renderizado (snapshot do estado no momento da transição).

Um processo separado (worker, entry-point próprio) faz polling dessa tabela a cada 2 segundos, processa os eventos pendentes em lote e realiza a chamada HTTP para os endpoints de webhook cadastrados pelo customer, filtrados pelos status que cada endpoint deseja receber.

Cada entrega é assinada com HMAC-SHA256 (secret própria por endpoint, rotacionável com grace period) e carrega um identificador único de evento para o cliente deduplicar do lado dele, já que a garantia de entrega é at-least-once. Falhas de entrega seguem uma política de retry com backoff exponencial; eventos que esgotam as tentativas vão para uma tabela de dead-letter, reprocessável manualmente por um endpoint administrativo.

O módulo (`src/modules/webhooks`) segue a mesma estrutura dos demais módulos do projeto (controller/service/repository/routes/schemas), reaproveitando a hierarquia de erros (`AppError`), o logger (Pino) e o middleware de erro centralizado já existentes — sem necessidade de alteração nesses componentes compartilhados.

O detalhamento de contratos HTTP, fluxos passo a passo, matriz de erros e pontos exatos de integração com o código está no FDD (`docs/FDD.md`). As decisões individuais abaixo estão formalizadas em ADRs (`docs/adrs/`).

## Alternativas Consideradas

### 1. Disparo síncrono dentro de `OrderService.changeStatus`
A opção mais direta seria chamar o endpoint do cliente de dentro da própria transação que muda o status do pedido.

**Trade-off que motivou o descarte:** a transação de mudança de status já é pesada (atualiza `orders`, insere em `order_status_history`, ajusta `stock_quantity`). Acoplar uma chamada HTTP de rede a essa transação significa que um cliente lento ou indisponível trava a mudança de status de *outros* pedidos, e não há como fazer rollback de uma notificação já enviada se o cliente cair no meio do processo.

### 2. Fila externa dedicada (ex.: Redis Streams)
Uma fila de mensagens dedicada resolveria o desacoplamento entre a mudança de status e a entrega do webhook, de forma mais "canônica" que uma tabela outbox.

**Trade-off que motivou o descarte:** exigiria subir e operar uma nova peça de infraestrutura para um time pequeno, sem ganho proporcional frente ao outbox no MySQL já disponível, que resolve o mesmo problema de atomicidade sem infraestrutura nova.

## Questões em Aberto

1. **Rate limiting de envio para o cliente.** Se um customer tiver muitos pedidos mudando de status em um curto intervalo (ex.: 50 em um minuto), o sistema hoje envia todas as chamadas sem controle de taxa por destino. A equipe decidiu observar o comportamento em produção antes de desenhar uma solução — não há decisão tomada.
2. **Notificação ao cliente sobre falhas recorrentes de entrega.** Foi perguntado se o sistema deveria avisar o cliente (ex.: por e-mail) quando o webhook dele falha repetidamente. Foi explicitamente adiado para uma fase futura, após medir o impacto real do problema.
3. **Escala para múltiplos workers em paralelo.** O desenho atual assume um único worker processando a outbox, o que garante ordenação de entrega por `order_id`. Se for necessário escalar para múltiplos workers, estratégias como particionamento por `order_id` ou lock pessimista foram mencionadas como caminhos possíveis, mas nenhuma foi definida — fica registrado como problema a resolver somente se e quando a necessidade aparecer.

## Impacto e Riscos

- **Impacto no código existente:** o único ponto crítico de modificação em código já existente é `OrderService.changeStatus`, que passa a publicar um evento dentro da transação atual. Os demais módulos (`auth`, `customers`, `products`) não são afetados.
- **Risco operacional:** o worker roda como processo independente da API; falha em mantê-lo ativo (ex.: ausência de supervisão/restart automático) pausa toda a entrega de webhooks sem afetar a API principal — risco a ser coberto por observabilidade (ver FDD).
- **Risco de segurança:** vazamento de secret de um endpoint de cliente compromete apenas aquele endpoint, dado o modelo de secret por endpoint (ADR-004) — mitigação já incorporada ao desenho, não um risco residual do RFC.
- **Risco de prazo:** a estimativa inicial da Tech Lead é de três sprints, incluindo a revisão de segurança da Sofia antes do deploy (mínimo dois dias úteis reservados para essa revisão).

## Decisões Relacionadas

- [ADR-001 — Outbox Pattern no MySQL](adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Worker dedicado em processo separado, com polling de 2 segundos](adrs/ADR-002-worker-dedicado-polling.md)
- [ADR-003 — Retry com backoff exponencial e Dead Letter Queue](adrs/ADR-003-retry-backoff-dead-letter-queue.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Entrega at-least-once com deduplicação via X-Event-Id](adrs/ADR-005-at-least-once-deduplicacao-x-event-id.md)
- [ADR-006 — Reuso dos padrões arquiteturais existentes do projeto](adrs/ADR-006-reuso-padroes-existentes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção do outbox](adrs/ADR-007-snapshot-payload-na-insercao-do-outbox.md)
