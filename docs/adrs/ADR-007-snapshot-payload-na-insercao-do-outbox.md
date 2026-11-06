# ADR-007: Snapshot do payload do evento no momento da inserção no outbox

## Status
Aceito

## Contexto
Quando um evento é inserido na `webhook_outbox` (ADR-001), é preciso decidir se o conteúdo (payload) a ser enviado ao cliente é montado e armazenado no momento da inserção, ou se apenas uma referência (`order_id`) é guardada e o payload é montado só na hora do envio pelo worker — que pode ocorrer segundos ou minutos depois, inclusive após múltiplas tentativas de retry ao longo de horas (ADR-003).

## Decisão
O payload é **renderizado (snapshot) no momento da inserção** na outbox, refletindo o estado do pedido exatamente no momento em que a transição de status ocorreu. O worker envia esse snapshot já montado — não consulta o estado atual do pedido no momento do envio.

## Alternativas Consideradas
- **Armazenar apenas `order_id` e renderizar o payload no momento do envio.** Descartada: entre a inserção do evento e o efetivo envio (especialmente após retries, que podem se estender por até ~15 horas), o pedido pode ter sofrido novas mudanças de status. Renderizar tardiamente faria o evento refletir um estado diferente daquele que efetivamente motivou a notificação, criando inconsistência entre o `event_type`/`to_status` do evento e o estado real do pedido consultado depois.

## Consequências

**Positivas**
- O evento entregue ao cliente reflete fielmente o estado do pedido no momento exato da transição que o originou, mesmo que o pedido tenha mudado de status várias vezes depois (ex.: durante uma janela de retry longa).
- Elimina uma classe de condição de corrida entre o estado atual do pedido e o conteúdo do evento entregue.

**Negativas**
- Cada linha da `webhook_outbox` armazena o payload completo (JSON), aumentando o espaço ocupado por linha em comparação a guardar apenas uma referência.
- A montagem do payload precisa ocorrer dentro da própria transação de `changeStatus`, antes do commit — adicionando responsabilidade à função de publicação do evento (`publishWebhookEvent`) no caminho crítico da transação.

## Fonte
TRANSCRICAO.md — [09:51] a [09:53] (Diego, Larissa, Bruno).
