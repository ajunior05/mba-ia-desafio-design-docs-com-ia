# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status
Aceito

## Contexto
Dado o modelo de outbox + retry (ADR-001, ADR-003), é possível que um mesmo evento seja entregue mais de uma vez ao cliente — por exemplo, se a chamada HTTP for bem-sucedida do lado do servidor de destino, mas a confirmação se perder antes de chegar ao worker, que então tentará novamente. Era preciso decidir o nível de garantia de entrega e como o cliente consegue lidar com entregas duplicadas.

## Decisão
Garantir **at-least-once delivery**: o cliente pode, eventualmente, receber o mesmo evento mais de uma vez, e deve estar preparado para isso.

Cada evento recebe um **`event_id`** (UUID) gerado no momento em que entra na `webhook_outbox`, único por evento, enviado no header **`X-Event-Id`**. O cliente é responsável por deduplicar do lado dele usando esse identificador. Essa responsabilidade deve ser documentada de forma destacada no portal de desenvolvedor.

## Alternativas Consideradas
- **Garantia exactly-once.** Descartada: exigiria coordenação transacional entre os dois lados (plataforma e cliente), o que aumenta substancialmente a complexidade do sistema para resolver um problema que o padrão at-least-once com deduplicação por identificador já resolve para a grande maioria dos casos — abordagem adotada por players de mercado como Stripe e GitHub.

## Consequências

**Positivas**
- Mantém o worker e o protocolo de entrega simples, sem necessidade de coordenação distribuída entre plataforma e cliente.
- Segue um padrão já validado e documentado por players de mercado, facilitando a implementação do lado do cliente.

**Negativas**
- Transfere para o cliente a responsabilidade de implementar deduplicação — uma integração que não trate `X-Event-Id` corretamente pode processar o mesmo evento de negócio mais de uma vez.
- Requer documentação explícita e destacada no portal do desenvolvedor para que os clientes não sejam surpreendidos por duplicatas.

## Fonte
TRANSCRICAO.md — [09:24] a [09:26] (Diego, Bruno, Sofia, Marcos, Larissa).
