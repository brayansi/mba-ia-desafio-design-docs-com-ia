# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo         | Valor                                                                          |
|---------------|--------------------------------------------------------------------------------|
| **Autor**     | Marcos (Product Manager)                                                       |
| **Status**    | Em revisão                                                                     |
| **Data**      | 2026-06-29                                                                     |
| **Revisores** | Larissa (Tech Lead), Sofia (Eng. Segurança), Bruno (Eng. Pleno, Pedidos)       |

---

## 1. Resumo e Contexto

O Sistema de Webhooks de Notificação de Pedidos é uma feature que permite à plataforma notificar proativamente clientes B2B integrados via API sobre mudanças no status dos seus pedidos. Em vez de os clientes consultarem repetidamente o endpoint `GET /orders` para verificar alterações, eles cadastram um endpoint próprio e passam a receber os eventos automaticamente assim que ocorrem.

A demanda surgiu de um pedido formal de três clientes estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — que precisam reagir em tempo real a eventos de pedido (como despacho, entrega e cancelamento) para acionar seus próprios sistemas de logística, faturamento e controle de estoque [09:00, Marcos].

---

## 2. Problema e Motivação

O modelo atual de integração exige que os clientes façam polling contínuo no `GET /orders` para detectar mudanças de status. Isso é lento, caro para o cliente e gera carga desnecessária na nossa API. Os clientes consideram "tempo real" qualquer resposta abaixo de 10 segundos — algo que o polling nunca entrega de forma consistente [09:02, Marcos].

A urgência é concreta: a Atlas Comercial sinalizou que, se a solução não for entregue até o fim do trimestre, eles podem migrar para um concorrente que já oferece webhooks [09:00, Marcos]. Manter o modelo de polling significa perder pelo menos um cliente estratégico e deixar os outros dois com integração degradada.

---

## 3. Público-alvo e Cenários de Uso

**Público-alvo:** clientes B2B integrados via API com necessidade de reação automatizada a eventos de pedido. Os clientes iniciais são Atlas Comercial, MaxDistribuição e Nova Cargo [09:00, Marcos]. A integração é exclusivamente outbound — a plataforma envia eventos para os clientes, não o contrário [09:02, Sofia e Marcos].

**Cenários principais:**

- **Acionar logística própria:** cliente quer receber notificação quando um pedido muda para `SHIPPED`, disparando automaticamente o rastreamento na transportadora parceira dele.
- **Reverter cobrança interna:** cliente quer saber quando um pedido é `CANCELLED` para estornar a cobrança no ERP ou sistema financeiro próprio, sem depender de atualização manual.
- **Confirmar recebimento:** cliente quer monitorar `DELIVERED` para fechar o ciclo de compra no sistema dele e liberar pagamento ao fornecedor.
- **Auditar transições:** cliente que gerencia múltiplos compradores quer receber todos os eventos de `PAID` em diante para alimentar um painel de controle próprio.

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Reduzir latência de notificação | Tempo entre mudança de status e entrega do webhook (p95) | < 10 segundos |
| Adotar os 3 clientes iniciais | Número de clientes com pelo menos 1 webhook ativo | 3 clientes até fim do trimestre |
| Alta confiabilidade de entrega | Taxa de entrega bem-sucedida (excluindo endpoints permanentemente offline) | ≥ 99% |
| Reduzir carga de polling na API | Volume de chamadas ao `GET /orders` pelos 3 clientes B2B após o lançamento | Redução ≥ 70% |
| Segurança de integração | Nenhum incidente de autenticação ou vazamento de secret em produção | 0 incidentes no trimestre de lançamento |

---

## 5. Escopo

### Incluso nesta fase

- Cadastro, edição, listagem e remoção de endpoints de webhook por customer
- Entrega automática de eventos quando o status de um pedido muda
- Filtro de eventos por status — o cliente configura quais transições quer receber (ex.: só `SHIPPED` e `DELIVERED`)
- Autenticação de cada entrega com HMAC-SHA256 e secret única por endpoint
- Rotação de secret com grace period de 24 horas para migração sem downtime
- Retry automático com backoff exponencial em caso de falha do endpoint do cliente (5 tentativas: 1m / 5m / 30m / 2h / 12h)
- Dead Letter Queue (DLQ) persistida para eventos que esgotam todas as tentativas
- Replay manual de itens da DLQ por administradores, com auditoria de quem executou
- Histórico de entregas por endpoint, com status, código HTTP, tempo de resposta e mensagem de erro

### Fora de escopo (descartado ou adiado)

- **Webhook inbound** (clientes enviando eventos para o sistema) — descartado na reunião: "Só saindo da gente pra eles. Eles querem receber, não mandar" [09:02, Sofia e Marcos].
- **Notificação por e-mail em caso de falha** — explicitamente adiada: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" [09:37, Larissa].
- **Dashboard visual** para o cliente acompanhar webhooks — fora de escopo: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" [09:40, Larissa].
- **Rate limiting de envio por cliente** — adiado para observação em produção: "Fica como 'observar e decidir depois'" [09:39, Larissa].
- **Garantia exactly-once** — descartada: at-least-once com deduplicação pelo cliente é o padrão adotado, conforme o mercado (Stripe, GitHub) [09:25, Diego].
- **Escalonamento com múltiplos workers** — adiado como problema futuro, sem requisito imediato dos clientes [09:13, Diego].

---

## 6. Requisitos Funcionais

| ID | Requisito |
|----|-----------|
| RF-01 | O sistema deve permitir cadastrar um endpoint de webhook com URL HTTPS, lista de eventos de interesse e vínculo a um customer |
| RF-02 | A URL do webhook deve ser obrigatoriamente HTTPS; URLs com `http://` devem ser rejeitadas com erro de validação |
| RF-03 | A secret do webhook deve ser gerada pela plataforma no momento da criação e retornada ao cliente apenas uma vez; endpoints de leitura subsequentes não devem expor a secret |
| RF-04 | O cliente deve poder solicitar a rotação da secret; a secret anterior deve permanecer válida por 24 horas após a rotação para que o cliente migre seus sistemas sem interrupção |
| RF-05 | O sistema deve disparar um evento automaticamente quando o status de um pedido muda, apenas para webhooks cujo filtro de eventos inclua aquela transição |
| RF-06 | O payload de cada evento deve incluir: identificador único do evento, tipo do evento, timestamp, identificadores e número do pedido, status anterior, status novo, identificador do customer e valor total do pedido |
| RF-07 | Cada entrega deve incluir os headers de autenticação e rastreamento: `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type: application/json` |
| RF-08 | O sistema deve retentar a entrega automaticamente em caso de falha, com backoff exponencial de 5 tentativas (1 min / 5 min / 30 min / 2 h / 12 h) |
| RF-09 | Após 5 falhas consecutivas, o evento deve ser movido para a Dead Letter Queue com registro do payload, motivo da falha e timestamp |
| RF-10 | Administradores devem poder reprocessar manualmente um item da DLQ via endpoint dedicado; a ação deve ser registrada em log de auditoria com identificação do administrador |
| RF-11 | O cliente deve poder consultar o histórico das últimas entregas de um webhook, com status de cada tentativa, código HTTP de resposta, tempo de resposta e mensagem de erro quando houver |
| RF-12 | O sistema deve permitir editar a URL, os eventos de interesse e o estado ativo/inativo de um endpoint de webhook |

---

## 7. Requisitos Não Funcionais

| ID | Requisito |
|----|-----------|
| RNF-01 | Latência máxima entre mudança de status e entrega do webhook: 10 segundos (p95) em condições normais de operação [09:02, Marcos; 09:10, Larissa] |
| RNF-02 | Timeout de 10 segundos por chamada HTTP do worker; endpoint do cliente que não responder em 10s é tratado como falha [09:42, Diego, Sofia] |
| RNF-03 | Tamanho máximo de payload: 64 KB; payloads maiores são rejeitados com erro [09:24, Diego] |
| RNF-04 | Garantia de entrega at-least-once: o mesmo evento pode ser entregue mais de uma vez em cenários de retry; o cliente é responsável por deduplicar usando o `X-Event-Id` [09:25, Diego] |
| RNF-05 | Autenticação de cada entrega via HMAC-SHA256 sobre o corpo da requisição, com secret única por endpoint [09:20–09:21, Sofia] |
| RNF-06 | O módulo não deve introduzir nova dependência de infraestrutura; nenhum Redis, filas externas ou serviço adicional além do MySQL existente [09:07, Diego] |
| RNF-07 | O worker de entrega deve rodar como processo separado da API; reinicialização da API não deve interromper o processamento de webhooks [09:11, Diego] |
| RNF-08 | Todos os passos de processamento do worker devem ser logados de forma estruturada, com identificadores de evento, webhook, tentativa, status HTTP e duração |

---

## 8. Decisões e Trade-offs Principais

| Decisão | Impacto no produto |
|---------|--------------------|
| Padrão Outbox no MySQL (ver [ADR-001](adrs/ADR-001-padrao-outbox-mysql.md)) | Eventos publicados são sempre consistentes com o estado do pedido — não há risco de notificar uma mudança que foi revertida. Não exige nova infraestrutura. |
| At-least-once com `X-Event-Id` (ver [ADR-004](adrs/ADR-004-garantia-at-least-once-x-event-id.md)) | Simplifica o sistema e segue o padrão de mercado (Stripe, GitHub). O cliente precisa implementar deduplicação no lado dele, o que deve ser documentado de forma clara no portal. |
| Secret por endpoint com rotação (ver [ADR-003](adrs/ADR-003-autenticacao-hmac-sha256.md)) | Segurança granular: o vazamento da secret de um cliente não compromete os demais. Grace period de 24h na rotação elimina downtime durante migrações. |
| Retry com DLQ (ver [ADR-002](adrs/ADR-002-retry-backoff-exponencial-dlq.md)) | Cobre indisponibilidades de até ~15 horas sem perda de evento. Eventos que não foram entregues ficam acessíveis para replay — nenhuma notificação é silenciosamente descartada. |
| Worker em processo separado (ver [ADR-005](adrs/ADR-005-worker-processo-separado-polling.md)) | Falhas ou deploys da API não interrompem a entrega de webhooks em andamento. Exige orquestração de um segundo processo em produção. |

---

## 9. Dependências

| Dependência | Responsável | Observação |
|-------------|-------------|------------|
| Implementação do módulo de webhooks e worker | Bruno (Pedidos) e Diego (Plataforma) | Estimativa de 3 sprints incluindo revisão de segurança [09:46, Larissa] |
| Revisão de segurança do código de HMAC e geração de secret | Sofia (Eng. Segurança) | Mínimo 2 dias úteis antes do deploy; bloqueante para produção [09:46, Sofia] |
| Documentação no portal do desenvolvedor | Marcos (PM) | Contrato de integração, semântica at-least-once e exemplos de deduplicação por `X-Event-Id` [09:26, Marcos; 09:40, Marcos] |
| Implementação do endpoint receptor | Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) | Cada cliente precisa expor um endpoint HTTPS e implementar deduplicação por `X-Event-Id` |
| Validação em ambiente de staging | Atlas Comercial (cliente piloto) | Cliente prioritário dado o risco de churn; validação antes do lançamento em produção |

---

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Clientes não implementarem deduplicação por `X-Event-Id`, causando ações duplicadas em seus sistemas | Alta | Médio | Documentar at-least-once com destaque no portal do desenvolvedor; oferecer exemplos de código de deduplicação; Marcos comprometeu-se com essa documentação [09:26, Marcos] |
| Atlas migrar para concorrente se a entrega atrasar além do prazo do trimestre | Média | Alto | Priorizar entrega básica (cadastro + entrega funcional) nas primeiras sprints; funcionalidades avançadas (replay de DLQ, histórico) nas seguintes |
| Worker parar silenciosamente sem alertas, acumulando fila de eventos não entregues | Média | Alto | Métrica de monitoramento do tamanho da fila pendente com alerta quando crescer acima do threshold; log de heartbeat periódico do worker |
| Acúmulo de eventos entregues na tabela de outbox comprometendo performance de queries | Baixa | Médio | Monitorar tamanho da tabela após lançamento; arquivamento de eventos entregues após 30 dias planejado para fase posterior [09:08, Diego] |
| Vazamento de secret nos logs da plataforma ou do cliente | Baixa | Alto | Secret nunca retornada em endpoints de leitura; orientação explícita no portal; revisão de código de segurança por Sofia antes do deploy [09:46, Sofia] |

---

## 11. Critérios de Aceitação

- [ ] Os 3 clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) conseguem cadastrar um endpoint e receber eventos de mudança de status
- [ ] Latência p95 abaixo de 10 segundos medida em ambiente de produção com carga normal
- [ ] Clientes param de fazer polling no `GET /orders` para monitorar mudanças de status após ativar o webhook
- [ ] Evento de mudança de status não é perdido mesmo que o endpoint do cliente esteja offline no momento — o retry entrega quando o endpoint volta
- [ ] Auditoria de quem fez replay de DLQ está registrada com identificação do administrador e timestamp
- [ ] Sofia revisou e aprovou o código de geração de secret e implementação de HMAC antes do deploy em produção
- [ ] Portal do desenvolvedor documenta a semântica at-least-once e como deduplicar por `X-Event-Id`

---

## 12. Estratégia de Testes e Validação

- **Testes de integração:** fluxo completo de mudança de status → inserção na outbox → processamento pelo worker → entrega confirmada; cobrindo também rollback da transação (evento não deve ser criado se a mudança de status falhar)
- **Testes de falha:** endpoint do cliente retornando erro 500 e timeout → verificar progressão correta do backoff e movimentação para DLQ após 5 tentativas
- **Revisão de segurança:** Sofia valida a geração de secret, o cálculo de HMAC-SHA256, a rotação com grace period e a proteção contra replay attack pelo `X-Timestamp` [09:46, Sofia]
- **Validação com cliente piloto:** Atlas Comercial integra em ambiente de staging antes do lançamento em produção, dado o risco de churn [09:00, Marcos]
- **Monitoramento pós-lançamento:** acompanhar volume de polling em `GET /orders` pelos 3 clientes para confirmar a métrica de redução ≥ 70%
