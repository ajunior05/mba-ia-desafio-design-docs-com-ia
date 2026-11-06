# PRD — Product Requirement Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O OMS hoje não notifica proativamente nenhum sistema externo sobre mudanças de status de pedido. O Sistema de Webhooks de Notificação de Pedidos passa a enviar, automaticamente, uma notificação HTTP para endpoints cadastrados pelos clientes B2B sempre que o status de um pedido deles muda — eliminando a necessidade de polling manual no `GET /orders`.

## Problema e Motivação

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — fizeram um pedido formal para serem notificados em tempo real sobre mudanças de status de seus pedidos. Hoje eles fazem polling periódico em `GET /orders`, o que torna a integração lenta e cara do lado deles. A **Atlas Comercial sinalizou risco de migrar para um concorrente** caso a feature não seja entregue até o fim do trimestre.

Para os clientes, "tempo real" significa qualquer latência **abaixo de 10 segundos** entre a mudança de status e o recebimento da notificação — não é necessário ser instantâneo, apenas eliminar a necessidade de atualização manual constante.

## Público-alvo e Cenários de Uso

**Público-alvo:** clientes B2B integrados à plataforma via API (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), representados por usuários operadores já autenticados no sistema (JWT existente) — não há, nesta fase, autocadastro pelo próprio cliente final fora da nossa API.

**Cenários de uso:**
1. Um operador cadastra, via API, um endpoint de webhook para um customer, informando a URL e os status de pedido de interesse (ex.: apenas `SHIPPED` e `DELIVERED`).
2. O status de um pedido desse customer muda; o cliente recebe uma notificação HTTP assinada em até 10 segundos.
3. O endpoint do cliente está temporariamente fora do ar; o sistema tenta novamente de forma automática por até ~15 horas antes de desistir.
4. Um administrador consulta e reprocessa manualmente notificações que falharam definitivamente.
5. Um operador consulta o histórico de entregas de um webhook para auditar sucesso/falha das notificações enviadas.

## Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Eliminar a necessidade de polling manual pelos clientes B2B | Latência entre a mudança de status do pedido e a entrega bem-sucedida do webhook | Abaixo de 10 segundos no percentil 95 das entregas |
| Reduzir o custo de integração do cliente | Volume de chamadas de polling em `GET /orders` pelos três clientes piloto (Atlas, MaxDistribuição, Nova Cargo) | Redução de pelo menos 80% em 30 dias após adoção |
| Garantir confiabilidade da notificação | Incidentes de mudança de status sem evento correspondente registrado | Zero incidentes (garantido pela atomicidade transacional do outbox) |

## Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por customer (URL + lista de status de interesse).
- Geração de secret na criação do webhook e endpoint de rotação de secret.
- Entrega assinada (HMAC-SHA256) com identificador único de evento para deduplicação pelo cliente.
- Processamento assíncrono via outbox + worker dedicado, com retry automático e Dead Letter Queue (DLQ).
- Endpoint administrativo para reprocessamento manual de itens da DLQ.
- Histórico de entregas por webhook (sucesso/falha, payload, resposta, tempo de resposta).

### Fora de Escopo
- **Notificação ao cliente sobre falhas recorrentes de entrega (ex.: e-mail).** Explicitamente adiado para uma fase futura, após medir o impacto real do problema em produção.
- **Dashboard visual para o cliente acompanhar seus webhooks.** Descartado para esta fase; considerado projeto separado do time de frontend.
- **Rate limiting de envio por destino.** Não entra nesta fase; o time decidiu observar o comportamento em produção antes de desenhar uma solução.
- **Garantia de entrega exactly-once.** Descartada — adotado at-least-once com deduplicação por `X-Event-Id`, por ser o padrão de mercado e evitar a complexidade de coordenação entre as duas partes.
- **Escala horizontal do worker (múltiplos workers em paralelo) e ordenação global de eventos entre pedidos diferentes.** Registrado como limitação conhecida; ordenação é garantida apenas por `order_id`, enquanto o sistema operar com um único worker.

## Requisitos Funcionais

1. O sistema permite cadastrar um webhook para um customer, informando URL e lista de status de pedido de interesse; a secret é gerada pela plataforma e devolvida apenas no momento da criação.
2. O sistema permite editar um webhook existente (URL, lista de status, status ativo/inativo).
3. O sistema permite remover um webhook existente.
4. O sistema permite listar os webhooks cadastrados de um customer.
5. O sistema envia a notificação apenas para webhooks cujo filtro de status inclui o novo status do pedido (filtragem no momento da geração do evento).
6. O sistema permite consultar o histórico de entregas de um webhook (sucesso/falha, payload enviado, resposta recebida, tempo de resposta).
7. O sistema permite a um administrador reprocessar manualmente um item da Dead Letter Queue, e registra em log quem executou essa ação.
8. O sistema assina cada notificação enviada com HMAC-SHA256, permitindo ao cliente validar autenticidade e integridade do payload.
9. O sistema permite a rotação da secret de um webhook, mantendo a secret anterior válida por 24 horas em paralelo.
10. O sistema garante entrega at-least-once e inclui um identificador único por evento (`X-Event-Id`) para que o cliente deduplique notificações repetidas.
11. O sistema tenta reentregar uma notificação que falhou, usando backoff exponencial, antes de considerá-la falha permanente.

## Requisitos Não Funcionais

- A URL de um webhook deve ser obrigatoriamente HTTPS; URLs HTTP são rejeitadas na validação.
- O payload de uma notificação não pode exceder 64KB; eventos que excedem o limite falham explicitamente (sem truncamento silencioso).
- Cada chamada HTTP de entrega tem timeout de 10 segundos.
- O processamento de notificações não pode adicionar latência perceptível à transação de mudança de status do pedido (processamento assíncrono, desacoplado via outbox).
- Toda ação administrativa sobre a DLQ deve ser auditável (log de quem executou o replay).
- A feature deve reaproveitar os padrões de erro, logging e estrutura de módulos já existentes no projeto, sem introduzir novos mecanismos equivalentes.

## Decisões e Trade-offs Principais

- **Outbox no MySQL em vez de fila externa dedicada:** evita subir nova infraestrutura, ao custo de depender de um worker em polling em vez de um modelo reativo (ver [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)).
- **Worker em processo separado, não embutido na API:** garante resiliência a reinícios da API, à custa de uma peça operacional adicional para manter ativa (ver [ADR-002](adrs/ADR-002-worker-dedicado-polling.md)).
- **5 tentativas com backoff até 12h, depois DLQ manual:** cobre indisponibilidades reais já observadas em clientes, à custa de exigir intervenção humana para falhas permanentes (ver [ADR-003](adrs/ADR-003-retry-backoff-dead-letter-queue.md)).
- **Secret por endpoint com rotação, em vez de secret global:** limita o raio de impacto de um vazamento, à custa de maior complexidade de gestão de secrets (ver [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md)).
- **At-least-once em vez de exactly-once:** mantém o sistema simples, transferindo a responsabilidade de deduplicação para o cliente (ver [ADR-005](adrs/ADR-005-at-least-once-deduplicacao-x-event-id.md)).

## Dependências

- Infraestrutura de banco MySQL já existente (sem necessidade de novo serviço de infraestrutura).
- Estrutura de módulos, autenticação JWT (`requireRole`) e tratamento de erros já existentes no projeto.
- Revisão de segurança da engenheira Sofia antes do deploy em produção (mínimo de dois dias úteis reservados), com foco em HMAC e geração/rotação de secret.
- Comunicação do Product Manager com os clientes piloto sobre prazo e sobre a responsabilidade deles de implementar verificação HMAC e deduplicação.

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso na entrega leva a Atlas Comercial migrar para um concorrente | Média | Alto | Prazo estimado em três sprints (incluindo revisão de segurança) já definido com a Tech Lead; PM confirma o prazo diretamente com o cliente |
| Vazamento de secret de um cliente (já ocorreu antes, em log de aplicação do lado do cliente) compromete a integração | Média | Médio | Secret única por endpoint (não global) e suporte a rotação com grace period de 24h, limitando o raio de impacto a um único endpoint |
| Cliente não implementa corretamente a deduplicação por `X-Event-Id`, processando eventos duplicados do lado dele | Média | Médio | Documentação destacada no portal do desenvolvedor, de responsabilidade do PM |
| Pico de mudanças de status em um customer gera volume alto de chamadas simultâneas para o endpoint dele, sem controle de taxa | Baixa | Médio | Rate limiting fica fora de escopo nesta fase; comportamento será observado em produção antes de se desenhar uma solução |

## Critérios de Aceitação

- Um cliente consegue cadastrar, editar, listar e remover webhooks via API, recebendo a secret apenas no momento da criação.
- Uma mudança de status de pedido gera uma notificação ao(s) webhook(s) do customer que escutam aquele status, entregue em até 10 segundos em condições normais.
- Uma notificação é assinada com HMAC-SHA256 e inclui um `X-Event-Id` único, verificável pelo cliente.
- Uma falha de entrega é reentregue automaticamente seguindo o backoff definido, e movida para DLQ após 5 tentativas.
- Um administrador consegue reprocessar manualmente um item da DLQ, com a ação registrada em log.
- Um usuário autenticado consegue consultar o histórico de entregas de um webhook específico.
- Nenhum requisito, decisão ou restrição desta feature contradiz o que foi definido na reunião técnica de kickoff (`TRANSCRICAO.md`).

## Estratégia de Testes e Validação

- **Testes automatizados** cobrindo o módulo de webhooks no mesmo padrão de `tests/orders.test.ts` (ex.: `tests/webhooks.test.ts`), incluindo: CRUD de configuração, filtragem por status na geração do evento, geração e verificação de assinatura HMAC, rotação de secret com grace period, e transição de uma `webhook_delivery` por todas as tentativas de retry até a DLQ.
- **Teste de atomicidade:** forçar uma falha na inserção da outbox dentro de `OrderService.changeStatus` e validar que a mudança de status sofre rollback (garantia central do ADR-001).
- **Teste de regressão:** garantir que a suíte existente (`tests/orders.test.ts`, `tests/auth.test.ts`) continua passando sem alteração de contrato nos endpoints já existentes.
- **Revisão de segurança dedicada** (Sofia), antes do deploy em produção, focada em geração/validação de HMAC e geração/rotação de secret — reservada explicitamente na estimativa de prazo.
- **Validação com clientes piloto:** Atlas Comercial, MaxDistribuição e Nova Cargo como primeiros consumidores reais da feature, usados para confirmar as métricas de sucesso (latência e redução de polling) após o lançamento.
