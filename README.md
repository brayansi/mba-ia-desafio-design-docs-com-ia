# Da Reunião ao Documento: Processo de Produção dos Design Docs com IA

## Sobre o Desafio

O ponto de partida foi uma transcrição de reunião técnica (`TRANSCRICAO.md`) — 55 minutos de discussão entre tech lead, PM, engenheiros e segurança sobre a construção de um Sistema de Webhooks de Notificação de Pedidos para um Order Management System (OMS) em produção. A tarefa era transformar essa transcrição mais o código-fonte existente em um pacote completo de design docs: PRD, RFC, FDD, ADRs e Tracker — documentação acionável o suficiente para o time de engenharia iniciar a implementação sem depender de ninguém que esteve na reunião.

A restrição principal era rastreabilidade: nada inventado, tudo com origem identificável na transcrição (timestamp + falante) ou no código (caminho de arquivo). Identificar o que **não** entra na documentação foi tão importante quanto identificar o que entra — a reunião descartou explicitamente várias ideias (Redis Streams, webhook inbound, exactly-once, email de alerta), e deixá-las escapar como requisito seria um sinal claro de falha na filtragem.

---

## Ferramentas de IA Utilizadas

- **Claude Code (claude-sonnet-4-6)** — ferramenta principal de análise e geração dos documentos
  - Leitura e análise da transcrição e do código-fonte (`src/`, `prisma/`, `tests/`)
  - Geração iterativa dos documentos via skills customizadas
  - Verificação cruzada de rastreabilidade entre documentos e fontes

- **Skills customizadas (`.claude/commands/`)** — prompts de alta precisão criados antes de qualquer geração de documento
  - `/gerar-adrs` — geração dos ADRs com template MADR, lista explícita do que NÃO deve virar decisão e filtros de rastreabilidade
  - `/gerar-rfc` — geração do RFC com instrução explícita de concisão (2–4 páginas) e fronteira clara com o FDD
  - `/gerar-fdd` — geração do FDD com seção obrigatória de integração com o código existente, citando caminhos reais
  - `/gerar-prd` — geração do PRD como consolidação dos outros documentos, em linguagem de produto/negócio
  - `/gerar-tracker` — geração do Tracker com verificação cruzada de fontes e caminhos de arquivo reais
  - `/gerar-readme` — geração deste arquivo de processo

---

## Workflow Adotado

A ordem de produção seguiu a lógica de dependência entre os documentos: as decisões formam o esqueleto, e os documentos mais amplos são construídos em cima das decisões já fechadas.

1. **Exploração inicial** — antes de criar qualquer documento, pedi ao Claude Code que lesse a transcrição e o código para mapear a estrutura existente: padrão de módulos em `src/modules/`, máquina de estados em `order.service.ts`, classes de erro em `src/shared/errors/`, middleware de autenticação. Isso evitou que os documentos gerados ficassem desconectados do que já existe no repositório.

2. **Criação das skills** — antes de gerar qualquer documento, construí os prompts como commands reutilizáveis em `.claude/commands/`. Cada skill foi escrita com: lista de arquivos que devem ser lidos antes de gerar, estrutura obrigatória do documento, exemplos concretos de conteúdo esperado, lista do que **não** deve aparecer, e regras de qualidade. Essa etapa foi crucial — prompts vagos produzem documentos genéricos.

3. **ADRs** — gerados primeiro com `/gerar-adrs`. As decisões formam o esqueleto de toda a documentação técnica. Sete ADRs ao total, cobrindo as seis decisões principais da reunião mais a decisão de payload como snapshot.

4. **RFC** — gerado com `/gerar-rfc` referenciando os ADRs já escritos. O RFC opera em nível de arquitetura — proposta técnica concisa, alternativas descartadas e questões em aberto. Não desce ao detalhe de implementação.

5. **FDD** — gerado com `/gerar-fdd`, o documento mais técnico. Fluxos detalhados, contratos HTTP completos, matriz de erros, estratégias de resiliência e a seção obrigatória de integração com o código existente.

6. **PRD** — gerado por último entre os grandes documentos com `/gerar-prd`. Com os ADRs, RFC e FDD em mãos, o PRD virou basicamente uma consolidação em linguagem de produto, sem repetir detalhes técnicos.

7. **Tracker** — gerado com `/gerar-tracker` varrendo todos os documentos prontos. 71 linhas mapeando cada item à transcrição ou ao código.

8. **README** — este arquivo, gerado por último com `/gerar-readme`, quando o processo já estava completo.

---

## Prompts Customizados

**Trecho da skill `/gerar-adrs` — lista do que NÃO deve virar ADR:**

```
## O que NÃO deve aparecer como decisão (foi descartado ou adiado)

- Redis Streams / filas externas — descartado em [09:07] por ser overengineering para time pequeno
- Webhook síncrono dentro do service de orders — descartado em [09:04]/[09:06]
- Trigger de banco para notificar worker — descartado em [09:09] (MySQL não tem NOTIFY/LISTEN)
- Email de alerta para cliente — explicitamente adiado para próxima fase em [09:37]
- Rate limiting de saída — adiado como "observar e decidir depois" em [09:39]
- Dashboard visual — fora de escopo em [09:40]
- Garantia exactly-once — descartada em [09:25], at-least-once é suficiente
```

**Trecho da skill `/gerar-fdd` — seção obrigatória de integração com o código:**

```
### 10. Integração com o Sistema Existente

**Esta seção é obrigatória e específica deste desafio.**

Descreva como o módulo de webhooks se integra com cada um dos arquivos abaixo (cite o caminho exato):

1. `src/modules/orders/order.service.ts` — o método `changeStatus` será estendido para chamar
   `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação Prisma (linhas 131–178).
   A função recebe o `tx` client, garantindo atomicidade com o commit de mudança de status.

2. `src/shared/errors/app-error.ts` — as classes de erro do módulo de webhooks (WebhookNotFoundError,
   WebhookInvalidUrlError, etc.) vão estender AppError seguindo o padrão já estabelecido no projeto.

3. `src/middlewares/error.middleware.ts` — o middleware centralizado já captura instâncias de AppError
   e formata a resposta. Os erros WEBHOOK_* serão tratados automaticamente sem alteração no middleware.

4. `src/middlewares/auth.middleware.ts` — os endpoints de webhook usarão o mesmo middleware de
   autenticação JWT existente. O endpoint de replay de DLQ usará adicionalmente o requireRole('ADMIN').
```

**Trecho da skill `/gerar-rfc` — instrução de fronteira com o FDD:**

```
## O que é o RFC neste contexto

O RFC é a **proposta técnica submetida à equipe para revisão**. Opera em nível de arquitetura —
apresenta a abordagem escolhida, as alternativas descartadas e as questões em aberto. É **conciso
(2 a 4 páginas)**. O detalhamento de implementação fica no FDD, não aqui.

**Fronteira clara:**
- RFC responde: "O que propomos e por quê?"
- FDD responde: "Como construir, em detalhe?"
- ADRs registram: "Por que decidimos exatamente assim em cada ponto?"

## Regras de qualidade

- Máximo 4 páginas — se estiver ultrapassando, mova detalhes para o FDD
- Não duplicate tabelas de erros, DDL de banco, exemplos de payload ou fluxos detalhados — isso fica no FDD
```

---

## Iterações e Ajustes

**1. RFC com profundidade do FDD na primeira versão**

A primeira versão do RFC incluía DDL completo das tabelas (`webhook_outbox`, `webhook_dead_letter`), exemplos detalhados de payload e a matriz de erros completa com todos os códigos `WEBHOOK_*`. O RFC estava com cinco páginas densas — o dobro do permitido. A solução foi ajustar a skill com a seção de "Fronteira clara" e a regra explícita "Máximo 4 páginas — se estiver ultrapassando, mova detalhes para o FDD". Com essa instrução, o RFC gerado ficou conciso: proposta técnica em visão geral, sem descer ao nível de implementação que pertence ao FDD.

**2. ADRs incluindo itens descartados como decisões**

Na primeira versão dos ADRs, a decisão sobre "email de alerta para clientes em caso de falha" apareceu documentada como se fosse uma funcionalidade a ser implementada. Ela havia sido explicitamente adiada na reunião em [09:37], mas o modelo capturou a menção e a converteu em ADR. A solução foi incluir na skill uma lista explícita do que **não** deve aparecer como decisão, com o timestamp de quando foi descartado. O resultado foi que os ADRs gerados passaram a registrar apenas as decisões fechadas — as alternativas descartadas apareceram na seção "Alternativas Consideradas" de cada ADR, que é exatamente onde pertencem.

**3. FDD sem integração com o código existente**

A primeira versão do FDD descrevia os fluxos e contratos de forma genérica, sem referenciar arquivos reais do repositório. As seções técnicas estavam corretas em abstrato, mas um desenvolvedor que pegasse o documento não saberia onde no código existente fazer cada integração. Adicionei à skill a seção obrigatória "Integração com o Sistema Existente" com instrução explícita de citar caminhos de arquivo reais e descrever como cada um seria estendido ou reutilizado. O FDD final cita seis arquivos reais: `order.service.ts`, `app-error.ts`, `error.middleware.ts`, `auth.middleware.ts`, `schema.prisma` e `database.ts`.

**4. Tracker sem verificação cruzada de caminhos**

O Tracker inicial não verificava se os caminhos de arquivo nas linhas com `Fonte = CODIGO` existiam de fato no repositório. Algumas linhas apontavam para caminhos plausíveis mas inexistentes. Adicionei na skill a instrução de verificar cada caminho antes de incluir na tabela. O Tracker final tem sete linhas com `Fonte = CODIGO`, todas apontando para arquivos reais verificados: `src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `prisma/schema.prisma`, `src/config/database.ts` e `src/shared/errors/http-errors.ts`.

---

## Como Navegar a Entrega

```
.
├── README.md                    ← você está aqui (processo de produção)
├── TRANSCRICAO.md               ← fonte primária — reunião técnica
├── docs/
│   ├── PRD.md                   ← (1) leia primeiro: contexto de produto e negócio
│   ├── RFC.md                   ← (2) proposta técnica para revisão
│   ├── FDD.md                   ← (3) especificação de implementação detalhada
│   ├── TRACKER.md               ← (4) rastreabilidade de cada item
│   └── adrs/
│       ├── ADR-001-padrao-outbox-mysql.md
│       ├── ADR-002-retry-backoff-exponencial-dlq.md
│       ├── ADR-003-autenticacao-hmac-sha256.md
│       ├── ADR-004-garantia-at-least-once-x-event-id.md
│       ├── ADR-005-worker-processo-separado-polling.md
│       ├── ADR-006-reuso-padroes-existentes-projeto.md
│       └── ADR-007-payload-snapshot-na-insercao.md
└── .claude/commands/            ← skills usadas para gerar os documentos
    ├── gerar-adrs.md
    ├── gerar-rfc.md
    ├── gerar-fdd.md
    ├── gerar-prd.md
    ├── gerar-tracker.md
    └── gerar-readme.md
```

**Ordem sugerida de leitura:** PRD → RFC → ADRs → FDD → TRACKER
