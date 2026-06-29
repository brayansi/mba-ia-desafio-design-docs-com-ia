---
description: Gera o TRACKER.md de rastreabilidade, mapeando cada item dos documentos produzidos à origem na transcrição ou no código
---

Você vai gerar o arquivo `docs/TRACKER.md` com a tabela de rastreabilidade completa do pacote de documentação.

## Antes de gerar

Leia todos os documentos produzidos:
- `TRANSCRICAO.md` — transcrição completa (fonte primária)
- `docs/PRD.md` — requisitos de produto
- `docs/RFC.md` — proposta técnica
- `docs/FDD.md` — especificação de implementação
- `docs/adrs/` — todos os ADRs
- Arquivos de código mencionados nos documentos (verifique que existem):
  - `src/modules/orders/order.service.ts`
  - `src/shared/errors/app-error.ts`
  - `src/shared/errors/http-errors.ts`
  - `src/middlewares/auth.middleware.ts`
  - `src/middlewares/error.middleware.ts`
  - `src/config/database.ts`
  - `prisma/schema.prisma`

## O que é o Tracker

O Tracker é uma **referência cruzada** entre os documentos produzidos e suas origens. Ele garante que nenhum requisito, decisão ou restrição foi inventado — tudo tem fonte identificável.

É a principal defesa contra alucinações da IA: se um item não tem localização válida na transcrição ou no código, ele não deveria existir nos documentos.

## Formato obrigatório da tabela

```markdown
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|----- |-------------------|-------|-------------|
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint de webhook com URL, eventos e customer | TRANSCRICAO | [09:31] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Usar padrão Outbox em MySQL para entrega atômica de eventos | TRANSCRICAO | [09:06] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus estendido para chamar publishWebhookEvent dentro da transação | CODIGO | src/modules/orders/order.service.ts |
```

## Tipos aceitos na coluna "Tipo"

- `Requisito Funcional`
- `Requisito Não Funcional`
- `Decisão`
- `Restrição`
- `Trade-off`
- `Exclusão de Escopo`
- `Integração`
- `Contrato de API`
- `Risco`

## Localização por fonte

- **TRANSCRICAO**: formato `[HH:MM] Nome` (ex: `[09:17] Diego`)
- **CODIGO**: caminho relativo do arquivo (ex: `src/modules/orders/order.service.ts`)

## IDs por documento

Use os prefixos abaixo para gerar IDs únicos:

| Documento | Prefixo de ID |
|-----------|--------------|
| docs/PRD.md — Requisito Funcional | `PRD-RF-NN` |
| docs/PRD.md — Requisito Não Funcional | `PRD-RNF-NN` |
| docs/PRD.md — Fora de Escopo | `PRD-ESC-NN` |
| docs/PRD.md — Risco | `PRD-RISCO-NN` |
| docs/RFC.md — Alternativa descartada | `RFC-ALT-NN` |
| docs/RFC.md — Questão em aberto | `RFC-QA-NN` |
| docs/FDD.md — Contrato de API | `FDD-API-NN` |
| docs/FDD.md — Erro | `FDD-ERR-NN` |
| docs/FDD.md — Integração com código | `FDD-INT-NN` |
| docs/FDD.md — Resiliência | `FDD-RES-NN` |
| docs/adrs/ADR-NNN-*.md | `ADR-NNN` |

## Cobertura mínima obrigatória

- **≥ 80%** dos itens identificáveis nos documentos devem ter linha no Tracker
- **≥ 70%** das linhas devem ter `Fonte = TRANSCRICAO` com timestamp válido no formato `[HH:MM] Nome`
- **≥ 5 linhas** devem ter `Fonte = CODIGO` com caminho de arquivo real

## Itens que DEVEM obrigatoriamente estar no Tracker

### Do PRD
Todos os requisitos funcionais (RF-01 a RF-12), todos os itens fora de escopo, pelo menos 2 riscos.

### Do RFC
As 2+ alternativas consideradas, as 2+ questões em aberto.

### Do FDD
Os 4+ endpoints de contrato de API, os 4+ pontos de integração com o sistema existente, os principais erros `WEBHOOK_*`.

### Dos ADRs
Cada ADR deve ter pelo menos 1 linha no Tracker representando a decisão principal.

## Verificação antes de salvar

Para cada linha com `Fonte = TRANSCRICAO`: confirme que o timestamp e o nome do falante existem na TRANSCRICAO.md.

Para cada linha com `Fonte = CODIGO`: confirme que o arquivo existe no repositório antes de incluir. Arquivos que **existem e devem ser referenciados**:
- `src/modules/orders/order.service.ts` ✓
- `src/shared/errors/app-error.ts` ✓
- `src/shared/errors/http-errors.ts` ✓
- `src/middlewares/auth.middleware.ts` ✓
- `src/middlewares/error.middleware.ts` ✓
- `src/config/database.ts` ✓
- `prisma/schema.prisma` ✓

**Não inclua caminhos de arquivo que não existem no repositório.**

## Saída esperada

Escreva o conteúdo diretamente em `docs/TRACKER.md`. Após gerar, reporte:
- Total de linhas na tabela
- Quantas têm Fonte = TRANSCRICAO (com %)
- Quantas têm Fonte = CODIGO (com %)
- Se a cobertura mínima de 80% foi atingida
