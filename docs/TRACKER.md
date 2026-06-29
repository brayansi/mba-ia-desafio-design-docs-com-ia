# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

**Data:** 2026-06-29
**Fonte primária:** `TRANSCRICAO.md` | **Fonte de código:** repositório local

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint de webhook com URL HTTPS, lista de eventos e vínculo a um customer | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | URL do webhook deve ser obrigatoriamente HTTPS; `http://` rejeitado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma na criação, retornada apenas uma vez; não exposta em leituras | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24 horas para migração sem downtime | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Disparo automático de evento quando status muda, apenas para webhooks que assinaram aquela transição | TRANSCRICAO | [09:33] Marcos |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Payload inclui: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Headers obrigatórios: X-Event-Id, X-Webhook-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Retry automático em caso de falha: 5 tentativas com backoff exponencial 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Após 5 falhas consecutivas, evento movido para Dead Letter Queue com payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Administradores reprocessam item da DLQ manualmente; ação registrada em log de auditoria com identificação do admin | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Histórico de deliveries por endpoint com status, código HTTP, tempo de resposta e mensagem de erro | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Editar URL, lista de eventos e estado ativo/inativo de um endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência máxima entre mudança de status e entrega: 10 segundos (p95) | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP do worker; sem resposta em 10s = falha | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Tamanho máximo de payload: 64 KB; payloads maiores rejeitados com erro | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; cliente é responsável por deduplicar usando X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Autenticação via HMAC-SHA256 sobre o corpo da requisição, com secret única por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Sem nova dependência de infraestrutura; somente MySQL já existente | TRANSCRICAO | [09:07] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API; restart da API não interrompe processamento | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Logs estruturados via Pino em todos os passos do worker | TRANSCRICAO | [09:29] Bruno |
| PRD-ESC-01 | docs/PRD.md | Exclusão de Escopo | Webhook inbound (clientes enviando eventos para o sistema) — descartado | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-02 | docs/PRD.md | Exclusão de Escopo | Notificação por e-mail em caso de falha no webhook — adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-03 | docs/PRD.md | Exclusão de Escopo | Dashboard visual para o cliente acompanhar webhooks — fora de escopo desta fase | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-04 | docs/PRD.md | Exclusão de Escopo | Rate limiting de envio por cliente — adiado para observação em produção | TRANSCRICAO | [09:39] Larissa |
| PRD-ESC-05 | docs/PRD.md | Exclusão de Escopo | Garantia exactly-once — descartada; at-least-once com X-Event-Id é o padrão adotado | TRANSCRICAO | [09:25] Diego |
| PRD-ESC-06 | docs/PRD.md | Exclusão de Escopo | Escalonamento com múltiplos workers e garantia de ordering global — adiado | TRANSCRICAO | [09:13] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Clientes sem deduplicação por X-Event-Id causando ações duplicadas nos sistemas deles | TRANSCRICAO | [09:26] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Atlas migrar para concorrente se entrega atrasar além do fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-03 | docs/PRD.md | Risco | Worker parado silenciosamente sem alertas, acumulando fila de eventos não entregues | TRANSCRICAO | [09:11] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Acúmulo de eventos na tabela webhook_outbox comprometendo performance | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Vazamento de secret de webhook nos logs da plataforma ou do cliente | TRANSCRICAO | [09:22] Diego |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Webhook síncrono dentro do OrderService — descartado por travar transação principal com clientes lentos ou offline | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams como fila de eventos — descartado por exigir nova infraestrutura desnecessária para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-QA-01 | docs/RFC.md | Restrição | Rate limiting de envio por cliente — questão em aberto, adiada para observação pós-lançamento | TRANSCRICAO | [09:38] Diego |
| RFC-QA-02 | docs/RFC.md | Restrição | Notificação por e-mail em caso de falha de webhook — questão em aberto, adiada para próxima fase | TRANSCRICAO | [09:37] Marcos |
| RFC-QA-03 | docs/RFC.md | Restrição | Garantia de ordering com múltiplos workers — limitação conhecida documentada, solução futura por particionamento | TRANSCRICAO | [09:13] Diego |
| FDD-API-01 | docs/FDD.md | Contrato de API | POST /webhooks — cadastrar endpoint; retorna secret apenas na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-API-02 | docs/FDD.md | Contrato de API | GET /webhooks?customerId=:id — listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-API-03 | docs/FDD.md | Contrato de API | PATCH /webhooks/:id — atualizar URL, eventos e estado ativo/inativo | TRANSCRICAO | [09:33] Bruno |
| FDD-API-04 | docs/FDD.md | Contrato de API | DELETE /webhooks/:id — remover endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-API-05 | docs/FDD.md | Contrato de API | POST /webhooks/:id/secret/rotate — rotacionar secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-API-06 | docs/FDD.md | Contrato de API | GET /webhooks/:id/deliveries — histórico dos últimos envios com status, código HTTP e duração | TRANSCRICAO | [09:34] Marcos |
| FDD-API-07 | docs/FDD.md | Contrato de API | POST /admin/webhooks/dead-letter/:id/replay — reprocessar item da DLQ, exige role ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND (404) — ID de endpoint inexistente em qualquer operação | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição | WEBHOOK_INVALID_URL (422) — URL sem https:// | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Restrição | WEBHOOK_SECRET_REQUIRED (422) — secret ausente em operação que a exige | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição | WEBHOOK_INVALID_EVENT (422) — valor em events não existe no enum OrderStatus | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-05 | docs/FDD.md | Restrição | WEBHOOK_ALREADY_INACTIVE (409) — tentativa de desativar endpoint já inativo | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-06 | docs/FDD.md | Restrição | WEBHOOK_PAYLOAD_TOO_LARGE (422) — payload acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-07 | docs/FDD.md | Restrição | WEBHOOK_DELIVERY_NOT_FOUND (404) — ID de registro de delivery inexistente | TRANSCRICAO | [09:34] Marcos |
| FDD-ERR-08 | docs/FDD.md | Restrição | WEBHOOK_DLQ_NOT_FOUND (404) — ID de registro na DLQ inexistente | TRANSCRICAO | [09:35] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus estendido para chamar publishWebhookEvent(tx, order, fromStatus, toStatus) dentro da mesma transação Prisma | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Erros de webhook estendem AppError com errorCode prefixado WEBHOOK_; capturados automaticamente pelo error middleware | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Integração | errorMiddleware captura AppError sem modificação; erros WEBHOOK_* tratados automaticamente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Rotas de webhook usam authenticate; replay de DLQ usa requireRole('ADMIN') existente | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Novos modelos WebhookEndpoint, WebhookOutbox, WebhookDelivery, WebhookDeadLetter seguem padrões UUID/@@map/índices existentes | CODIGO | prisma/schema.prisma |
| FDD-INT-06 | docs/FDD.md | Integração | Worker importa createPrismaClient() e instancia PrismaClient próprio com mesmo DATABASE_URL | CODIGO | src/config/database.ts |
| FDD-RES-01 | docs/FDD.md | Restrição | Timeout de 10 segundos por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Restrição | Máximo de 5 tentativas de entrega por evento | TRANSCRICAO | [09:15] Diego |
| FDD-RES-03 | docs/FDD.md | Restrição | Backoff exponencial: 1m → 5m → 30m → 2h → 12h (~15h janela total) | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-04 | docs/FDD.md | Restrição | Após 5ª falha: evento movido para webhook_dead_letter com payload e motivo | TRANSCRICAO | [09:18] Diego |
| FDD-RES-05 | docs/FDD.md | Restrição | At-least-once: X-Event-Id estável entre retentativas para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| FDD-RES-06 | docs/FDD.md | Restrição | Limite de 64 KB por payload; rejeitar com erro se ultrapassar | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | docs/FDD.md | Restrição | TLS obrigatório: URL deve iniciar com https://, validado no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-mysql.md | Decisão | Usar padrão Transactional Outbox no MySQL existente para publicação atômica de eventos de webhook | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-exponencial-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h; após esgotamento, DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003 | docs/adrs/ADR-003-autenticacao-hmac-sha256.md | Decisão | HMAC-SHA256 sobre o payload com secret única por endpoint; rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-x-event-id.md | Decisão | Garantia at-least-once; UUID gerado na inserção da outbox enviado como X-Event-Id em todas as tentativas | TRANSCRICAO | [09:26] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado (src/worker.ts) com polling a cada 2 segundos; MySQL sem trigger nativo NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes-projeto.md | Decisão | Módulo webhooks segue padrão existente: AppError, Pino, error middleware, estrutura src/modules/webhooks/, prefixo WEBHOOK_ | CODIGO | src/shared/errors/http-errors.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload JSON serializado na inserção da outbox (snapshot); não lazy rendering no momento do envio | TRANSCRICAO | [09:52] Larissa |

---

## Resumo de Cobertura

| Métrica | Valor |
|---------|-------|
| **Total de linhas na tabela** | 71 |
| **Fonte = TRANSCRICAO** | 64 (90,1%) |
| **Fonte = CODIGO** | 7 (9,9%) |
| **Cobertura mínima de 80% atingida** | Sim |
| **≥ 70% com TRANSCRICAO** | Sim (90,1%) |
| **≥ 5 linhas com CODIGO** | Sim (7 linhas) |

### Arquivos de código referenciados

| Arquivo | Referenciado em |
|---------|----------------|
| `src/modules/orders/order.service.ts` | FDD-INT-01 |
| `src/shared/errors/app-error.ts` | FDD-INT-02 |
| `src/middlewares/error.middleware.ts` | FDD-INT-03 |
| `src/middlewares/auth.middleware.ts` | FDD-INT-04 |
| `prisma/schema.prisma` | FDD-INT-05 |
| `src/config/database.ts` | FDD-INT-06 |
| `src/shared/errors/http-errors.ts` | FDD-ERR-05, ADR-006 |

### Itens obrigatórios — checagem final

| Grupo | Exigido | Presente |
|-------|---------|----------|
| PRD — Requisitos Funcionais (RF-01 a RF-12) | 12 | 12 ✓ |
| PRD — Itens fora de escopo | ≥ 2 | 6 ✓ |
| PRD — Riscos | ≥ 2 | 5 ✓ |
| RFC — Alternativas descartadas | ≥ 2 | 2 ✓ |
| RFC — Questões em aberto | ≥ 2 | 3 ✓ |
| FDD — Contratos de API | ≥ 4 | 7 ✓ |
| FDD — Integrações com código | ≥ 4 | 6 ✓ |
| FDD — Erros WEBHOOK_* | ≥ 1 | 8 ✓ |
| ADRs — 1 linha por ADR | 7 | 7 ✓ |
