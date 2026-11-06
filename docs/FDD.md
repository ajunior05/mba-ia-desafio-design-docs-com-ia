# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e Motivação Técnica

O OMS hoje não possui nenhum mecanismo de evento, fila ou notificação externa. Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser avisados em até 10 segundos quando o status de um pedido muda, hoje descoberto apenas via polling manual em `GET /orders`.

A decisão arquitetural (outbox + worker dedicado + retry/DLQ + HMAC + at-least-once) está registrada nos ADRs em `docs/adrs/`. Este documento detalha **como construir** essa decisão: fluxos passo a passo, contratos HTTP, modelagem de dados, matriz de erros, resiliência, observabilidade e pontos exatos de integração com o código existente.

## 2. Objetivos Técnicos

- Publicar um evento de forma transacionalmente consistente com a mudança de status do pedido (zero perda, zero evento "fantasma").
- Entregar o evento ao endpoint do cliente em até 10 segundos no caso comum (meta de negócio), com piso de latência de 2 segundos (ciclo de polling do worker).
- Garantir que uma falha de entrega não bloqueie o processamento de outros pedidos nem a API principal.
- Permitir ao cliente validar autenticidade e integridade do payload recebido.
- Reaproveitar 100% dos padrões de erro, log e estrutura de módulo já existentes no projeto.

## 3. Escopo e Exclusões

**Dentro do escopo:** outbox, worker de polling, retry+backoff, DLQ com replay administrativo, CRUD de configuração de webhook, histórico de entregas, HMAC-SHA256 com rotação de secret, payload enxuto por evento.

**Fora de escopo desta fase** (ver PRD para detalhamento de negócio):
- Notificação (ex.: e-mail) ao cliente sobre falhas recorrentes de entrega.
- Rate limiting de envio por destino.
- Dashboard visual para o cliente.
- Escala horizontal do worker (múltiplos workers em paralelo) e ordenação global de eventos.
- Arquivamento/retenção de linhas antigas da outbox.

## 4. Modelagem de Dados (nova)

Seguindo a convenção do projeto (`@id @default(uuid()) @db.Char(36)`, `@@map`, índices em colunas de filtro frequente — ver `prisma/schema.prisma`):

- **`webhook_endpoints`**: `id`, `customer_id`, `url`, `secret`, `previous_secret` (nullable), `previous_secret_expires_at` (nullable), `events` (lista de `OrderStatus` aceitos), `active`, `created_at`, `updated_at`. Índice em `customer_id` e `active`.
- **`webhook_outbox`**: `id`, `order_id`, `customer_id`, `event_type` (`order.status_changed`), `from_status`, `to_status`, `payload` (JSON, snapshot — ADR-007), `status` (`pending` | `processing` | `delivered`), `created_at`. Índice em `status` e `created_at` (consulta do worker).
- **`webhook_deliveries`**: `id`, `outbox_event_id`, `webhook_endpoint_id`, `attempt`, `status` (`pending` | `success` | `failed` | `dead_letter`), `status_code` (nullable), `response_time_ms` (nullable), `error_message` (nullable), `next_retry_at` (nullable), `delivered_at` (nullable), `created_at`. Índice em `webhook_endpoint_id` (histórico por endpoint) e `status`+`next_retry_at` (fila de retry do worker).
- **`webhook_dead_letter`**: `id`, `delivery_id`, `payload`, `reason`, `created_at`. Índice em `created_at`.

Racional: uma linha de `webhook_outbox` representa o evento (independente de destino); uma linha de `webhook_deliveries` representa uma tentativa de entrega a um endpoint específico, permitindo que cada endpoint tenha seu próprio estado de retry/backoff independente — necessário porque um customer pode ter mais de um webhook cadastrado ouvindo o mesmo status.

## 5. Fluxos Detalhados

### 5.1 Criação do evento na outbox (dentro de `OrderService.changeStatus`)

1. `OrderService.changeStatus` muda o status do pedido dentro de `prisma.$transaction` (já existente).
2. Imediatamente após o `tx.order.update(...)` e o `tx.orderStatusHistory.create(...)` já existentes, chama `publishWebhookEvent(tx, order, from, to)`.
3. `publishWebhookEvent` busca os `webhook_endpoints` ativos do `customerId` do pedido cujo campo `events` contém o novo status (`to`).
4. Se **nenhum** endpoint ativo escuta aquele status, a função retorna sem inserir nada (filtro na inserção, não no envio — decisão explícita da reunião).
5. Se houver ao menos um endpoint interessado, monta o payload (snapshot, ver seção 6.2) e insere **uma linha em `webhook_outbox`** com `status = 'pending'`.
6. Se a inserção falhar, a exceção propaga e toda a transação (incluindo a mudança de status) sofre rollback — não existe cenário de status mudado sem evento registrado.

### 5.2 Processamento pelo worker (`src/worker.ts` + `webhook.processor.ts`)

1. Loop contínuo, intervalo de 2 segundos.
2. `SELECT * FROM webhook_outbox WHERE status = 'pending' ORDER BY created_at ASC LIMIT <batch_size>` (ex.: 20).
3. Para cada evento pendente, marca `status = 'processing'` e resolve os `webhook_endpoints` ativos que escutam `to_status` daquele evento (mesmo filtro da inserção, reavaliado para capturar endpoints criados/desativados entre a inserção e o processamento).
4. Para cada endpoint resolvido, cria (se não existir) uma linha em `webhook_deliveries` com `attempt = 1`.
5. Executa a chamada HTTP (ver contratos de saída, seção 6.3) com timeout de 10 segundos.
6. Sucesso (2xx): marca a `webhook_delivery` como `success`, grava `status_code` e `response_time_ms`.
7. Falha (timeout, erro de rede, status ≠ 2xx): aplica a política de retry (seção 7).
8. Quando **todas** as `webhook_deliveries` de um evento da outbox estiverem em estado terminal (`success`, `failed` ou `dead_letter`), marca a linha correspondente em `webhook_outbox` como `delivered` (processada, independentemente do resultado final de cada destino).

### 5.3 Retry

1. Em cada falha, incrementa `attempt` e calcula `next_retry_at` conforme a tabela de backoff (ADR-003): tentativa 1 falhou → próxima em 1 minuto; 2 → 5 minutos; 3 → 30 minutos; 4 → 2 horas; 5 → 12 horas.
2. O worker, em cada ciclo de polling, também seleciona `webhook_deliveries` com `status = 'failed'` e `next_retry_at <= now()` para reprocessar, além dos eventos novos da outbox.
3. Após a 5ª tentativa falhar, a `webhook_delivery` é marcada `dead_letter` e uma linha correspondente é criada em `webhook_dead_letter`, com o payload, o motivo da última falha e o timestamp.

### 5.4 DLQ e Replay Manual

1. Um item em `webhook_dead_letter` só é reprocessado manualmente, via `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`).
2. O replay cria uma nova `webhook_delivery` com `attempt = 1` e `status = 'pending'` para o mesmo par (evento, endpoint), e registra em log (Pino) o `userId` do administrador que executou o replay, para auditoria.

## 6. Contratos Públicos

Todos os endpoints abaixo exigem `Authorization: Bearer <jwt>` (middleware `authenticate` já existente), exceto onde indicado `requireRole('ADMIN')`.

### 6.1 `POST /customers/:customerId/webhooks` — Cadastrar webhook

Request:
```json
{
  "url": "https://api.atlascomercial.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "b2f1c6d2-8e2a-4b31-9a7a-3e6a5a1f9c01",
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://api.atlascomercial.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_3f29a7c1e4b8d9f0a1b2c3d4e5f6a7b8",
  "active": true,
  "createdAt": "2026-11-05T10:00:00.000Z"
}
```
A `secret` é devolvida **apenas na criação**. Erros possíveis: `WEBHOOK_EVENT_LIST_EMPTY`, `WEBHOOK_INVALID_EVENT_TYPE`, `VALIDATION_ERROR` (URL não HTTPS, via Zod), `NOT_FOUND` (customer inexistente, reaproveitando `NotFoundError('Customer')` igual a `order.service.ts`).

### 6.2 `GET /customers/:customerId/webhooks` — Listar webhooks do customer

Request: sem corpo; `customerId` via path param, paginação via query string (`?page=1&pageSize=20`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "b2f1c6d2-8e2a-4b31-9a7a-3e6a5a1f9c01",
      "customerId": "550e8400-e29b-41d4-a716-446655440000",
      "url": "https://api.atlascomercial.com/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-05T10:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Segue o mesmo formato de `paginated()` em `src/shared/http/response.ts`. A `secret` nunca é retornada em listagens.

### 6.3 `PATCH /webhooks/:webhookId` — Editar webhook

Request (campos opcionais):
```json
{ "url": "https://api.atlascomercial.com/webhooks/v2/orders", "events": ["SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```
Response `200 OK`: mesmo formato do item da listagem (sem `secret`). Erros: `WEBHOOK_NOT_FOUND`, `WEBHOOK_EVENT_LIST_EMPTY`, `WEBHOOK_INVALID_EVENT_TYPE`, `VALIDATION_ERROR`.

### 6.4 `DELETE /webhooks/:webhookId` — Remover webhook

Request: sem corpo; `webhookId` via path param.

Response: `204 No Content`. Erros: `WEBHOOK_NOT_FOUND`.

### 6.5 `POST /webhooks/:webhookId/rotate-secret` — Rotacionar secret

Request: sem corpo; `webhookId` via path param.

Response `200 OK`:
```json
{
  "secret": "whsec_9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d",
  "previousSecretValidUntil": "2026-11-06T10:00:00.000Z"
}
```
A secret anterior permanece válida por 24h (ADR-004). Erros: `WEBHOOK_NOT_FOUND`, `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` (já existe rotação em grace period ativo).

### 6.6 `GET /webhooks/:webhookId/deliveries` — Histórico de entregas

Request: sem corpo; `webhookId` via path param, paginação via query string (`?page=1&pageSize=20`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "f1a2b3c4-...-uuid",
      "eventId": "a1b2c3d4-...-uuid",
      "eventType": "order.status_changed",
      "toStatus": "SHIPPED",
      "attempt": 1,
      "status": "success",
      "statusCode": 200,
      "responseTimeMs": 184,
      "deliveredAt": "2026-11-05T10:00:02.100Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Erros: `WEBHOOK_NOT_FOUND`.

### 6.7 `POST /admin/webhooks/dead-letter/:id/replay` — Replay manual (requer `requireRole('ADMIN')`)

Request: sem corpo; `id` (do item de dead-letter) via path param.

Response `200 OK`:
```json
{ "deadLetterId": "d4e5f6a7-...-uuid", "newDeliveryId": "e5f6a7b8-...-uuid", "status": "requeued" }
```
Erros: `WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `UNAUTHORIZED`, `FORBIDDEN`.

### 6.8 Chamada de saída (plataforma → cliente)

```
POST <url cadastrada pelo cliente>
Content-Type: application/json
X-Event-Id: a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
X-Webhook-Id: b2f1c6d2-8e2a-4b31-9a7a-3e6a5a1f9c01
X-Signature: 7a1c9e3f...  (HMAC-SHA256 do corpo, hex)
X-Timestamp: 2026-11-05T10:00:00.000Z
```
Body:
```json
{
  "event_id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-05T10:00:00.000Z",
  "order_id": "9f8e7d6c-...-uuid",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "550e8400-e29b-41d4-a716-446655440000",
  "total_cents": 15990
}
```
Timeout: 10s. Resposta esperada: qualquer `2xx`. Qualquer outro status ou timeout é tratado como falha (ver seção 7).

## 7. Matriz de Erros (`WEBHOOK_*`)

Todos estendem `AppError` (`src/shared/errors/app-error.ts`), seguindo o mesmo padrão de `InvalidStatusTransitionError`/`InsufficientStockError` em `src/shared/errors/http-errors.ts`.

| Código | HTTP Status | Classe base | Quando ocorre |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `AppError` (dedicado — não reaproveita `NotFoundError` genérico, pelo prefixo explícito pedido na reunião) | Webhook config com o id informado não existe |
| `WEBHOOK_EVENT_LIST_EMPTY` | 400 | `BadRequestError` | Lista de `events` vazia na criação/edição |
| `WEBHOOK_INVALID_EVENT_TYPE` | 400 | `BadRequestError` | Status informado em `events` não existe no enum `OrderStatus` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload renderizado excede 64KB no momento da inserção na outbox |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Nova rotação solicitada antes do fim do grace period de 24h da rotação anterior |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` (dedicado) | Id de dead-letter informado não existe |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `ConflictError` | Replay solicitado para um item já reprocessado |

**Nota:** a validação de URL obrigatoriamente HTTPS é feita no schema Zod (não gera um código `WEBHOOK_*` dedicado) — conforme explicitamente definido na reunião ("não é decisão arquitetural, é só uma validação no schema"), surge como `VALIDATION_ERROR` genérico, igual ao tratamento já existente em `error.middleware.ts` para `ZodError`.

## 8. Estratégias de Resiliência

- **Timeout:** 10 segundos por chamada HTTP de entrega (worker → cliente).
- **Retry:** backoff exponencial, 5 tentativas (1m / 5m / 30m / 2h / 12h) — ADR-003.
- **Fallback:** após esgotar as tentativas, o item vai para `webhook_dead_letter`; não há fallback automático (ex.: e-mail) nesta fase — reprocessamento é manual via endpoint admin.
- **Isolamento de falha:** falha de entrega a um endpoint não afeta a entrega a outros endpoints do mesmo evento (cada `webhook_delivery` tem estado independente) nem a disponibilidade da API principal (worker é processo separado — ADR-002).
- **Fora de escopo nesta fase:** rate limiting de saída por destino e circuit breaker para endpoints cronicamente indisponíveis (ver RFC, "Questões em Aberto").

## 9. Observabilidade

- **Métricas (contadores/gauges a expor via logging estruturado, sem APM novo introduzido):** `webhook_outbox_pending_count`, `webhook_deliveries_success_total`, `webhook_deliveries_failed_total`, `webhook_dead_letter_total`, `webhook_worker_cycle_duration_ms`.
- **Logs:** via a mesma instância Pino do projeto (`src/shared/logger/index.ts`). Cada tentativa de entrega loga `{ eventId, webhookId, customerId, attempt, statusCode, durationMs }`, no mesmo espírito estruturado do log `http_request` já emitido por `src/middlewares/request-logger.middleware.ts`.
- **Tracing/correlação:** `event_id` (gerado na inserção da outbox) funciona como identificador de correlação ponta a ponta — da criação do evento, pelas tentativas de retry, até o replay de DLQ —, no mesmo papel que `requestId`/`X-Request-Id` já exerce para requisições HTTP no projeto.

## 10. Dependências e Compatibilidade

- Mesmo banco MySQL e mesmo `DATABASE_URL` da API (ADR-002); novas tabelas via migration Prisma.
- Worker depende de uma instância própria de `PrismaClient` (processo separado).
- Nenhuma dependência de infraestrutura nova (sem Redis, sem broker de mensagens) — reaproveita integralmente a stack atual (Express, Prisma, Zod, Pino, JWT).
- Compatibilidade: módulos existentes (`auth`, `customers`, `products`) não são alterados; único ponto de modificação em código existente é `OrderService.changeStatus`.

## 11. Integração com o Sistema Existente

- **`src/modules/orders/order.service.ts`** — método `changeStatus`: após o `tx.order.update(...)` e `tx.orderStatusHistory.create(...)` já existentes (dentro do mesmo `prisma.$transaction`), passa a chamar `publishWebhookEvent(tx, order, from, to)`. Se a inserção na outbox falhar, a transação inteira (incluindo a própria mudança de status) sofre rollback, preservando a garantia transacional descrita no ADR-001.
- **`src/shared/errors/http-errors.ts`** — as novas classes de erro do módulo de webhooks reaproveitam `BadRequestError` e `ConflictError` (que já aceitam um parâmetro `code` customizável), seguindo exatamente o padrão usado por `InvalidStatusTransitionError`. Nenhuma alteração é necessária neste arquivo.
- **`src/middlewares/error.middleware.ts`** — nenhuma alteração necessária: como todas as novas classes de erro estendem `AppError`, o middleware já serializa `{ error: { code, message, details } }` corretamente para o módulo de webhooks.
- **`src/middlewares/auth.middleware.ts`** — `requireRole('ADMIN')` é reaproveitado sem modificação para proteger `POST /admin/webhooks/dead-letter/:id/replay`.
- **`src/shared/http/response.ts`** — `paginated()`/`buildPagination()` são reaproveitados sem modificação para `GET /customers/:customerId/webhooks` e `GET /webhooks/:webhookId/deliveries`.
- **`prisma/schema.prisma`** — adição dos modelos `WebhookEndpoint`, `WebhookOutboxEvent`, `WebhookDelivery` e `WebhookDeadLetter`, seguindo a convenção já usada por `Order`/`Customer` (`@id @default(uuid()) @db.Char(36)`, `@@map`, índices em colunas de filtro frequente).
- **`src/shared/logger/index.ts`** — o worker e o módulo de webhooks reaproveitam a instância Pino já configurada (incluindo os `redactPaths` existentes), sem novo mecanismo de logging.
- **`src/routes/index.ts`** — as rotas do módulo de webhooks (`buildWebhookRouter`) e a rota administrativa de DLQ são registradas em `buildApiRouter`, no mesmo padrão dos módulos existentes (`auth`, `customers`, `products`, `orders`).

## 12. Critérios de Aceite Técnicos

- Nenhuma mudança de status de pedido ocorre sem o evento correspondente ser registrado na outbox (quando há ao menos um endpoint interessado), e vice-versa (atomicidade verificada via teste de rollback forçado).
- O worker entrega um evento a um endpoint saudável em até 10 segundos desde a criação do evento, em condições normais (meta de negócio).
- Uma entrega que falha 5 vezes é movida para `webhook_dead_letter` e desaparece da fila de retry ativa do worker.
- O replay administrativo de um item de DLQ gera uma nova tentativa de entrega e é auditável (log com `userId`).
- Toda assinatura HMAC gerada pelo worker é verificável pelo cliente usando a secret vigente (atual ou, durante o grace period, a anterior).
- Todos os testes de `tests/orders.test.ts` continuam passando sem modificação (nenhuma mudança de contrato nos endpoints de pedidos existentes).

## 13. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Worker fica indisponível (crash, deploy sem supervisão) e a outbox acumula eventos pendentes sem ser notado | Média | Alto | Métrica `webhook_outbox_pending_count` e `webhook_worker_cycle_duration_ms` monitoradas (seção 9); processo separado do worker facilita restart automático via supervisor de processo |
| Volume de linhas na `webhook_outbox`/`webhook_deliveries` cresce sem controle (sem arquivamento nesta fase) | Média | Médio | Índices em `status`+`created_at` mantêm a consulta do worker eficiente mesmo com crescimento; arquivamento fica registrado como débito técnico fora de escopo (ver RFC) |
| Cliente implementa incorretamente a verificação HMAC ou a deduplicação por `X-Event-Id` | Média | Médio | Documentação destacada no portal do desenvolvedor (responsabilidade do PM, conforme reunião) |
