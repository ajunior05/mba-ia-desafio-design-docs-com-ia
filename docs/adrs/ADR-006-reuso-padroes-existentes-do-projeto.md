# ADR-006: Reuso dos padrões arquiteturais existentes do projeto para o módulo de webhooks

## Status
Aceito

## Contexto
O projeto já tem convenções estabelecidas e consistentes entre os módulos existentes (`auth`, `customers`, `products`, `orders`): estrutura de módulo com `controller`/`service`/`repository`/`routes`/`schemas`, hierarquia de erros baseada em `AppError` (`src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`) com códigos de erro em `UPPER_SNAKE_CASE` (ex.: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`), logging centralizado com Pino (`src/shared/logger/index.ts`) e um middleware de erro único (`src/middlewares/error.middleware.ts`) que já sabe tratar `AppError`, `ZodError` e erros conhecidos do Prisma. Ao desenhar o módulo de webhooks, a equipe decidiu explicitamente não introduzir mecanismos paralelos para resolver os mesmos problemas.

## Decisão
O módulo de webhooks segue os padrões já estabelecidos no projeto:

- Nova pasta `src/modules/webhooks`, com a mesma estrutura de `controller`/`service`/`repository`/`routes`/`schemas` usada pelos demais módulos (ex.: `src/modules/orders`).
- Lógica de processamento do worker isolada em um arquivo próprio dentro do módulo (ex.: `webhook.worker.ts` ou `webhook.processor.ts`), com entry-point separado em `src/worker.ts`.
- Erros do módulo estendem `AppError` (`src/shared/errors/app-error.ts`), com códigos prefixados por `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo o padrão de `src/shared/errors/http-errors.ts`.
- Nenhuma mudança é necessária em `src/middlewares/error.middleware.ts`: por estender `AppError`, os novos erros já são tratados automaticamente pelo middleware existente.
- Logging via a instância Pino já compartilhada pelo projeto — nenhum novo mecanismo de log é introduzido.
- O worker usa uma instância própria de `PrismaClient` (por ser outro processo Node), mas aponta para o mesmo banco e a mesma `DATABASE_URL` da API.

## Alternativas Consideradas
- **Criar um mecanismo de erro/log específico para o módulo de webhooks**, dado que ele lida com integrações externas e pode ter necessidades diferentes dos demais módulos. Descartada implicitamente: a equipe foi explícita em não introduzir nada novo ("não vamos botar nada novo"), priorizando consistência sobre uma possível especialização futura.

## Consequências

**Positivas**
- Reduz a curva de entendimento do módulo para qualquer desenvolvedor já familiarizado com o restante do projeto.
- O middleware de erro centralizado funciona para o módulo de webhooks sem qualquer alteração, reduzindo risco de regressão nos demais módulos.

**Negativas**
- O módulo de webhooks herda qualquer limitação dos padrões atuais de erro/resposta HTTP do projeto (ex.: modelo de resposta de erro único para todos os módulos), mesmo que uma necessidade futura específica de integrações externas surja.

## Fonte
TRANSCRICAO.md — [09:27] a [09:30] (Bruno, Diego, Larissa). Código-fonte: `src/modules/orders/*`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`.
