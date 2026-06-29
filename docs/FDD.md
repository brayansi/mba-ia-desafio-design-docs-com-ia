# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo         | Valor                                                                             |
|---------------|-----------------------------------------------------------------------------------|
| **Autor**     | Larissa (Tech Lead)                                                               |
| **Status**    | Em elaboração                                                                     |
| **Data**      | 2026-06-29                                                                        |
| **Revisores** | Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança) |

---

## 1. Contexto e Motivação Técnica

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fazem polling contínuo em `GET /orders` para detectar mudanças de status, gerando carga desnecessária na API e latência inaceitável para eles [09:00, Marcos]. A solução é um sistema de webhooks outbound que notifica proativamente os endpoints desses clientes quando o status de um pedido muda.

O ponto de integração é o método `changeStatus` em `src/modules/orders/order.service.ts` (linhas 126–178), que já executa dentro de uma `prisma.$transaction`. A arquitetura escolhida — padrão Transactional Outbox — insere o evento de notificação dentro dessa mesma transação, garantindo atomicidade completa: se a mudança de status fizer rollback, o evento de webhook não é criado.

---

## 2. Objetivos Técnicos

- Latência máxima de 10 segundos entre a mudança de status e a entrega do webhook ao cliente [09:02, Marcos; 09:09, Diego]
- Garantia de que eventos publicados são entregues ao menos uma vez (at-least-once) mesmo em caso de falha temporária do endpoint do cliente
- Autenticação de cada entrega via HMAC-SHA256, com secret única por endpoint e suporte a rotação sem downtime
- Zero nova infraestrutura além do MySQL já existente
- Módulo de webhooks consistente com os demais módulos do projeto em estrutura, erros e logging

---

## 3. Escopo e Exclusões

**Dentro do escopo:**
- Cadastro, edição, listagem e remoção de endpoints de webhook por customer
- Rotação de secret com grace period de 24 horas
- Publicação atômica de eventos na outbox dentro da transação de `changeStatus`
- Worker em processo separado com polling de 2 segundos
- Retry com backoff exponencial: 5 tentativas, 1m/5m/30m/2h/12h
- Dead Letter Queue (`webhook_dead_letter`) com replay manual via endpoint admin
- Histórico de entregas por endpoint (`GET /webhooks/:id/deliveries`)
- Filtro de eventos por status na inserção da outbox

**Fora do escopo (explicitamente excluídos):**
- Webhook inbound (cliente enviando eventos para a plataforma) — descartado em [09:02, Marcos]
- Notificação por e-mail em caso de falhas consecutivas — adiado para próxima fase [09:37, Larissa]
- Rate limiting de envio por cliente — adiado como "observar e decidir depois" [09:39, Larissa]
- Dashboard visual para o cliente — projeto separado do time de frontend [09:40, Larissa]
- Garantia exactly-once — descartada em favor de at-least-once com `X-Event-Id` [09:25, Diego]
- Escalonamento com múltiplos workers e garantia de ordering global — adiado [09:13, Diego]
- Arquivamento de eventos entregues — fora do escopo desta fase [09:08, Diego]

---

## 4. Fluxos Detalhados

### Fluxo 1: Inserção atômica na outbox

```
OrderController.changeStatus
        │
        ▼
OrderService.changeStatus(id, input, userId)
        │
        └─► prisma.$transaction(async (tx) => {
                │
                ├─ tx.order.findUnique()          — busca pedido atual
                ├─ validação de transição de status
                ├─ debitStock / replenishStock     — ajuste de estoque
                ├─ tx.order.update()              — atualiza status
                ├─ tx.orderStatusHistory.create() — registra histórico
                │
                └─ publishWebhookEvent(tx, order, fromStatus, toStatus)
                        │
                        ├─ consulta webhooks ativos do customer
                        │   que filtram o toStatus
                        │
                        ├─ [nenhum webhook] → retorna sem inserir
                        │
                        └─ [há webhooks] → para cada webhook:
                              tx.webhookOutbox.create({
                                eventId: uuid(),
                                webhookEndpointId: webhook.id,
                                payload: JSON.stringify(snapshot),
                                status: 'PENDING'
                              })
            })
            │
            ├─ COMMIT → mudança de status + eventos persistidos atomicamente
            └─ ROLLBACK → nada é persistido (nem status, nem evento)
```

### Fluxo 2: Processamento pelo worker

```
src/worker.ts (processo separado, npm run worker)
        │
        └─► loop infinito com intervalo de 2 segundos
                │
                ▼
        busca batch de eventos PENDING
        ORDER BY created_at ASC
        LIMIT 50 (batch pequeno)
                │
                ├─ [sem eventos] → aguarda 2s, repete
                │
                └─ [com eventos] → para cada evento:
                        │
                        ├─ carrega configuração do WebhookEndpoint
                        ├─ calcula HMAC-SHA256(payload, secret)
                        ├─ HTTP POST para endpoint.url
                        │   Headers: X-Event-Id, X-Webhook-Id,
                        │            X-Signature, X-Timestamp,
                        │            Content-Type: application/json
                        │   Timeout: 10 segundos
                        │
                        ├─ resposta 2xx:
                        │       marca outbox status = 'DELIVERED'
                        │       insere em webhook_delivery (sucesso)
                        │
                        └─ falha (timeout, non-2xx, erro de rede):
                                incrementa attempt_count
                                calcula next_retry_at (backoff)
                                atualiza status = 'PENDING' com next_retry_at
                                insere em webhook_delivery (falha)
                                se attempt_count >= 5 → Fluxo 3 (DLQ)
```

### Fluxo 3: Retry com backoff exponencial e DLQ

```
attempt_count │ status após falha │ next_retry_at
──────────────┼───────────────────┼──────────────────────────
      1        │ PENDING           │ now + 1 minuto
      2        │ PENDING           │ now + 5 minutos
      3        │ PENDING           │ now + 30 minutos
      4        │ PENDING           │ now + 2 horas
      5        │ FAILED → DLQ      │ —

Após 5ª falha:
  webhook_outbox.status = 'FAILED'
  INSERT INTO webhook_dead_letter {
    webhookOutboxId,
    webhookEndpointId,
    payload,
    lastErrorMessage,
    failedAt: now()
  }
```

### Fluxo 4: Replay de DLQ

```
POST /admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <jwt com role=ADMIN>
        │
        ├─ authenticate (verifica JWT)
        ├─ requireRole('ADMIN') (verifica papel)
        │
        ▼
AdminWebhookController.replayDeadLetter(id)
        │
        ├─ busca registro em webhook_dead_letter → 404 se não encontrar
        │
        └─ prisma.$transaction:
                INSERT INTO webhook_outbox {
                  eventId: uuid(),          ← novo UUID para nova tentativa
                  webhookEndpointId: dlq.webhookEndpointId,
                  payload: dlq.payload,
                  status: 'PENDING',
                  attemptCount: 0,
                  nextRetryAt: null
                }
                log de auditoria: { adminUserId, dlqId, replayedAt }

Response 200: { id: novoOutboxId, replayedAt }
```

---

## 5. Contratos Públicos (Endpoints HTTP)

### POST /webhooks — Cadastrar endpoint de webhook

```
POST /webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

Request body:
{
  "customerId": "3f2a1b4c-...",
  "url": "https://cliente.com/webhook/pedidos",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "description": "Webhook principal Atlas Comercial"
}

Response 201:
{
  "id": "9e1d7f3a-...",
  "customerId": "3f2a1b4c-...",
  "url": "https://cliente.com/webhook/pedidos",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "description": "Webhook principal Atlas Comercial",
  "secret": "whsec_a3f9...",
  "active": true,
  "createdAt": "2026-06-29T10:00:00.000Z"
}

Observação: o campo "secret" é retornado APENAS na criação. Endpoints de leitura não retornam a secret.

Erros possíveis:
  422 WEBHOOK_INVALID_URL    — URL não inicia com https://
  422 WEBHOOK_INVALID_EVENT  — algum valor em "events" não existe no enum OrderStatus
  404 NOT_FOUND              — customerId não encontrado
```

### GET /webhooks?customerId=:id — Listar webhooks de um customer

```
GET /webhooks?customerId=3f2a1b4c-...
Authorization: Bearer <jwt>

Response 200:
{
  "items": [
    {
      "id": "9e1d7f3a-...",
      "customerId": "3f2a1b4c-...",
      "url": "https://cliente.com/webhook/pedidos",
      "events": ["PAID", "SHIPPED", "DELIVERED"],
      "description": "Webhook principal Atlas Comercial",
      "active": true,
      "createdAt": "2026-06-29T10:00:00.000Z",
      "updatedAt": "2026-06-29T10:00:00.000Z"
    }
  ],
  "total": 1
}

Erros possíveis:
  400 VALIDATION_ERROR — customerId ausente ou inválido
```

### PATCH /webhooks/:id — Atualizar configuração

```
PATCH /webhooks/9e1d7f3a-...
Authorization: Bearer <jwt>
Content-Type: application/json

Request body (todos os campos opcionais):
{
  "url": "https://cliente.com/webhook/v2",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "description": "Webhook atualizado",
  "active": false
}

Response 200:
{
  "id": "9e1d7f3a-...",
  "customerId": "3f2a1b4c-...",
  "url": "https://cliente.com/webhook/v2",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "description": "Webhook atualizado",
  "active": false,
  "createdAt": "2026-06-29T10:00:00.000Z",
  "updatedAt": "2026-06-29T11:30:00.000Z"
}

Erros possíveis:
  404 WEBHOOK_NOT_FOUND       — ID não encontrado
  422 WEBHOOK_INVALID_URL     — URL sem https://
  422 WEBHOOK_INVALID_EVENT   — status inválido em "events"
  409 WEBHOOK_ALREADY_INACTIVE — tentativa de desativar já inativo
```

### DELETE /webhooks/:id — Remover endpoint

```
DELETE /webhooks/9e1d7f3a-...
Authorization: Bearer <jwt>

Response 204: (sem corpo)

Erros possíveis:
  404 WEBHOOK_NOT_FOUND — ID não encontrado
```

### POST /webhooks/:id/secret/rotate — Rotacionar secret

```
POST /webhooks/9e1d7f3a-.../secret/rotate
Authorization: Bearer <jwt>

Request body: (sem corpo)

Response 200:
{
  "id": "9e1d7f3a-...",
  "newSecret": "whsec_b7e2...",
  "previousSecretExpiresAt": "2026-06-30T11:30:00.000Z"
}

Comportamento: a secret anterior permanece válida por 24 horas (grace period)
para o cliente migrar seus sistemas sem interrupção [09:21, Sofia].
Após o grace period, apenas a nova secret é aceita.

Erros possíveis:
  404 WEBHOOK_NOT_FOUND — ID não encontrado
```

### GET /webhooks/:id/deliveries — Histórico de entregas

```
GET /webhooks/9e1d7f3a-.../deliveries?limit=100
Authorization: Bearer <jwt>

Response 200:
{
  "items": [
    {
      "id": "c4a8e1f2-...",
      "eventId": "d9b3a7e5-...",
      "status": "DELIVERED",
      "httpStatusCode": 200,
      "attemptNumber": 1,
      "durationMs": 234,
      "deliveredAt": "2026-06-29T10:00:02.000Z",
      "errorMessage": null
    },
    {
      "id": "f1c2d3e4-...",
      "eventId": "a8b9c0d1-...",
      "status": "FAILED",
      "httpStatusCode": 503,
      "attemptNumber": 3,
      "durationMs": 10001,
      "deliveredAt": null,
      "errorMessage": "Timeout after 10000ms"
    }
  ],
  "total": 2
}

Erros possíveis:
  404 WEBHOOK_NOT_FOUND — ID do webhook não encontrado
```

### POST /admin/webhooks/dead-letter/:id/replay — Reprocessar item da DLQ

```
POST /admin/webhooks/dead-letter/e5f6a7b8-.../replay
Authorization: Bearer <jwt com role=ADMIN>

Request body: (sem corpo)

Response 200:
{
  "replayedOutboxId": "12ab34cd-...",
  "replayedAt": "2026-06-29T12:00:00.000Z"
}

Requer role ADMIN. Registra auditoria com o ID do usuário que executou o replay [09:36, Sofia].

Erros possíveis:
  404 WEBHOOK_DLQ_NOT_FOUND  — ID de DLQ não encontrado
  403 FORBIDDEN              — usuário sem role ADMIN
  401 UNAUTHORIZED           — token ausente ou inválido
```

---

## 6. Payload do Webhook (enviado ao cliente)

**Body JSON:**
```json
{
  "event_id": "d9b3a7e5-1234-4abc-8def-000000000001",
  "event_type": "order.status_changed",
  "timestamp": "2026-06-29T10:00:00.000Z",
  "order_id": "7c3f1a2b-...",
  "order_number": "ORD-000042",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "3f2a1b4c-...",
  "total_cents": 15000
}
```

O payload não inclui os itens do pedido para manter o tamanho reduzido. Se o cliente precisar de detalhes completos, deve consultar `GET /orders/:id` [09:43, Diego].

**Headers enviados em cada entrega:**
```
X-Event-Id: d9b3a7e5-1234-4abc-8def-000000000001
X-Webhook-Id: 9e1d7f3a-...
X-Signature: sha256=<hmac-sha256-hex-do-body>
X-Timestamp: 2026-06-29T10:00:00.000Z
Content-Type: application/json
```

O `X-Event-Id` é o mesmo UUID em todas as retentativas do mesmo evento — o cliente usa esse valor para deduplicar entregas duplicadas (at-least-once) [09:25, Diego]. O `X-Webhook-Id` permite ao cliente com múltiplos endpoints cadastrados identificar qual configuração gerou aquele envio [09:44, Sofia].

**Cálculo da assinatura:**
```
signature = HMAC-SHA256(body_string_utf8, webhook_secret)
X-Signature: sha256=<hex(signature)>
```

Usando a biblioteca `crypto` nativa do Node.js — sem dependência externa.

---

## 7. Matriz de Erros

Todos os erros do módulo de webhooks estendem `AppError` de `src/shared/errors/app-error.ts` e são capturados automaticamente pelo `src/middlewares/error.middleware.ts`.

| Código | HTTP | Mensagem | Quando ocorre |
|--------|------|----------|---------------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook not found | ID de endpoint inexistente em qualquer operação |
| `WEBHOOK_INVALID_URL` | 422 | URL must use HTTPS | URL cadastrada ou atualizada sem `https://` [09:23, Sofia] |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Secret is required | Operação de validação de assinatura sem secret configurada |
| `WEBHOOK_INVALID_EVENT` | 422 | Invalid event type | Valor em `events` não existe no enum `OrderStatus` do Prisma |
| `WEBHOOK_ALREADY_INACTIVE` | 409 | Webhook is already inactive | Tentativa de desativar um endpoint já com `active = false` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload exceeds 64KB limit | Payload serializado ultrapassa 64 × 1024 bytes [09:24, Diego] |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | Delivery record not found | ID de registro em `webhook_delivery` inexistente |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Dead letter record not found | ID de registro em `webhook_dead_letter` inexistente |

---

## 8. Estratégias de Resiliência

| Estratégia | Configuração | Origem |
|------------|-------------|--------|
| Timeout HTTP | 10 segundos por chamada do worker | [09:42, Diego, Sofia] |
| Retry máximo | 5 tentativas | [09:15, Diego] |
| Backoff exponencial | 1m → 5m → 30m → 2h → 12h | [09:17, Diego, Larissa] |
| Dead Letter Queue | Tabela `webhook_dead_letter` separada após 5ª falha | [09:18, Diego] |
| Replay de DLQ | Manual via `POST /admin/webhooks/dead-letter/:id/replay`, role ADMIN | [09:18, Diego; 09:36, Sofia] |
| Garantia de entrega | At-least-once; cliente deduplica por `X-Event-Id` | [09:25, Diego] |
| Limite de payload | 64 KB; rejeitar com `WEBHOOK_PAYLOAD_TOO_LARGE` | [09:24, Diego, Larissa] |
| TLS obrigatório | URL deve iniciar com `https://`; validado no schema Zod | [09:23, Sofia] |

---

## 9. Observabilidade

### Logs estruturados (Pino)

O worker usa `logger` importado de `src/shared/logger/index.ts`. Campos loggados em cada etapa:

**Início de ciclo do worker:**
```json
{ "level": "debug", "msg": "worker cycle start", "pendingCount": 12 }
```

**Tentativa de entrega:**
```json
{
  "level": "info",
  "msg": "webhook delivery attempt",
  "eventId": "d9b3a7e5-...",
  "webhookId": "9e1d7f3a-...",
  "attempt": 1,
  "url": "https://cliente.com/webhook"
}
```

**Entrega bem-sucedida:**
```json
{
  "level": "info",
  "msg": "webhook delivered",
  "eventId": "d9b3a7e5-...",
  "webhookId": "9e1d7f3a-...",
  "attempt": 1,
  "statusCode": 200,
  "durationMs": 234
}
```

**Falha com retry:**
```json
{
  "level": "warn",
  "msg": "webhook delivery failed, scheduling retry",
  "eventId": "d9b3a7e5-...",
  "webhookId": "9e1d7f3a-...",
  "attempt": 2,
  "statusCode": 503,
  "durationMs": 10001,
  "error": "Timeout after 10000ms",
  "nextRetryAt": "2026-06-29T10:35:00.000Z"
}
```

**Movido para DLQ:**
```json
{
  "level": "error",
  "msg": "webhook moved to dead letter queue",
  "eventId": "d9b3a7e5-...",
  "webhookId": "9e1d7f3a-...",
  "totalAttempts": 5,
  "dlqId": "e5f6a7b8-..."
}
```

A configuração de redação (`redact`) em `src/shared/logger/index.ts` já cobre `*.token`; a secret do webhook deve ser tratada com o mesmo padrão e nunca incluída nos campos loggados.

### Métricas

Contadores e gauges a expor (via endpoint `/metrics` ou ferramenta de APM):

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_deliveries_total` | Counter | Total de tentativas de entrega, por `status` (success/failure) |
| `webhook_delivery_duration_ms` | Histogram | Duração das chamadas HTTP do worker |
| `webhook_retry_count` | Counter | Total de retentativas agrupadas por `attempt_number` |
| `webhook_dlq_total` | Counter | Total de eventos movidos para DLQ |
| `webhook_outbox_pending_total` | Gauge | Tamanho atual da fila pendente — métrica de alerta crítica para detectar worker parado |

### Tracing

O projeto não usa uma biblioteca de tracing distribuído (OpenTelemetry, Jaeger, etc.) — a abordagem adotada é propagar um `traceId` por evento como campo estruturado nos logs do Pino, permitindo correlacionar todas as entradas de log de um mesmo evento ao longo de todo o seu ciclo de vida: da inserção na outbox ao DELIVERED ou à DLQ.

**Geração do traceId:**
O `traceId` é gerado uma única vez na inserção do evento na `webhook_outbox` — coincide com o `eventId` (UUID) já persistido, eliminando a necessidade de um campo adicional. Isso garante que o mesmo identificador viaja pela transação de mudança de status, por cada tentativa do worker e pelo registro final na `webhook_delivery` ou `webhook_dead_letter`.

**Propagação nos logs:**
Todos os logs do worker incluem `eventId` como campo de correlação. Para correlacionar com o contexto da API que gerou o evento (ex.: a requisição HTTP que disparou o `changeStatus`), a função `publishWebhookEvent` recebe e persiste o `requestId` do logger Pino da requisição originadora quando disponível:

```json
{ "level": "info", "msg": "webhook outbox inserted", "eventId": "d9b3a7e5-...", "requestId": "req-abc123", "webhookId": "9e1d7f3a-...", "fromStatus": "PAID", "toStatus": "SHIPPED" }
{ "level": "info", "msg": "webhook delivery attempt", "eventId": "d9b3a7e5-...", "requestId": "req-abc123", "attempt": 1 }
{ "level": "info", "msg": "webhook delivered",        "eventId": "d9b3a7e5-...", "requestId": "req-abc123", "attempt": 1, "durationMs": 234 }
```

Com `eventId` e `requestId` indexados no sistema de log (ex.: Loki, CloudWatch Logs Insights), é possível reconstruir o caminho completo de qualquer notificação: qual requisição HTTP originou o evento → quantas tentativas ocorreram → qual foi o desfecho.

**Span do lado do cliente:**
O header `X-Event-Id` enviado em cada entrega serve também como identificador de span para o cliente: ele pode correlacionar a entrega recebida com o evento no seu próprio sistema de tracing usando esse valor.

---

## 10. Integração com o Sistema Existente

### `src/modules/orders/order.service.ts`

O método `changeStatus` (linha 126) será estendido para chamar `publishWebhookEvent(tx, refreshed, from, to)` dentro do bloco `prisma.$transaction` existente (linha 131), após `tx.orderStatusHistory.create()` e antes do `return refreshed`. A função recebe o `tx` client da transação ativa, garantindo que a inserção na `webhook_outbox` participa do mesmo commit. Se a inserção falhar, o rollback propaga para toda a transação — nenhum status é alterado sem o correspondente evento na outbox [09:40–09:41, Bruno, Diego].

### `src/shared/errors/app-error.ts`

As classes de erro do módulo de webhooks seguirão o padrão estabelecido: subclasses de `AppError` (ou das classes em `src/shared/errors/http-errors.ts`) com `errorCode` prefixado com `WEBHOOK_`. Exemplo:

```typescript
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook'); }
  // errorCode: 'NOT_FOUND', statusCode: 404
}

export class WebhookInvalidUrlError extends UnprocessableEntityError {
  constructor() {
    super('URL must use HTTPS', 'WEBHOOK_INVALID_URL');
  }
}
```

### `src/middlewares/error.middleware.ts`

O middleware centralizado já trata qualquer instância de `AppError` (linha 15) retornando `{ error: { code, message, details } }`. Como todos os erros de webhook estendem `AppError`, nenhuma modificação no middleware é necessária. Erros Zod de validação de schemas do módulo de webhooks também são capturados automaticamente (linha 26).

### `src/middlewares/auth.middleware.ts`

As rotas do módulo de webhooks usarão `authenticate` (linha 27) para verificar o JWT em todos os endpoints. O endpoint de replay de DLQ usará adicionalmente `requireRole('ADMIN')` (linha 49), que já verifica `req.user.role` contra os roles permitidos — comportamento idêntico ao que já existe em outros endpoints protegidos.

### `prisma/schema.prisma`

Serão adicionados quatro novos modelos seguindo os padrões do arquivo existente:
- `id` como `String @id @default(uuid()) @db.Char(36)` (igual a todos os modelos)
- `@@map("snake_case_name")` para nomear as tabelas MySQL
- Índices em campos de busca frequente (`status`, `created_at`, `webhookEndpointId`)
- Relações com `onDelete: Cascade` onde aplicável

Novos modelos: `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`.

### `src/config/database.ts`

O worker (`src/worker.ts`) importará `createPrismaClient` de `src/config/database.ts` e instanciará seu próprio `PrismaClient`. Não pode reutilizar a instância `prisma` exportada (ela pertence ao processo da API). Ambos apontam para o mesmo `DATABASE_URL` [09:29–09:30, Bruno, Diego]:

```typescript
// src/worker.ts
import { createPrismaClient } from './config/database.js';
const prisma = createPrismaClient();
```

---

## 11. Dependências e Compatibilidade

- **Sem nova infraestrutura:** MySQL e Prisma já existem no projeto
- **HMAC:** módulo `crypto` nativo do Node.js — `createHmac('sha256', secret).update(body).digest('hex')`
- **HTTP client no worker:** `fetch` nativo do Node.js 18+ (verificar versão em uso) ou `undici` como alternativa
- **UUID:** `crypto.randomUUID()` nativo do Node.js — sem nova dependência, seguindo padrão UUID já adotado no Prisma (`@default(uuid())`)

---

## 12. Critérios de Aceite Técnicos

- [ ] Inserção na outbox é atômica com a mudança de status: rollback em `changeStatus` não cria evento na `webhook_outbox`
- [ ] Worker em processo separado (`src/worker.ts`) com polling de 2s, `npm run worker`
- [ ] Retry com 5 tentativas e backoff exato: 1m / 5m / 30m / 2h / 12h
- [ ] Após 5ª falha, evento movido para `webhook_dead_letter` com payload e motivo
- [ ] HMAC-SHA256 calculado sobre o body; header `X-Signature` presente em toda entrega
- [ ] Secret única por endpoint; gerada pela plataforma no cadastro; não retornada em leituras subsequentes
- [ ] Rotação de secret mantém a anterior válida por exatamente 24 horas (grace period)
- [ ] URL com `http://` ou inválida rejeitada com `WEBHOOK_INVALID_URL` (HTTP 422)
- [ ] Payload acima de 64 KB rejeitado com `WEBHOOK_PAYLOAD_TOO_LARGE` (HTTP 422)
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` requer role `ADMIN`; gera log de auditoria com `userId` e timestamp
- [ ] `GET /webhooks/:id/deliveries` retorna histórico de tentativas com `statusCode`, `durationMs` e `errorMessage`
- [ ] Filtro de eventos aplicado na inserção da outbox — webhooks que não assinaram o `toStatus` não recebem linha na outbox
- [ ] Testes de integração cobrem: fluxo happy path, rollback da transação, retry até DLQ e replay

---

## 13. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Crescimento não controlado da `webhook_outbox` | Média | Médio | Arquivamento de eventos com status `DELIVERED` após 30 dias; fora do escopo desta fase, mas índice em `status` + `created_at` mantém queries baratas [09:08, Diego] |
| Vazamento de secret no log do worker | Baixa | Alto | Campo `secret` nunca incluído em logs; padrão `redact` do Pino já cobre `*.token`; Sofia revisará o código antes do deploy [09:46, Sofia] |
| Worker parado sem monitoramento | Média | Alto | Métrica `webhook_outbox_pending_total` com alerta quando fila cresce acima de threshold; worker deve logar heartbeat periódico |
| Clientes sem deduplicação por `X-Event-Id` | Alta | Baixo | At-least-once documentado de forma destacada no portal de desenvolvedor; Marcos comprometeu-se com isso [09:26, Marcos] |
| Falha na inserção da outbox causa rollback do status | Baixa | Alto | Comportamento intencional e correto; monitorar taxa de erros em `changeStatus` após deploy; testes de integração obrigatórios [09:40–09:41, Bruno, Diego] |
