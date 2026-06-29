---
description: Gera o FDD (Feature Design Document) completo e acionável do Sistema de Webhooks — especificação de implementação com fluxos, contratos, erros e integração com o código existente
---

Você vai gerar o arquivo `docs/FDD.md` para o Sistema de Webhooks de Notificação de Pedidos.

## Antes de gerar

Leia obrigatoriamente:
- `TRANSCRICAO.md` — transcrição completa da reunião técnica
- `prisma/schema.prisma` — modelos de dados existentes (Order, OrderStatus, User, Customer)
- `src/modules/orders/order.service.ts` — método `changeStatus` que será estendido (linhas 126–178)
- `src/shared/errors/app-error.ts` — classe base de erros
- `src/shared/errors/http-errors.ts` — classes de erro específicas (BadRequestError, NotFoundError, ConflictError, etc.)
- `src/shared/errors/index.ts` — exports do módulo de erros
- `src/modules/orders/order.routes.ts` — padrão de rotas existente
- `src/middlewares/auth.middleware.ts` — middleware de autenticação existente
- `src/middlewares/error.middleware.ts` — middleware de erro centralizado
- `src/config/database.ts` — configuração do Prisma
- `docs/adrs/` — ADRs gerados (para não contradizer decisões já registradas)

## O que é o FDD neste contexto

O FDD é o documento mais técnico — **especificação de implementação** detalhada o suficiente para um desenvolvedor pegar e começar a codar. Vai mais fundo que o RFC e fala em "como construir", não em "o que propomos".

## Estrutura obrigatória

### 1. Contexto e Motivação Técnica
Por que precisamos desta feature e qual o impacto na arquitetura atual.

### 2. Objetivos Técnicos
Lista de objetivos mensuráveis (ex: latência máxima de 10s entre mudança de status e entrega).

### 3. Escopo e Exclusões
O que está dentro e o que está fora. Itens explicitamente excluídos da reunião:
- Webhook inbound (clientes enviando para o sistema) — descartado em [09:02]
- Email de alerta por falha — adiado em [09:37]
- Rate limiting de saída — adiado em [09:39]
- Dashboard visual — fora de escopo em [09:40]
- Garantia exactly-once — descartada em [09:25]
- Escalonamento com múltiplos workers (ordering global) — adiado em [09:13]

### 4. Fluxos Detalhados

Descreva os 4 fluxos principais com diagrama em texto (ASCII ou Mermaid):

**Fluxo 1: Inserção na outbox (dentro da transação do changeStatus)**
- `OrderService.changeStatus()` abre transação Prisma
- Atualiza `orders`, insere em `order_status_history`, ajusta estoque
- Chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação
- `publishWebhookEvent` consulta webhooks ativos do customer que filtram aquele status
- Para cada webhook correspondente, insere linha em `webhook_outbox` com payload snapshot e `event_id` UUID
- Commit atômico — se qualquer parte falhar, rollback de tudo

**Fluxo 2: Processamento pelo worker (polling)**
- Loop a cada 2 segundos
- Busca eventos `status = PENDING` ordenados por `created_at` (batch pequeno)
- Para cada evento: lê configuração do webhook, calcula HMAC-SHA256 do payload, envia HTTP POST
- Timeout: 10 segundos
- Sucesso (2xx): marca evento como `DELIVERED` em `webhook_outbox`
- Falha: incrementa `attempt_count`, atualiza `next_retry_at` com backoff

**Fluxo 3: Retry com backoff exponencial**
- 5 tentativas máximas
- Progressão: 1min / 5min / 30min / 2h / 12h após cada falha
- Após 5ª falha: move para `webhook_dead_letter` com payload, motivo e timestamp

**Fluxo 4: Replay de DLQ**
- `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN obrigatório)
- Reinsere evento na `webhook_outbox` com `status = PENDING`, `attempt_count = 0`
- Log de auditoria: quem fez o replay e quando

### 5. Contratos Públicos (Endpoints HTTP)

Inclua **pelo menos 4 endpoints** com payload completo de request, response e status codes:

1. `POST /webhooks` — cadastrar endpoint de webhook
2. `GET /webhooks?customerId=:id` — listar webhooks de um customer
3. `PATCH /webhooks/:id` — atualizar configuração
4. `DELETE /webhooks/:id` — remover endpoint
5. `POST /webhooks/:id/secret/rotate` — rotacionar secret (grace period 24h)
6. `GET /webhooks/:id/deliveries` — histórico dos últimos envios
7. `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar item da DLQ (role ADMIN)

Para cada endpoint, mostre:
```
POST /webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

Request body:
{
  "customerId": "uuid",
  "url": "https://cliente.com/webhook",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "description": "Webhook principal Atlas Comercial"
}

Response 201:
{
  "id": "uuid",
  "customerId": "uuid",
  "url": "https://cliente.com/webhook",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_...",   // retornado apenas na criação
  "active": true,
  "createdAt": "2026-06-29T10:00:00Z"
}
```

### 6. Payload do Webhook (enviado ao cliente)

```json
{
  "event_id": "uuid-gerado-na-outbox",
  "event_type": "order.status_changed",
  "timestamp": "2026-06-29T10:00:00Z",
  "order_id": "uuid",
  "order_number": "ORD-000042",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "uuid",
  "total_cents": 15000
}
```

Headers enviados:
```
X-Event-Id: <uuid do evento>
X-Webhook-Id: <uuid do endpoint webhook>
X-Signature: <hmac-sha256 hex do payload>
X-Timestamp: <timestamp do envio ISO 8601>
Content-Type: application/json
```

### 7. Matriz de Erros

Todos os códigos devem usar o prefixo `WEBHOOK_`:

| Código | HTTP | Mensagem | Quando ocorre |
|--------|------|----------|---------------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook not found | ID inexistente |
| `WEBHOOK_INVALID_URL` | 422 | URL must use HTTPS | URL com http:// ou inválida |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Secret is required | Secret ausente na operação que exige |
| `WEBHOOK_INVALID_EVENT` | 422 | Invalid event type | Status não existe no enum OrderStatus |
| `WEBHOOK_ALREADY_INACTIVE` | 409 | Webhook is already inactive | Tentativa de desativar webhook já inativo |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload exceeds 64KB limit | Payload gerado ultrapassa limite |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | Delivery record not found | ID de delivery inexistente |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Dead letter record not found | ID de DLQ inexistente |

### 8. Estratégias de Resiliência

- **Timeout:** 10s por chamada HTTP do worker (decisão [09:42] Diego)
- **Retry:** 5 tentativas, backoff 1m/5m/30m/2h/12h (decisão [09:15]/[09:17] Diego e Larissa)
- **DLQ:** tabela `webhook_dead_letter` separada após 5ª falha (decisão [09:18] Diego)
- **At-least-once:** possibilidade de entrega duplicada; cliente deduplicar por `X-Event-Id` (decisão [09:25] Diego)
- **Limite de payload:** 64KB máximo; rejeitar com erro se ultrapassar (decisão [09:24] Larissa)
- **TLS obrigatório:** URL deve ser `https://` (decisão [09:23] Sofia)

### 9. Observabilidade

Descreva:
- **Métricas** (ex: `webhook_deliveries_total`, `webhook_delivery_duration_ms`, `webhook_retry_count`, `webhook_dlq_total`)
- **Logs estruturados** com Pino — o que logar em cada etapa do worker (event_id, webhook_id, attempt, status_code, duration_ms, error)
- **Tracing** — o projeto não usa OpenTelemetry/Jaeger; adote propagação de `traceId` via campos estruturados no Pino. O `eventId` (UUID gerado na inserção da outbox) serve como `traceId` natural — o mesmo valor viaja por todos os logs do worker (inserção, tentativas, DELIVERED, DLQ). Quando disponível, propague também o `requestId` da requisição HTTP originadora (do `changeStatus`) para correlacionar a entrega com a chamada da API que a disparou. O header `X-Event-Id` enviado ao cliente serve como identificador de span no sistema de tracing do receptor.

### 10. Integração com o Sistema Existente

**Esta seção é obrigatória e específica deste desafio.**

Descreva como o módulo de webhooks se integra com cada um dos arquivos abaixo (cite o caminho exato):

1. `src/modules/orders/order.service.ts` — o método `changeStatus` será estendido para chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação Prisma (linhas 131–178). A função recebe o `tx` client, garantindo atomicidade com o commit de mudança de status.

2. `src/shared/errors/app-error.ts` — as classes de erro do módulo de webhooks (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) vão estender `AppError` seguindo o padrão já estabelecido no projeto.

3. `src/middlewares/error.middleware.ts` — o middleware centralizado já captura instâncias de `AppError` e formata a resposta. Os erros `WEBHOOK_*` serão tratados automaticamente sem alteração no middleware.

4. `src/middlewares/auth.middleware.ts` — os endpoints de webhook usarão o mesmo middleware de autenticação JWT existente. O endpoint de replay de DLQ usará adicionalmente o `requireRole('ADMIN')` já disponível.

5. `prisma/schema.prisma` — serão adicionados os modelos `WebhookEndpoint`, `WebhookOutbox` e `WebhookDeadLetter` seguindo os padrões existentes (UUID como PK, `@@map` para snake_case, índices por status e `created_at`).

6. `src/config/database.ts` — o worker importará o `PrismaClient` via este módulo, mas instanciará uma conexão separada (mesmo `DATABASE_URL`, processo diferente).

### 11. Dependências e Compatibilidade

- Sem dependências de nova infra (MySQL já existe, Prisma já existe)
- Biblioteca para HMAC: `crypto` nativo do Node.js (sem dependência externa)
- Biblioteca para HTTP client no worker: `fetch` nativo do Node.js 18+ ou `undici`

### 12. Critérios de Aceite Técnicos

Lista verificável de condições para considerar a feature completa:
- [ ] Inserção na outbox é atômica com a mudança de status (rollback propaga)
- [ ] Worker em processo separado (`src/worker.ts`) com polling de 2s
- [ ] Retry com 5 tentativas e backoff 1m/5m/30m/2h/12h
- [ ] DLQ persistida em `webhook_dead_letter`
- [ ] HMAC-SHA256 calculado sobre o body; header `X-Signature` presente
- [ ] Secret única por endpoint; rotação com grace period de 24h
- [ ] URL com `http://` rejeitada com `WEBHOOK_INVALID_URL`
- [ ] Payload acima de 64KB rejeitado com `WEBHOOK_PAYLOAD_TOO_LARGE`
- [ ] Endpoint de replay requer role ADMIN e registra auditoria
- [ ] `GET /webhooks/:id/deliveries` retorna histórico de tentativas
- [ ] Testes de integração cobrem fluxo happy path e retry

### 13. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Crescimento não controlado da tabela `webhook_outbox` | Média | Médio | Arquivamento de eventos entregues após 30 dias (fora do escopo desta fase) |
| Vazar secret no log de aplicação do cliente | Baixa | Alto | Documentar explicitamente no portal que a secret é sensível; não retornar secret em endpoints de leitura |
| Worker parado sem monitoramento | Média | Alto | Métrica de fila crescente (`webhook_outbox_pending_total`) com alerta |
| Clientes não implementarem deduplicação por `X-Event-Id` | Alta | Baixo | Documentar at-least-once no portal do desenvolvedor (Marcos, [09:26]) |

## Regras de qualidade

- Toda informação deve ter origem rastreável na transcrição (timestamp + nome) ou no código (caminho do arquivo)
- Não invente endpoints, campos ou comportamentos não discutidos
- Não contradiga nenhuma decisão registrada nos ADRs
- A seção "Integração com o sistema existente" deve citar os caminhos de arquivo exatos e existentes no repositório

## Saída esperada

Escreva o conteúdo diretamente em `docs/FDD.md`. Após gerar, confirme que a seção "Contratos Públicos" tem pelo menos 4 endpoints e que a seção "Integração com o sistema existente" cita pelo menos 4 arquivos reais.
