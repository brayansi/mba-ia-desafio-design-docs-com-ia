---
description: Gera os ADRs (Architecture Decision Records) para o Sistema de Webhooks a partir da TRANSCRICAO.md e do código existente
---

Você vai gerar entre 5 e 8 ADRs para o Sistema de Webhooks de Notificação de Pedidos.

## Antes de gerar

Leia obrigatoriamente:
- `TRANSCRICAO.md` — transcrição completa da reunião técnica
- `prisma/schema.prisma` — modelos de dados existentes
- `src/modules/orders/order.service.ts` — método `changeStatus` que será integrado
- `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — padrão de erros
- `src/shared/logger/index.ts` — logger Pino existente

## Decisões que DEVEM virar ADRs (cobertura mínima de 5 das 6)

1. **Padrão Outbox no MySQL** — decisão de usar outbox na mesma transação SQL ao invés de Redis Streams ou chamada HTTP síncrona dentro do `changeStatus`
2. **Política de retry com backoff exponencial e DLQ** — 5 tentativas, progressão 1m/5m/30m/2h/12h, tabela `webhook_dead_letter` separada
3. **Autenticação HMAC-SHA256 com secret por endpoint** — secret única por endpoint, rotação com grace period de 24h
4. **Garantia at-least-once com X-Event-Id** — UUID por evento na inserção da outbox, deduplicação pelo cliente
5. **Worker em processo separado com polling de 2 segundos** — entry-point `src/worker.ts`, não dentro da API
6. **Reuso dos padrões existentes do projeto** — `AppError`, Pino, error middleware, estrutura de módulo em `src/modules/webhooks/`, prefixo `WEBHOOK_` nos códigos de erro

Decisões secundárias que podem virar ADRs adicionais (até chegar em 8):
- UUID como identificador da outbox (seguir padrão do projeto)
- Payload renderizado (snapshot) na inserção, não lazy no envio
- Filtro de eventos na inserção da outbox, não no envio
- Timeout de 10 segundos no HTTP call do worker

## O que NÃO deve aparecer como decisão (foi descartado ou adiado)

- Redis Streams / filas externas — descartado em [09:07] por ser overengineering para time pequeno
- Webhook síncrono dentro do service de orders — descartado em [09:04]/[09:06]
- Trigger de banco para notificar worker — descartado em [09:09] (MySQL não tem NOTIFY/LISTEN)
- Email de alerta para cliente — explicitamente adiado para próxima fase em [09:37]
- Rate limiting de saída — adiado como "observar e decidir depois" em [09:39]
- Dashboard visual — fora de escopo em [09:40]
- Garantia exactly-once — descartada em [09:25], at-least-once é suficiente

## Formato obrigatório de cada ADR

Use o formato MADR. Nome do arquivo: `ADR-NNN-titulo-em-kebab-case.md`. Salve em `docs/adrs/`.

```markdown
# ADR-NNN: Título da Decisão

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

[Qual era o problema ou situação que motivou esta decisão? Referencie o timestamp da transcrição quando aplicável.]

## Decisão

[O que foi decidido, de forma clara e afirmativa.]

## Alternativas Consideradas

### Alternativa 1: [Nome]
- **Descrição:** ...
- **Por que foi descartada:** ...

### Alternativa 2: [Nome] (se houver)
- **Descrição:** ...
- **Por que foi descartada:** ...

## Consequências

**Positivas:**
- ...

**Negativas / Trade-offs:**
- ...

## Referências

- TRANSCRICAO.md [timestamp] [falante]
- [caminho/arquivo.ts] (quando a decisão referencia código existente)
```

## Regras de qualidade

- Pelo menos 1 ADR deve referenciar explicitamente um arquivo ou classe do código existente (ex: `src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`)
- Cada alternativa considerada deve citar o trade-off real que levou ao descarte — sem alternativas genéricas inventadas
- Toda informação deve ter origem rastreável na transcrição (timestamp + nome) ou no código (caminho do arquivo)
- Não invente restrições ou requisitos ausentes na transcrição

## Saída esperada

Crie os arquivos diretamente em `docs/adrs/`. Após criar todos, liste os arquivos gerados e confirme que cobrem pelo menos 5 das 6 decisões principais.
