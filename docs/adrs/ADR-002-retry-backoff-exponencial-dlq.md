# ADR-002: Política de Retry com Backoff Exponencial e Dead Letter Queue

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

O worker de webhooks faz chamadas HTTP para endpoints externos que podem estar temporariamente indisponíveis. Era preciso definir quantas vezes tentar novamente, com qual progressão de intervalo, e o que fazer quando todas as tentativas se esgotam. Havia pressão para ter uma janela de retry suficientemente longa para cobrir manutenções planejadas dos clientes (que historicamente chegaram a durar 2 horas) [09:16, Diego], sem deixar eventos pendentes indefinidamente.

## Decisão

Adotar 5 tentativas com backoff exponencial de intervalos fixos: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — totalizando aproximadamente 15 horas entre a primeira falha e a última tentativa. Após o esgotamento das tentativas, mover o evento para uma tabela `webhook_dead_letter` separada, contendo payload, motivo da falha e timestamp. Reprocessamento manual via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito à role `ADMIN`.

## Alternativas Consideradas

### Alternativa 1: 3 tentativas com backoff mais agressivo
- **Descrição:** Realizar apenas 3 retentativas em intervalos menores, encerrando o ciclo em menos de 1 hora.
- **Por que foi descartada:** Insuficiente para cobrir indisponibilidades planejadas. Houve caso real de cliente indisponível por 2 horas em manutenção; com 3 tentativas, o evento seria descartado antes de o cliente voltar [09:16, Diego].

### Alternativa 2: Retry indefinido com backoff
- **Descrição:** Continuar tentando sem limite de tentativas enquanto o evento não for entregue.
- **Por que foi descartada:** Eventos ficam pendurados indefinidamente se o cliente sumir permanentemente, crescendo a tabela `webhook_outbox` sem controle. O limite de 5 tentativas equilibra resiliência com previsibilidade [09:15, Diego].

### Alternativa 3: DLQ marcada na própria tabela `webhook_outbox`
- **Descrição:** Em vez de tabela separada, marcar o evento como `failed` na mesma `webhook_outbox`.
- **Por que foi descartada:** Polui a leitura dos eventos pendentes na outbox principal e mistura dados de debug (payload de falha, mensagem de erro, histórico de tentativas) com a fila operacional. Tabela separada mantém a outbox limpa e a DLQ como evidência auditável [09:18, Diego].

## Consequências

**Positivas:**
- Cobre janelas de indisponibilidade de até ~15 horas sem intervenção manual.
- A DLQ em tabela separada permite debug e reprocessamento seletivo.
- Endpoint de replay com role `ADMIN` e log de auditoria de quem executou o replay [09:36, Sofia].

**Negativas / Trade-offs:**
- Eventos com falha permanente só são reprocessados manualmente — não há reprocessamento automático da DLQ.
- A janela de 15 horas significa que um evento pode permanecer na outbox por esse período antes de ser descartado, exigindo monitoramento ativo da tabela.

## Referências

- TRANSCRICAO.md [09:14–09:18] Larissa, Diego, Bruno
- TRANSCRICAO.md [09:35–09:36] Diego, Sofia, Larissa
