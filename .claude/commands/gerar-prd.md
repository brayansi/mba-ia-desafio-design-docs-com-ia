---
description: Gera o PRD (Product Requirements Document) do Sistema de Webhooks consolidando decisões dos ADRs, RFC e FDD já produzidos
---

Você vai gerar o arquivo `docs/PRD.md` para o Sistema de Webhooks de Notificação de Pedidos.

## Antes de gerar

Leia obrigatoriamente (nesta ordem):
- `TRANSCRICAO.md` — transcrição completa da reunião técnica
- `docs/adrs/` — todos os ADRs gerados
- `docs/RFC.md` — proposta técnica já aprovada
- `docs/FDD.md` — especificação de implementação

O PRD é o **último grande documento** a ser produzido. Com os outros em mãos, ele é basicamente uma consolidação em linguagem de produto/negócio. Não repita detalhes técnicos de implementação — isso já está no FDD. Não repita decisões arquiteturais — isso está nos ADRs.

## O que é o PRD neste contexto

O PRD opera em nível de **produto e negócio**. Responde: "Por que construir isso e o quê exatamente?"
- Fala para PM, stakeholders e para o time de engenharia como contexto de negócio
- Não desce ao nível de DDL, fluxos de retry ou código

## Estrutura obrigatória

### 1. Resumo e Contexto

[1-2 parágrafos. O que é essa feature e por que existe. Use os dados concretos da transcrição: 3 clientes B2B, polling atual, risco de churn da Atlas.]

### 2. Problema e Motivação

[Por que o modelo atual (polling GET /orders) é inadequado. Impacto para o cliente. Urgência. Use dados de [09:00] Marcos.]

### 3. Público-alvo e Cenários de Uso

Público-alvo: clientes B2B integrados via API (Atlas Comercial, MaxDistribuição, Nova Cargo foram mencionados).

Cenários principais:
- Cliente quer receber notificação quando pedido vira SHIPPED para acionar logística própria
- Cliente quer saber quando pedido é CANCELLED para reverter cobrança interna
- Cliente quer monitorar DELIVERED para confirmar recebimento no sistema deles

### 4. Objetivos e Métricas de Sucesso

**Inclua pelo menos 1 objetivo com métrica quantitativa.**

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Reduzir latência de notificação | Tempo entre mudança de status e entrega do webhook | < 10 segundos (p95) |
| Adotar 3 clientes iniciais | Número de clientes com pelo menos 1 webhook ativo | 3 clientes até fim do trimestre |
| Alta confiabilidade de entrega | Taxa de entrega bem-sucedida (excluindo endpoints permanentemente offline) | ≥ 99% |
| Reduzir carga de polling | Volume de chamadas ao GET /orders pelos 3 clientes B2B | Redução de ≥ 70% |

### 5. Escopo

#### Incluso nesta fase
- Cadastro, edição, listagem e remoção de endpoints de webhook por customer
- Entrega automática de eventos de mudança de status de pedidos
- Filtro de eventos por status (o cliente escolhe quais statuses quer ouvir)
- Autenticação HMAC-SHA256 com secret por endpoint
- Rotação de secret com grace period de 24 horas
- Retry automático com backoff exponencial (5 tentativas: 1m/5m/30m/2h/12h)
- Dead Letter Queue (DLQ) com histórico de falhas
- Replay manual de itens da DLQ por administradores
- Histórico de deliveries por endpoint (`GET /webhooks/:id/deliveries`)

#### Fora de escopo (explicitamente descartado ou adiado)
- **Webhook inbound** (clientes enviando eventos para o sistema) — descartado em [09:02] Sofia e Marcos: "Só saindo da gente pra eles"
- **Notificação por e-mail** em caso de falha no webhook — adiado para próxima fase em [09:37] Larissa: "Email tá fora de escopo dessa fase. Talvez próxima fase"
- **Dashboard visual** para o cliente acompanhar webhooks — fora de escopo em [09:40]: "Só endpoints. Painel é projeto separado do time de frontend"
- **Rate limiting de saída** por cliente — adiado para observar em produção, [09:39] Larissa: "Fica como 'observar e decidir depois'"
- **Garantia exactly-once** — descartada em [09:25] Diego, at-least-once com deduplicação pelo cliente é o padrão adotado
- **Escalonamento com múltiplos workers** — adiado como problema futuro em [09:13] Diego

### 6. Requisitos Funcionais

Liste pelo menos 8 requisitos funcionais com ID rastreável:

| ID | Requisito |
|----|-----------|
| RF-01 | O sistema deve permitir cadastrar um endpoint de webhook com URL, lista de eventos e vínculo a um customer |
| RF-02 | A URL do webhook deve ser obrigatoriamente HTTPS; URLs com `http://` devem ser rejeitadas |
| RF-03 | A secret do webhook deve ser gerada pelo sistema na criação e retornada apenas uma vez |
| RF-04 | O cliente deve poder rotacionar a secret; a secret anterior permanece válida por 24 horas após a rotação |
| RF-05 | O sistema deve disparar o evento automaticamente quando o status de um pedido muda, dentro dos eventos configurados para aquele webhook |
| RF-06 | O payload do evento deve incluir: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents |
| RF-07 | Cada envio deve incluir os headers: X-Event-Id, X-Webhook-Id, X-Signature, X-Timestamp, Content-Type |
| RF-08 | O sistema deve retentar a entrega em caso de falha, com backoff exponencial de 5 tentativas (1m/5m/30m/2h/12h) |
| RF-09 | Após 5 falhas consecutivas, o evento deve ser movido para a Dead Letter Queue |
| RF-10 | Administradores devem poder reprocessar manualmente um item da DLQ |
| RF-11 | O cliente deve poder consultar o histórico de deliveries de um webhook, com status, payload, resposta e tempo de resposta |
| RF-12 | O sistema deve permitir editar e desativar um endpoint de webhook |

### 7. Requisitos Não Funcionais

| ID | Requisito |
|----|-----------|
| RNF-01 | Latência máxima entre mudança de status e entrega do webhook: 10 segundos (p95) |
| RNF-02 | Timeout de 10 segundos por chamada HTTP do worker |
| RNF-03 | Tamanho máximo de payload: 64KB |
| RNF-04 | Garantia de entrega: at-least-once (o cliente deve deduplicar por X-Event-Id) |
| RNF-05 | Autenticação do payload: HMAC-SHA256 com secret por endpoint |
| RNF-06 | O módulo não deve introduzir nova dependência de infraestrutura (sem Redis, sem filas externas) |
| RNF-07 | O worker deve rodar como processo separado da API |
| RNF-08 | Logs estruturados via Pino em todos os passos do worker |

### 8. Decisões e Trade-offs Principais

[Não repita os ADRs completos — referencie-os. Liste apenas o resumo de decisão e impacto no produto.]

| Decisão | Impacto |
|---------|---------|
| Padrão Outbox no MySQL (ver [ADR-001](adrs/ADR-001-....md)) | Garante consistência sem nova infra; pequeno delay de polling |
| At-least-once com X-Event-Id | Simplifica o sistema; transfere responsabilidade de deduplicação para o cliente |
| Secret por endpoint (ver [ADR-003](adrs/ADR-003-....md)) | Segurança granular; compromisso de uma secret não afeta outros endpoints |

### 9. Dependências

- Time de Engenharia: Bruno (Pedidos) e Diego (Plataforma) para implementação
- Revisão de segurança: Sofia reserva pelo menos 2 dias úteis antes do deploy ([09:46])
- Portal do desenvolvedor: Marcos documenta o contrato de integração para os clientes ([09:26], [09:40])
- Clientes B2B: implementação do endpoint receptor e deduplicação por X-Event-Id

### 10. Riscos e Mitigação

Inclua pelo menos 2 riscos com probabilidade, impacto e mitigação:

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Clientes não implementarem deduplicação por X-Event-Id, causando ações duplicadas | Alta | Médio | Documentar at-least-once com destaque no portal do desenvolvedor; oferecer exemplos de deduplicação |
| Atlas migrar para concorrente se entrega atrasar além do trimestre | Média | Alto | Priorizar CRUD de configuração + entrega básica na primeira sprint; funcionalidades avançadas nas seguintes |
| Acúmulo de eventos na outbox comprometendo performance | Baixa | Médio | Monitorar tamanho da tabela; arquivamento após 30 dias planejado para fase posterior |
| Vazar secret de webhook nos logs do cliente | Baixa | Alto | Orientar no portal; never retornar secret em endpoints de leitura após a criação |

### 11. Critérios de Aceitação

- [ ] Os 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) conseguem cadastrar e receber webhooks
- [ ] Latência p95 abaixo de 10 segundos em condições normais de operação
- [ ] Clientes param de fazer polling no GET /orders para monitorar mudanças de status
- [ ] Evento de mudança de status não é perdido mesmo que o endpoint do cliente esteja offline no momento (retry funciona)
- [ ] Auditoria de quem fez replay de DLQ está registrada no log

### 12. Estratégia de Testes e Validação

- Testes de integração: fluxo completo de mudança de status → inserção na outbox → entrega pelo worker
- Testes de falha: endpoint do cliente retornando 500 → verificar retry e progressão do backoff
- Testes de segurança: Sofia valida HMAC, geração de secret e rotação
- Validação com clientes: Atlas Comercial como cliente piloto para validação em ambiente de staging

## Regras de qualidade

- Toda informação deve ter origem rastreável na transcrição (timestamp + nome) ou no código
- A seção "Fora de escopo" deve citar explicitamente pelo menos 2 itens descartados ou adiados com referência à transcrição
- A seção "Riscos" deve ter pelo menos 2 riscos com probabilidade, impacto e mitigação
- Pelo menos 1 objetivo deve ter métrica quantitativa
- Pelo menos 8 requisitos funcionais
- Não desça ao nível de detalhe do FDD (sem DDL, sem exemplos de payload, sem fluxos de retry detalhados)

## Saída esperada

Escreva o conteúdo diretamente em `docs/PRD.md`. Após gerar, confirme que a seção "Fora de escopo" lista pelo menos 2 itens com referência à transcrição e que há pelo menos 1 objetivo com métrica quantitativa.
