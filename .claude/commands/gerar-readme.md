---
description: Substitui o README.md do repositório pela documentação do processo de produção dos design docs com IA
---

Você vai substituir o conteúdo do `README.md` na raiz do repositório pela documentação do processo de produção.

## Antes de gerar

Leia:
- `README.md` atual — para entender o que existe antes de substituir
- `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/`, `docs/TRACKER.md` — para ter a visão completa do que foi produzido
- `.claude/commands/` — as skills criadas fazem parte do processo e devem ser mencionadas

## O que este README deve conter

O README não é mais o enunciado do desafio. Ele documenta **como o trabalho foi feito** — a jornada de produção dos design docs usando IA como ferramenta principal.

O leitor-alvo é alguém que vai avaliar: como você usou a IA, quais decisões tomou, onde teve que corrigir, o que aprendeu.

## Estrutura obrigatória

### Sobre o Desafio
1-2 parágrafos descrevendo a tarefa em suas palavras. O que era o ponto de partida (TRANSCRICAO.md + codebase), o que foi produzido (pacote de design docs) e qual era a restrição principal (rastreabilidade — nada inventado, tudo com origem identificável).

### Ferramentas de IA Utilizadas
Lista das ferramentas com nota sobre o papel de cada uma:

```
- **Claude Code (claude-sonnet-4-6)** — ferramenta principal de análise e geração dos documentos
  - Leitura e análise da transcrição e do código-fonte
  - Geração iterativa dos documentos via skills customizadas
- **Skills customizadas (.claude/commands/)** — prompts de alta precisão criados para cada documento
  - `/gerar-adrs` — geração dos ADRs com template MADR e filtros de rastreabilidade
  - `/gerar-rfc` — geração do RFC com instrução explícita de concisão (2-4 páginas)
  - `/gerar-fdd` — geração do FDD com seção obrigatória de integração com o código
  - `/gerar-prd` — geração do PRD como consolidação dos outros documentos
  - `/gerar-tracker` — geração do Tracker com verificação cruzada de fontes
  - `/gerar-readme` — geração deste arquivo de processo
```

### Workflow Adotado
Descreva em que ordem os documentos foram produzidos e como foi a interação com a IA.

Ordem sugerida (conforme orientação do desafio):
1. Exploração inicial — leitura do código e da transcrição para entender o contexto
2. Criação das skills — antes de gerar qualquer documento, construir os prompts como commands reutilizáveis
3. ADRs — as decisões formam o esqueleto; gerados primeiro com `/gerar-adrs`
4. RFC — consolida a proposta; gerado com `/gerar-rfc` referenciando os ADRs
5. FDD — detalha a implementação; gerado com `/gerar-fdd` com a seção de integração
6. PRD — consolidação em linguagem de produto; gerado com `/gerar-prd` por último
7. Tracker — gerado com `/gerar-tracker` varrendo todos os documentos
8. README — este arquivo, produzido por último

### Prompts Customizados
Mostre pelo menos 2 prompts relevantes em blocos de código. Estes são exemplos do que ficou nas skills — mostre trechos reais que ilustram a intenção de cada prompt.

**Exemplo de prompt para os ADRs (trecho da skill `/gerar-adrs`):**
```
## O que NÃO deve aparecer como decisão (foi descartado ou adiado)

- Redis Streams / filas externas — descartado em [09:07] por ser overengineering para time pequeno
- Webhook síncrono dentro do service de orders — descartado em [09:04]/[09:06]
- Trigger de banco para notificar worker — descartado em [09:09] (MySQL não tem NOTIFY/LISTEN)
- Email de alerta para cliente — explicitamente adiado para próxima fase em [09:37]
- Rate limiting de saída — adiado como "observar e decidir depois" em [09:39]
```

**Exemplo de prompt de integração com o código (trecho da skill `/gerar-fdd`):**
```
## Integração com o Sistema Existente (seção obrigatória)

1. `src/modules/orders/order.service.ts` — o método `changeStatus` será estendido para chamar
   `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação Prisma
   (linhas 131–178). A função recebe o `tx` client, garantindo atomicidade.

2. `src/shared/errors/app-error.ts` — as classes de erro do módulo de webhooks vão estender
   `AppError` seguindo o padrão já estabelecido no projeto.
```

### Iterações e Ajustes
Descreva os principais momentos em que a IA gerou algo errado ou superficial e você teve que corrigir. Inclua pelo menos 2 exemplos concretos.

Exemplos de situações a descrever (adapte ao que realmente aconteceu):
- A primeira versão do RFC incluía nível de detalhe do FDD (DDL de tabelas, exemplos completos de payload) — foi necessário ajustar a skill para deixar explícito que o RFC tem no máximo 4 páginas e que o detalhamento fica no FDD
- O Tracker inicial não verificava se os caminhos de arquivo existiam no repositório — foram adicionadas instruções explícitas de verificação cruzada
- A primeira versão dos ADRs incluía a decisão sobre "email de alerta" como requisito, quando na verdade foi explicitamente descartada na reunião — a skill foi ajustada com lista de itens que NÃO devem aparecer

### Como Navegar a Entrega

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
│       ├── ADR-001-*.md         ← decisões arquiteturais individuais
│       ├── ADR-002-*.md
│       ├── ...
│       └── ADR-00N-*.md
└── .claude/commands/            ← skills usadas para gerar os documentos
    ├── gerar-adrs.md
    ├── gerar-rfc.md
    ├── gerar-fdd.md
    ├── gerar-prd.md
    ├── gerar-tracker.md
    └── gerar-readme.md
```

**Ordem sugerida de leitura:** PRD → RFC → ADRs → FDD → TRACKER

## Regras de qualidade

- Escreva na primeira pessoa — você está documentando sua própria jornada
- Seja específico nas iterações: "a IA gerou X errado; ajustei Y no prompt e o resultado ficou Z"
- As skills mencionadas devem existir em `.claude/commands/` — não invente arquivos
- Os prompts nos blocos de código devem ser trechos reais das skills criadas

## Saída esperada

Substitua o conteúdo de `README.md` na raiz. Após gerar, confirme que o arquivo contém todas as seções obrigatórias e que os nomes dos arquivos nas skills correspondem aos arquivos que existem em `.claude/commands/`.
