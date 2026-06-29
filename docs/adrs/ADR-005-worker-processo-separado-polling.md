# ADR-005: Worker em Processo Separado com Polling a Cada 2 Segundos

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

O cliente tem requisito de latência abaixo de 10 segundos [09:02, Marcos]. Era necessário definir como o worker leria a `webhook_outbox` e onde rodaria. Havia a questão de acoplamento com o processo da API: se o worker rodasse no mesmo processo, um restart da API interromperia o processamento. O MySQL não oferece mecanismo nativo de notificação para processos externos (ao contrário do PostgreSQL com NOTIFY/LISTEN) [09:09, Diego].

## Decisão

O worker roda como processo Node.js separado com entry-point em `src/worker.ts` e script `npm run worker`, independente do processo da API (`src/server.ts`). Utiliza polling ativo a cada 2 segundos: busca eventos pendentes mais antigos em batch, processa e marca como entregues. A lógica de processamento fica em `src/modules/webhooks/webhook.processor.ts`. O worker instancia seu próprio `PrismaClient` apontando para o mesmo `DATABASE_URL`.

## Alternativas Consideradas

### Alternativa 1: Worker dentro da mesma instância da API
- **Descrição:** Iniciar o worker como uma rotina interna no mesmo processo do servidor HTTP.
- **Por que foi descartada:** Um restart da API (deploys, crashes) interromperia o processamento de webhooks. Processos separados têm ciclos de vida independentes [09:11, Diego].

### Alternativa 2: Trigger de banco notificando o worker
- **Descrição:** Usar trigger no MySQL para notificar o worker reativamente quando um novo evento é inserido na outbox.
- **Por que foi descartada:** MySQL não possui mecanismo nativo NOTIFY/LISTEN como o PostgreSQL. Qualquer implementação de notificação via trigger exigiria gambiarras (escrever em arquivo, bater em endpoint), tornando a solução frágil [09:09, Diego].

### Alternativa 3: Polling com intervalo maior (ex.: 10 segundos)
- **Descrição:** Reduzir a frequência de polling para diminuir carga no banco.
- **Por que foi descartada:** Com 10 segundos de intervalo, no pior caso a latência seria de 10 segundos, exatamente no limite do requisito. Polling de 2 segundos garante latência máxima de ~2 segundos com margem confortável [09:09–09:10, Diego, Marcos].

## Consequências

**Positivas:**
- Ciclo de vida independente da API: restart ou falha da API não interrompe a entrega de webhooks.
- Polling de 2 segundos atende ao requisito de latência abaixo de 10 segundos com margem de 5x.
- Mesma stack tecnológica (Node.js, Prisma, TypeScript) sem nova dependência de runtime.

**Negativas / Trade-offs:**
- Polling gera carga constante no banco a cada 2 segundos, mesmo quando não há eventos pendentes — índices em `status` e `created_at` são essenciais para manter essa query barata [09:08, Diego].
- Com único worker, a ordem de entrega por `order_id` é garantida; escalar para múltiplos workers no futuro exigiria particionamento por `order_id` ou lock pessimista [09:12–09:13, Diego]. Documentado como limitação conhecida.
- Requer orquestração separada em produção (ex.: PM2, systemd, container separado).

## Referências

- TRANSCRICAO.md [09:08–09:11] Diego, Bruno, Larissa
- TRANSCRICAO.md [09:12–09:13] Diego, Larissa
- `src/modules/orders/order.service.ts` — `src/server.ts` mencionado como referência de entry-point existente [09:11, Larissa]
