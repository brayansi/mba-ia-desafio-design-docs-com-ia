---
description: Gera o RFC (Request for Comments) da proposta técnica do Sistema de Webhooks — documento conciso de 2 a 4 páginas para revisão da equipe
---

Você vai gerar o arquivo `docs/RFC.md` para o Sistema de Webhooks de Notificação de Pedidos.

## Antes de gerar

Leia obrigatoriamente:
- `TRANSCRICAO.md` — transcrição completa da reunião técnica
- `docs/adrs/` — todos os ADRs já gerados (referencie-os com links relativos)
- `prisma/schema.prisma` — estrutura de dados existente
- `src/modules/orders/order.service.ts` — ponto de integração principal

## O que é o RFC neste contexto

O RFC é a **proposta técnica submetida à equipe para revisão**. Opera em nível de arquitetura — apresenta a abordagem escolhida, as alternativas descartadas e as questões em aberto. É **conciso (2 a 4 páginas)**. O detalhamento de implementação fica no FDD, não aqui.

**Fronteira clara:**
- RFC responde: "O que propomos e por quê?"
- FDD responde: "Como construir, em detalhe?"
- ADRs registram: "Por que decidimos exatamente assim em cada ponto?"

## Estrutura obrigatória

```markdown
# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo      | Valor                                      |
|------------|--------------------------------------------|
| **Autor**  | Larissa (Tech Lead)                        |
| **Status** | Em revisão                                 |
| **Data**   | 2026-06-29                                 |
| **Revisores** | Diego (Eng. Sênior), Bruno (Eng. Pleno), Sofia (Eng. Segurança), Marcos (PM) |

---

## TL;DR

[2-3 frases resumindo a proposta inteira. Quem lê só isso deve entender o que está sendo proposto.]

## Contexto e Problema

[Por que precisamos disso? Qual o impacto de não fazer? Use dados da transcrição — clientes B2B, polling atual, risco de churn.]

## Proposta Técnica

[Visão geral da solução — padrão outbox, worker separado, HMAC, at-least-once. NÃO desça ao nível de detalhe do FDD: sem DDL de tabelas, sem exemplos de payload completos, sem matriz de erros. Apenas a abordagem arquitetural.]

## Alternativas Consideradas

### Alternativa 1: [Nome — deve ser uma alternativa real discutida na reunião]
- **Descrição:** ...
- **Trade-off que levou ao descarte:** ...

### Alternativa 2: [Nome — deve ser uma alternativa real discutida na reunião]
- **Descrição:** ...
- **Trade-off que levou ao descarte:** ...

## Questões em Aberto

[Pelo menos 2 pontos levantados na reunião e não decididos ou adiados. Seja específico: cite o que foi dito e por quem.]

| # | Questão | Decisão tomada | Observação |
|---|---------|---------------|------------|
| 1 | ...     | Adiado        | ...        |
| 2 | ...     | Adiado        | ...        |

## Impacto e Riscos

[Principais riscos arquiteturais e de integração. Não repita o que já está nos ADRs — apenas o que tem impacto no nível da proposta como um todo.]

## Decisões Relacionadas

[Links para os ADRs correspondentes. Pelo menos 2 links.]

- [ADR-001: ...](adrs/ADR-001-....md)
- [ADR-002: ...](adrs/ADR-002-....md)
- ...
```

## Alternativas obrigatórias (devem ser reais da reunião)

As duas alternativas mínimas exigidas pelos critérios de aceite são:

1. **Webhook síncrono dentro do OrderService** — descartado em [09:04]/[09:06] porque travaria a transação principal se o cliente estivesse lento ou offline
2. **Redis Streams (ou fila externa)** — descartado em [09:07] por exigir infra adicional e ser overengineering para um time pequeno com MySQL já disponível

## Questões em aberto obrigatórias (devem ser reais da reunião)

1. **Rate limiting de envio para clientes** — levantado por Diego em [09:38], adiado como "observar e decidir depois" ([09:39])
2. **Notificação por e-mail em caso de falha** — levantado por Marcos em [09:37], explicitamente adiado para próxima fase ([09:37])

Outros pontos adiados que podem complementar: dashboard visual ([09:40]), escalonamento de múltiplos workers com garantia de ordering global ([09:13]).

## Regras de qualidade

- Máximo 4 páginas — se estiver ultrapassando, mova detalhes para o FDD
- Não duplicate tabelas de erros, DDL de banco, exemplos de payload ou fluxos detalhados — isso fica no FDD
- Toda afirmação deve ter origem na transcrição ou no código
- Os links para ADRs devem apontar para arquivos que realmente existem em `docs/adrs/`

## Saída esperada

Escreva o conteúdo diretamente em `docs/RFC.md`. Após gerar, confirme que o arquivo contém as seções obrigatórias e que os links para ADRs são válidos.
