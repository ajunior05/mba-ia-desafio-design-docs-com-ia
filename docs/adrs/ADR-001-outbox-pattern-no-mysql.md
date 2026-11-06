# ADR-001: Outbox Pattern no MySQL para publicação de eventos de pedido

## Status
Aceito

## Contexto
A feature de Webhooks precisa notificar sistemas externos quando o status de um pedido muda, sem comprometer a confiabilidade da transação que já existe em `OrderService.changeStatus` (atualização de `orders`, inserção em `order_status_history` e ajuste de `stock_quantity`).

Disparar a chamada HTTP de forma síncrona dentro dessa transação foi descartado: qualquer cliente lento travaria a mudança de status de outros pedidos, e não há como fazer rollback de uma notificação já enviada se o cliente estiver fora do ar.

O time é pequeno e já opera MySQL como banco principal; subir uma nova peça de infraestrutura (ex.: fila dedicada) para resolver o desacoplamento foi considerado desproporcional ao problema.

## Decisão
Adotar o padrão **Outbox**: ao mudar o status de um pedido, dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`, inserir também uma linha em uma nova tabela `webhook_outbox` com o evento a ser entregue. Um worker separado lê essa tabela de forma assíncrona e realiza as chamadas HTTP.

Se a transação principal der commit, o evento foi registrado; se der rollback, o evento desaparece junto. Não há inconsistência possível entre o estado do pedido e a existência do evento.

## Alternativas Consideradas
- **Fila externa (ex.: Redis Streams).** Resolveria o desacoplamento, mas exigiria subir e operar uma nova peça de infraestrutura. Descartada por overengineering para o tamanho do time e da necessidade atual.

## Consequências

**Positivas**
- Garantia transacional nativa: evento e mudança de status são atômicos, sem necessidade de two-phase commit ou infraestrutura adicional.
- Reutiliza o MySQL já operado pelo time, sem novo componente de infra para manter.

**Negativas**
- Introduz a necessidade de um worker assíncrono dedicado para drenar a tabela (ver ADR-002).
- A tabela `webhook_outbox` cresce continuamente; arquivamento de linhas entregues após um período fica fora do escopo desta feature.

## Fonte
TRANSCRICAO.md — [09:03] a [09:08] (Larissa, Bruno, Diego).
