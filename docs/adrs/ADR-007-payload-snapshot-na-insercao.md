# ADR-007: Payload Renderizado como Snapshot na Inserção da Outbox

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
**Revisores:** Sofia (Eng. Segurança), Marcos (PM)

## Contexto

Ao inserir um evento na `webhook_outbox`, havia duas opções: guardar apenas o `order_id` e montar o payload JSON completo no momento do envio (lazy rendering), ou guardar o payload já serializado no momento da inserção (snapshot). A decisão impacta a consistência do dado entregue ao cliente em casos onde o pedido sofre alterações após a mudança de status [09:51–09:52, Bruno, Larissa, Diego].

## Decisão

O payload JSON completo é serializado e armazenado na `webhook_outbox` no momento da inserção, dentro da mesma transação do `changeStatus`. O campo `payload` na tabela guarda o snapshot do estado do pedido no instante exato da transição de status — incluindo `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`.

## Alternativas Consideradas

### Alternativa 1: Lazy rendering — guardar apenas `order_id` e montar payload no envio
- **Descrição:** Armazenar somente o identificador do pedido na outbox e consultar o banco no momento em que o worker for enviar o webhook.
- **Por que foi descartada:** Se o pedido sofrer outra mudança de status entre a inserção na outbox e o envio (o que pode ocorrer em cenários de retry com backoff de horas), o payload refletiria o estado atual e não o estado no momento do evento. Isso geraria situações confusas onde o cliente recebe uma notificação de transição PAID→PROCESSING com dados que já refletem o estado SHIPPED [09:51–09:52, Larissa, Diego].

## Consequências

**Positivas:**
- O payload entregue ao cliente sempre reflete fielmente o estado do pedido no instante exato da transição, independentemente de quando o worker processar o evento.
- Simplifica o worker: não precisa consultar o banco para montar o payload, apenas lê e envia o que está na outbox.
- Em caso de retentativas (backoff de horas), o payload permanece consistente.

**Negativas / Trade-offs:**
- Aumenta o tamanho de cada linha na `webhook_outbox` em comparação com guardar apenas o `order_id`.
- Se o schema do payload precisar ser alterado futuramente, eventos antigos na outbox e na DLQ terão formato diferente dos novos.

## Referências

- TRANSCRICAO.md [09:51–09:52] Bruno, Larissa, Diego
- TRANSCRICAO.md [09:43] Diego — definição dos campos do payload: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`
- `src/modules/orders/order.service.ts` — método `changeStatus` (linha 126), contexto da transação onde o snapshot será gerado
