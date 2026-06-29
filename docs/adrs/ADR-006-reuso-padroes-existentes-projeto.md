# ADR-006: Reuso dos Padrões Existentes do Projeto no Módulo de Webhooks

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Bruno (Eng. Pleno, Pedidos), Larissa (Tech Lead)
**Revisores:** Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

O projeto já possui convenções consolidadas: módulos em `src/modules/<domínio>/` com controller, service, repository, routes e schemas; classe `AppError` e subclasses tipadas com códigos de erro em SCREAMING_SNAKE_CASE; logger Pino centralizado em `src/shared/logger/index.ts`; middleware de erro que trata `AppError`, erros Zod e erros Prisma automaticamente. Era necessário decidir se o módulo de webhooks seguiria essas convenções ou introduziria abordagens diferentes [09:27–09:30, Bruno, Diego, Larissa].

## Decisão

O módulo de webhooks seguirá integralmente os padrões existentes:
- Estrutura de diretório em `src/modules/webhooks/` com os mesmos sub-arquivos dos outros módulos.
- Erros do domínio como subclasses de `AppError` (ou das classes de `src/shared/errors/http-errors.ts`) com prefixo `WEBHOOK_` nos códigos (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
- Logger Pino via `import { logger } from '../../shared/logger/index.js'` — sem nova biblioteca de logging.
- Middleware de erro centralizado já captura `AppError` sem modificação.
- Schemas de validação com Zod, seguindo o padrão dos outros módulos.

## Alternativas Consideradas

### Alternativa 1: Novo padrão de erros específico para webhooks
- **Descrição:** Criar uma hierarquia de erros própria para o módulo de webhooks com estrutura diferente.
- **Por que foi descartada:** O middleware de erro centralizado já trata `AppError` e suas subclasses. Criar uma hierarquia paralela quebraria esse tratamento e exigiria modificar o middleware, aumentando superfície de mudança sem benefício [09:28–09:29, Bruno].

### Alternativa 2: Biblioteca de logging diferente para o worker
- **Descrição:** Usar `console.log` ou outra biblioteca no `src/worker.ts` por ser um processo separado.
- **Por que foi descartada:** O `createLogger()` de `src/shared/logger/index.ts` é uma função pura que não depende do contexto HTTP — pode ser importada pelo worker normalmente. Manter um único logger garante consistência de formato e redação de campos sensíveis (ex.: `*.token`) [09:29, Bruno].

## Consequências

**Positivas:**
- Zero custo de aprendizado para novos desenvolvedores: o módulo de webhooks funciona exatamente como `src/modules/orders/` ou qualquer outro módulo.
- O middleware de erro centralizado captura `WEBHOOK_*` errors sem nenhuma alteração.
- Redação de campos sensíveis (`*.token`) no Pino já cobre o campo de secret do webhook [ver `src/shared/logger/index.ts`, linha 9].
- Tempo de desenvolvimento reduzido por não precisar criar scaffolding novo.

**Negativas / Trade-offs:**
- O worker precisa instanciar um `PrismaClient` próprio (não pode reutilizar a instância da API por ser outro processo), mesmo que use o mesmo `DATABASE_URL` [09:29–09:30, Diego, Bruno].

## Referências

- TRANSCRICAO.md [09:27–09:30] Bruno, Diego, Larissa
- `src/shared/errors/app-error.ts` — classe base `AppError` com `statusCode`, `errorCode` e `details`
- `src/shared/errors/http-errors.ts` — subclasses `NotFoundError`, `ConflictError`, `InvalidStatusTransitionError` como referência de padrão a ser seguido
- `src/shared/logger/index.ts` — `createLogger()` importável tanto pela API quanto pelo worker
- `prisma/schema.prisma` — padrão `@id @default(uuid()) @db.Char(36)` a ser seguido nos novos modelos da outbox
