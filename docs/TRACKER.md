# Tracker de Rastreabilidade

Mapeia cada requisito, decisão, restrição ou trade-off registrado nos documentos à sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte da aplicação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10s (meta p95) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Redução de polling em GET /orders pelos clientes piloto | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Zero incidentes de status sem evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Notificação por e-mail em falhas recorrentes adiada | TRANSCRICAO | [09:37] Marcos / Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Dashboard visual para o cliente fora de escopo | TRANSCRICAO | [09:39]–[09:40] Marcos / Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Rate limiting de envio não decidido, observar depois | TRANSCRICAO | [09:38]–[09:39] Diego / Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Trade-off | Exactly-once descartado em favor de at-least-once | TRANSCRICAO | [09:24]–[09:25] Diego |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Ordenação global e escala multi-worker fora de escopo | TRANSCRICAO | [09:12]–[09:13] Diego / Bruno / Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL + status) com secret gerada na criação | TRANSCRICAO | [09:31]–[09:32] Marcos / Bruno |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtragem por status de interesse na inserção do evento | TRANSCRICAO | [09:33]–[09:34] Marcos / Bruno / Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via endpoint admin, com auditoria | TRANSCRICAO | [09:18]–[09:19], [09:35]–[09:36] Diego / Sofia / Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload | TRANSCRICAO | [09:19]–[09:20] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Entrega at-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:24]–[09:25] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reentrega automática com backoff antes de falha permanente | TRANSCRICAO | [09:14]–[09:17] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | URL do webhook deve ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, erro se exceder | TRANSCRICAO | [09:23]–[09:24] Sofia / Diego / Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP de entrega | TRANSCRICAO | [09:42] Sofia / Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Processamento assíncrono não pode impactar a transação de changeStatus | TRANSCRICAO | [09:03]–[09:04] Larissa / Bruno |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Replay administrativo deve ser auditável | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-06 | docs/PRD.md | Restrição | Reaproveitar padrões existentes de erro/log/módulos, sem novos mecanismos | TRANSCRICAO | [09:29] Bruno |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso na entrega pode levar Atlas Comercial a migrar de fornecedor | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret já ocorreu antes com um cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente pode implementar dedup por X-Event-Id incorretamente | TRANSCRICAO | [09:25]–[09:26] Sofia / Marcos |
| PRD-RISK-04 | docs/PRD.md | Risco | Pico de eventos por customer sem rate limiting | TRANSCRICAO | [09:38]–[09:39] Diego / Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado (trava mudança de status de outros pedidos) | TRANSCRICAO | [09:03]–[09:04] Larissa / Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila externa (Redis Streams) descartada por overengineering | TRANSCRICAO | [09:06]–[09:07] Larissa / Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Rate limiting de envio não decidido | TRANSCRICAO | [09:38]–[09:39] Diego / Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Notificação de falha recorrente adiada para fase futura | TRANSCRICAO | [09:37] Marcos / Larissa |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Escala multi-worker e ordenação global não definidas | TRANSCRICAO | [09:12]–[09:13] Diego / Bruno |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo de três sprints incluindo revisão de segurança da Sofia | TRANSCRICAO | [09:45]–[09:47] Larissa / Sofia |
| FDD-MODEL-01 | docs/FDD.md | Decisão | Chaves das novas tabelas em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-MODEL-02 | docs/FDD.md | Decisão | Payload snapshot armazenado no momento da inserção na outbox | TRANSCRICAO | [09:51]–[09:52] Bruno / Larissa / Diego |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação de changeStatus, com filtro por status | TRANSCRICAO | [09:33]–[09:34], [09:40]–[09:41] Bruno / Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Worker em polling de 2s processa pendentes em lote | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Replay manual recoloca item da DLQ como pendente | TRANSCRICAO | [09:18]–[09:19] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /customers/:customerId/webhooks | TRANSCRICAO | [09:31]–[09:32] Marcos / Bruno / Larissa |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | GET /customers/:customerId/webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | PATCH /webhooks/:webhookId | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | DELETE /webhooks/:webhookId | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | POST /webhooks/:webhookId/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | GET /webhooks/:webhookId/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18]–[09:19], [09:35] Diego / Larissa |
| FDD-CONTRATO-08 | docs/FDD.md | Requisito Funcional | Payload e headers de saída (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id) | TRANSCRICAO | [09:43]–[09:45] Diego / Sofia |
| FDD-ERR-01 | docs/FDD.md | Decisão | Código WEBHOOK_NOT_FOUND, prefixo WEBHOOK_ nos erros do módulo | TRANSCRICAO | [09:28]–[09:29] Bruno / Larissa |
| FDD-ERR-02 | docs/FDD.md | Restrição | WEBHOOK_EVENT_LIST_EMPTY (lista de status obrigatória) | TRANSCRICAO | [09:33]–[09:34] Marcos / Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição | WEBHOOK_INVALID_EVENT_TYPE (status deve existir no enum OrderStatus) | TRANSCRICAO | [09:33]–[09:34] Marcos / Bruno |
| FDD-ERR-04 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_PAYLOAD_TOO_LARGE (limite de 64KB) | TRANSCRICAO | [09:23]–[09:24] Sofia / Diego / Larissa |
| FDD-ERR-05 | docs/FDD.md | Restrição | WEBHOOK_SECRET_ROTATION_IN_PROGRESS (grace period de rotação) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-06 | docs/FDD.md | Restrição | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18]–[09:19] Diego |
| FDD-ERR-07 | docs/FDD.md | Restrição | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:18]–[09:19] Diego |
| FDD-INTEG-01 | docs/FDD.md | Restrição | Integração via OrderService.changeStatus (publishWebhookEvent) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Reuso de BadRequestError/ConflictError com código customizado | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Middleware de erro centralizado tratará AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | requireRole('ADMIN') reaproveitado no endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | paginated()/buildPagination() reaproveitados nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Novas tabelas seguem convenção de UUID/índices do schema atual | CODIGO | prisma/schema.prisma |
| FDD-INTEG-07 | docs/FDD.md | Restrição | Worker e módulo reaproveitam a instância Pino já configurada | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Rotas do módulo registradas em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-RES-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP | TRANSCRICAO | [09:42] Sofia / Diego |
| FDD-RES-02 | docs/FDD.md | Decisão | Retry com 5 tentativas e backoff exponencial | TRANSCRICAO | [09:15]–[09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Decisão | Fallback é DLQ manual, sem automação de reenvio | TRANSCRICAO | [09:18]–[09:19], [09:37] Diego / Larissa |
| FDD-RES-04 | docs/FDD.md | Restrição | Falha de um endpoint não afeta outros nem a API principal | TRANSCRICAO | [09:04], [09:11] Bruno / Diego |
| FDD-OBS-01 | docs/FDD.md | Restrição | Logs estruturados reaproveitando Pino, sem novo mecanismo | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Decisão | event_id como identificador de correlação, análogo ao X-Request-Id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-03 | docs/FDD.md | Risco | Necessidade de monitorar volume/performance da outbox | TRANSCRICAO | [09:07]–[09:08] Bruno / Diego |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite | Rollback atômico se inserção na outbox falhar | TRANSCRICAO | [09:40]–[09:41] Bruno / Diego |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite | Suíte de testes de pedidos existente não pode regredir | CODIGO | tests/orders.test.ts |
| FDD-RISK-01 | docs/FDD.md | Risco | Worker indisponível sem supervisão pausa toda a entrega | TRANSCRICAO | [09:10]–[09:11] Diego / Larissa |
| FDD-RISK-02 | docs/FDD.md | Risco | Outbox/deliveries crescem sem arquivamento nesta fase | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Cliente implementa HMAC ou dedup incorretamente | TRANSCRICAO | [09:25]–[09:26] Sofia / Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Outbox pattern no MySQL dentro da transação de changeStatus | TRANSCRICAO | [09:06]–[09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-dedicado-polling.md | Decisão | Worker dedicado em processo separado, polling de 2s | TRANSCRICAO | [09:09]–[09:11] Diego / Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-dead-letter-queue.md | Decisão | Retry com backoff exponencial e DLQ em tabela separada | TRANSCRICAO | [09:14]–[09:19] Diego |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period | TRANSCRICAO | [09:19]–[09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-deduplicacao-x-event-id.md | Decisão | Entrega at-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:24]–[09:26] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Decisão | Módulo de webhooks reaproveita padrões arquiteturais existentes | TRANSCRICAO | [09:27]–[09:30] Bruno / Diego / Larissa |
| ADR-006-CODIGO | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Referência explícita a AppError e hierarquia de erros existente | CODIGO | src/shared/errors/app-error.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-insercao-do-outbox.md | Decisão | Snapshot do payload renderizado na inserção da outbox | TRANSCRICAO | [09:51]–[09:53] Bruno / Larissa / Diego |

**Cobertura:** 84 itens rastreados — 72 com fonte TRANSCRICAO (≈86%) e 12 com fonte CODIGO (caminhos reais verificados no repositório, ver seção de verificação no README).
