# ADR-002: Worker dedicado em processo separado, com polling de 2 segundos

## Status
Aceito

## Contexto
A tabela `webhook_outbox` (ADR-001) precisa ser drenada por algum processo que efetivamente dispare as chamadas HTTP para os clientes. MySQL não possui um mecanismo nativo de notificação para processos externos equivalente ao `LISTEN`/`NOTIFY` do PostgreSQL — triggers no banco só executam SQL, não conseguem avisar um processo fora dele.

Os clientes B2B definiram "tempo real" como qualquer latência abaixo de 10 segundos entre a mudança de status e o recebimento do webhook.

Também foi levantado o risco de rodar esse processamento dentro da própria instância da API: se a API reiniciar (deploy, crash, scale down), o worker embutido reinicia ou desaparece junto, criando um ponto de falha compartilhado sem necessidade.

## Decisão
Implementar um **worker em processo separado** (novo entry-point `src/worker.ts`, executado via script `npm run worker`), que roda em **loop de polling a cada 2 segundos**: busca os eventos pendentes mais antigos na `webhook_outbox` em lotes pequenos, processa e marca como entregue. O worker conecta no mesmo banco (mesma `DATABASE_URL`) e usa o mesmo Prisma Client como dependência, mas com uma instância própria de `PrismaClient` — por ser outro processo Node, não pode compartilhar a instância da API.

Com um único worker ativo, o processamento segue a ordem de `created_at` da outbox, o que garante ordenação implícita por `order_id` (mas não ordenação global) enquanto o modelo permanecer single-worker.

## Alternativas Consideradas
- **Rodar o processamento embutido na própria instância da API** (sem processo separado). Descartada explicitamente: um restart da API eliminaria o worker junto, criando janelas sem processamento de eventos.
- **Reagir via trigger de banco** que avisasse o worker diretamente. Descartada: MySQL não tem notificação nativa para processos externos; a alternativa (trigger escrevendo em arquivo ou chamando um endpoint) foi considerada artificial demais frente ao ganho.

## Consequências

**Positivas**
- Processo do worker é independente do ciclo de vida/deploy da API — reinícios da API não interrompem o processamento de eventos pendentes.
- Polling de 2s atende com folga confortável o requisito de latência (< 10s definido pelos clientes).

**Negativas**
- Latência mínima de até 2 segundos no melhor caso, inerente ao modelo de polling.
- Ordenação de entrega só é garantida por `order_id` e apenas enquanto houver um único worker ativo; escalar para múltiplos workers em paralelo (partição por `order_id` ou lock pessimista) é um problema declarado como futuro, fora do escopo atual.

## Fonte
TRANSCRICAO.md — [09:09] a [09:13] (Diego, Bruno, Larissa, Marcos).
