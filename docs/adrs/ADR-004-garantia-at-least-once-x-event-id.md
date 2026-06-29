# ADR-004: Garantia At-Least-Once com Deduplicação por X-Event-Id

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

O sistema de retry pode entregar o mesmo evento mais de uma vez: por exemplo, se o worker envia o webhook, o cliente recebe e processa, mas a resposta HTTP é perdida antes de o worker marcar o evento como entregue — a próxima tentativa reenvia o mesmo payload. Era preciso decidir qual garantia de entrega adotar e como permitir que o cliente lide com duplicatas [09:24–09:25, Diego].

## Decisão

Adotar garantia at-least-once: cada evento recebe um UUID gerado no momento da inserção na `webhook_outbox`, enviado no header `X-Event-Id` em todas as tentativas de entrega daquele evento. O cliente é responsável por deduplicar usando esse identificador. A plataforma não implementa coordenação de exactly-once.

## Alternativas Consideradas

### Alternativa 1: Garantia exactly-once
- **Descrição:** Implementar mecanismo de coordenação bidirecional para garantir que cada evento seja processado exatamente uma vez pelo cliente.
- **Por que foi descartada:** Exige coordenação dos dois lados (plataforma e cliente), com armazenamento de estado de quais eventos foram confirmados, aumentando significativamente a complexidade. Stripe e GitHub adotam at-least-once com deduplicação no cliente como padrão de mercado [09:25, Diego].

### Alternativa 2: At-least-once sem identificador de evento
- **Descrição:** Entregar at-least-once sem fornecer qualquer identificador único de evento ao cliente.
- **Por que foi descartada:** Sem `X-Event-Id`, o cliente não tem como detectar duplicatas de forma confiável. O identificador não adiciona custo relevante e é essencial para que o padrão at-least-once seja utilizável [09:25, Diego; 09:26, Marcos].

## Consequências

**Positivas:**
- Implementação significativamente mais simples do que exactly-once.
- O `X-Event-Id` (UUID gerado na inserção da outbox) é estável entre retentativas — o cliente pode usar como chave de idempotência.
- Padrão documentado e reconhecido; Marcos comprometeu-se a destacá-lo no portal de desenvolvedor [09:26, Marcos].

**Negativas / Trade-offs:**
- A responsabilidade de deduplicação é do cliente, o que precisa estar claramente documentado na integração.
- Em cenários de falha de rede entre o worker e o endpoint do cliente, há risco real de entrega duplicada que o cliente deve estar preparado para tratar.

## Referências

- TRANSCRICAO.md [09:24–09:26] Diego, Sofia, Marcos, Larissa
- TRANSCRICAO.md [09:51] Diego, Larissa — confirmação de UUID como identificador da outbox, seguindo padrão `@id @default(uuid())` já presente em todos os modelos do `prisma/schema.prisma`
