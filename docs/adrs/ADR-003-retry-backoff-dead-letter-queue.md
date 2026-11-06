# ADR-003: Retry com backoff exponencial e Dead Letter Queue em tabela separada

## Status
Aceito

## Contexto
Clientes podem estar temporariamente indisponíveis (o time já teve casos de clientes com indisponibilidade de até duas horas em manutenções planejadas). O sistema precisa de uma política de reentrega que tolere essas janelas sem manter eventos pendurados indefinidamente, e de um destino claro para eventos que esgotam as tentativas.

## Decisão
Aplicar **backoff exponencial com teto de 5 tentativas**, nos intervalos **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** (janela total de aproximadamente 15 horas entre a primeira falha e a última tentativa).

Após esgotar as 5 tentativas, o evento é movido para uma tabela dedicada, `webhook_dead_letter`, contendo payload, motivo da falha e timestamp — separada da `webhook_outbox` para manter a leitura da fila principal limpa e preservar evidência para debug.

O reprocessamento de itens na DLQ é **manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Esse endpoint exige role `ADMIN` (reaproveitando `requireRole` já existente) e deve registrar em log quem executou o replay, para auditoria.

## Alternativas Consideradas
- **3 tentativas em vez de 5.** Descartada: considerada agressiva demais — cobriria uma janela pequena (cerca de 30 minutos) e mataria o evento antes de uma indisponibilidade comum de cliente (já observada de até 2 horas).
- **Retry indefinido com backoff, sem teto.** Descartada: traz o risco de eventos ficarem pendurados para sempre caso o cliente desapareça definitivamente, sem um ponto claro de decisão/intervenção.
- **Marcar o evento como "failed" na própria tabela `webhook_outbox`** em vez de uma tabela separada. Descartada: poluiria a leitura da fila principal de pendentes e dificultaria o uso da falha como evidência de debug/reprocessamento.

## Consequências

**Positivas**
- Janela de ~15 horas cobre a grande maioria das indisponibilidades reais de clientes observadas até hoje, sem reter eventos indefinidamente.
- Separar a DLQ em tabela própria mantém a `webhook_outbox` enxuta para o worker e preserva o contexto completo da falha para investigação.

**Negativas**
- Falhas permanentes exigem intervenção manual via endpoint administrativo — não há notificação automática ao cliente final sobre a falha (ver fora de escopo: fallback por e-mail, adiado para fase futura).
- O teto de 5 tentativas significa que falhas acima de ~15 horas resultam em perda de entrega automática do evento, dependendo de ação humana para recuperação.

## Fonte
TRANSCRICAO.md — [09:14] a [09:19] e [09:35]–[09:36] (Diego, Bruno, Larissa, Sofia).
