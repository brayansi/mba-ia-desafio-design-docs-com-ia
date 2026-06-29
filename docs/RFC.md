# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo         | Valor                                                                             |
|---------------|-----------------------------------------------------------------------------------|
| **Autor**     | Larissa (Tech Lead)                                                               |
| **Status**    | Em revisão                                                                        |
| **Data**      | 2026-06-29                                                                        |
| **Revisores** | Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM) |

---

## TL;DR

Propomos um sistema de webhooks outbound que notifica clientes B2B sobre mudanças de status de pedidos usando o padrão Transactional Outbox sobre o MySQL existente. Um worker em processo separado faz polling a cada 2 segundos, entrega os eventos com assinatura HMAC-SHA256 e aplica retry com backoff exponencial (5 tentativas, até ~15 horas). A entrega é at-least-once com `X-Event-Id` para deduplicação no lado do cliente.

---

## Contexto e Problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram notificações em tempo real sobre mudanças de status dos seus pedidos. Atualmente, eles fazem polling contínuo no `GET /orders`, o que torna a integração lenta e cara para eles. A Atlas sinalizou risco de churn caso a solução não seja entregue até fim do trimestre [09:00, Marcos].

O requisito de latência é abaixo de 10 segundos, que os clientes consideram "tempo real" [09:02, Marcos]. Não fazer nada implica perda de ao menos um cliente estratégico e continuidade do problema de polling excessivo sobre nossa API.

O ponto de integração natural é o método `changeStatus` em `src/modules/orders/order.service.ts`, que já executa dentro de uma transação SQL que atualiza `orders`, insere em `order_status_history` e ajusta estoque — tornando qualquer abordagem síncrona de notificação inviável.

---

## Proposta Técnica

### Visão geral

A solução é composta por três partes:

**1. Publicação atômica via Outbox**
Dentro da mesma transação do `changeStatus`, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` insere o evento serializado (snapshot do estado do pedido naquele instante) na tabela `webhook_outbox`. Se a transação principal fizer rollback, o evento some junto — sem risco de inconsistência entre estado do pedido e eventos publicados.

**2. Worker em processo separado**
Um processo Node.js independente (`src/worker.ts`, `npm run worker`) faz polling na `webhook_outbox` a cada 2 segundos, buscando eventos pendentes. Para cada evento, consulta os endpoints cadastrados do cliente que assinaram aquele tipo de transição e realiza o HTTP POST com timeout de 10 segundos. O worker usa o mesmo Prisma e o mesmo `DATABASE_URL`, mas com instância própria de `PrismaClient`.

**3. Entrega confiável e autenticada**
Cada requisição carrega `X-Event-Id` (UUID gerado na inserção da outbox, estável entre retentativas), `X-Signature` (HMAC-SHA256 do corpo com a secret do endpoint) e `X-Timestamp`. A secret é única por endpoint, gerada pela plataforma no cadastro, com suporte a rotação com grace period de 24 horas. A garantia é at-least-once: o cliente deduplica usando `X-Event-Id`.

Em caso de falha, o worker aplica backoff exponencial: 1m → 5m → 30m → 2h → 12h (5 tentativas). Eventos que esgotam as tentativas são movidos para `webhook_dead_letter` e podem ser reprocessados manualmente via `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN` obrigatória).

O filtro de quais status cada endpoint quer receber é aplicado na inserção da outbox: se nenhum endpoint do cliente assinou aquela transição, nenhuma linha é inserida.

### Estrutura de módulos

O módulo segue o padrão existente do projeto: `src/modules/webhooks/` com controller, service, repository, routes e schemas. Erros usam `AppError` com prefixo `WEBHOOK_`. Logger Pino e middleware de erro centralizados são reaproveitados sem modificação.

---

## Alternativas Consideradas

### Alternativa 1: Webhook síncrono dentro do `OrderService`
- **Descrição:** Executar o POST para o endpoint do cliente diretamente dentro da transação de `changeStatus`, antes do commit.
- **Trade-off que levou ao descarte:** A transação já é pesada (atualiza `orders`, `order_status_history`, `stock_quantity`). Um cliente HTTP lento travaria mudanças de status de outros pedidos. Se o endpoint do cliente estivesse offline, seria necessário dar rollback na mudança de status — comportamento inaceitável [09:04, Bruno; 09:06, Diego].

### Alternativa 2: Redis Streams como fila de eventos
- **Descrição:** Publicar o evento em um Redis Stream após o commit da transação principal; um worker consumiria esse stream de forma reativa.
- **Trade-off que levou ao descarte:** Exigiria subir e operar um Redis Cluster como nova infraestrutura. Para um time pequeno, isso é overengineering: o MySQL existente resolve o problema do outbox sem adicionar dependência operacional [09:07, Diego].

---

## Questões em Aberto

| # | Questão | Decisão tomada | Observação |
|---|---------|----------------|------------|
| 1 | **Rate limiting de envio por cliente** — se um cliente tem 50 pedidos mudando de status em um minuto, a plataforma envia 50 chamadas HTTP simultâneas para ele? | Adiado — observar e decidir depois | Levantado por Diego [09:38]; sem dados de volume reais, a decisão foi não implementar agora e monitorar após o lançamento [09:39, Larissa] |
| 2 | **Notificação por e-mail em caso de falha de webhook** — alertar o cliente quando o endpoint dele acumular falhas consecutivas | Adiado para próxima fase | Levantado por Marcos [09:37]; explicitamente fora de escopo desta fase; pode ser retomado após medir impacto [09:37, Larissa] |
| 3 | **Garantia de ordering com múltiplos workers** — com único worker, a ordem por `order_id` é implícita; escalar para múltiplos workers quebra essa garantia | Documentado como limitação conhecida | Diego propôs particionamento por `order_id` ou lock pessimista como solução futura [09:13]; clientes não exigem ordering global [09:14, Marcos] |

---

## Impacto e Riscos

**Integração com `changeStatus`**
A inserção na outbox ocorre dentro da transação existente. Qualquer falha na inserção (ex.: constraint violation, timeout de banco) causará rollback da mudança de status inteira. Esse comportamento é intencional, mas adiciona um novo ponto de falha na operação mais crítica do sistema de pedidos. O teste de integração deve cobrir esse caminho.

**Processo separado requer orquestração**
O worker é um segundo processo Node.js que precisa ser iniciado, monitorado e reiniciado em caso de crash. Em produção, isso exige configuração explícita (PM2, systemd, container dedicado) — ausente hoje no projeto. O risco de o worker parar silenciosamente sem alertas é real.

**Revisão de segurança obrigatória antes do deploy**
Sofia reservou ao menos dois dias úteis para revisar o código de geração de secret e a implementação de HMAC antes de qualquer deploy em produção [09:46, Sofia]. O deploy não pode acontecer antes dessa revisão.

**DLQ sem reprocessamento automático**
Eventos na `webhook_dead_letter` requerem intervenção manual via endpoint admin. Se o volume de falhas for alto (ex.: cliente offline por período longo), a fila pode crescer sem visibilidade — monitoramento do tamanho da DLQ deve ser configurado junto com a feature.

---

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-padrao-outbox-mysql.md)
- [ADR-002: Retry com Backoff Exponencial e DLQ](adrs/ADR-002-retry-backoff-exponencial-dlq.md)
- [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](adrs/ADR-003-autenticacao-hmac-sha256.md)
- [ADR-004: Garantia At-Least-Once com X-Event-Id](adrs/ADR-004-garantia-at-least-once-x-event-id.md)
- [ADR-005: Worker em Processo Separado com Polling de 2s](adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006: Reuso dos Padrões Existentes do Projeto](adrs/ADR-006-reuso-padroes-existentes-projeto.md)
- [ADR-007: Payload Renderizado como Snapshot na Inserção](adrs/ADR-007-payload-snapshot-na-insercao.md)
