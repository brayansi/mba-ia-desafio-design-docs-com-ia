# ADR-001: Padrão Outbox no MySQL para Publicação de Eventos de Webhook

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

Quando o status de um pedido muda, três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados com latência abaixo de 10 segundos [09:02, Marcos]. A operação de mudança de status em `src/modules/orders/order.service.ts` (método `changeStatus`) já executa dentro de uma transação SQL que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos [09:04, Bruno]. Precisava-se de uma forma de publicar o evento de notificação sem comprometer a atomicidade nem adicionar dependência síncrona de rede dentro dessa transação.

## Decisão

Utilizar o padrão Transactional Outbox: dentro da mesma transação SQL do `changeStatus`, inserir uma linha na tabela `webhook_outbox` contendo o evento serializado. Um worker separado lê essa tabela de forma assíncrona e dispara as chamadas HTTP para os endpoints dos clientes.

## Alternativas Consideradas

### Alternativa 1: Chamada HTTP síncrona dentro do `changeStatus`
- **Descrição:** Realizar o POST para o endpoint do cliente diretamente dentro da transação, antes do commit.
- **Por que foi descartada:** A transação de mudança de status já é pesada. Um cliente HTTP lento travaria mudanças de status de outros pedidos. Se o endpoint do cliente estiver fora do ar, seria necessário dar rollback na mudança de status — comportamento inaceitável [09:04, Bruno; 09:04, Larissa].

### Alternativa 2: Redis Streams como fila de eventos
- **Descrição:** Publicar o evento em um Redis Stream após o commit da transação principal, e um worker consumiria esse stream.
- **Por que foi descartada:** Exige subir e operar infraestrutura adicional (Redis Cluster). Para um time pequeno, seria overengineering desnecessário; o MySQL existente resolve o problema sem nova infra [09:07, Diego].

## Consequências

**Positivas:**
- Atomicidade garantida: se a transação principal fizer rollback (ex.: estoque insuficiente), o evento some junto — não há inconsistência possível entre estado da ordem e eventos publicados.
- Sem nova infraestrutura: reutiliza o MySQL existente.
- A chamada HTTP do worker é desacoplada da latência do cliente externo, sem impacto na API.

**Negativas / Trade-offs:**
- Adiciona carga de escrita na tabela `webhook_outbox` a cada mudança de status.
- Requer índices em `status` e `created_at` na `webhook_outbox` para que o worker faça leitura eficiente [09:08, Diego].
- Linhas entregues precisam ser arquivadas periodicamente (escopo fora desta feature) para evitar crescimento indefinido da tabela.

## Referências

- TRANSCRICAO.md [09:03–09:08] Larissa, Diego, Bruno
- `src/modules/orders/order.service.ts` — método `changeStatus` (linha 126), onde a inserção na `webhook_outbox` ocorrerá dentro do `prisma.$transaction`
- `prisma/schema.prisma` — modelos `Order`, `OrderStatusHistory` como referência da transação existente
