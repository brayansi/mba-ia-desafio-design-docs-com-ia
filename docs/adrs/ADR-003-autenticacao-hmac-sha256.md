# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period

**Status:** Aceito
**Data:** 2026-06-29
**Autores:** Sofia (Eng. Segurança), Larissa (Tech Lead)
**Revisores:** Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Marcos (PM)

## Contexto

O sistema envia eventos com dados de pedidos para endpoints externos fora da nossa infraestrutura. Os clientes precisam verificar que a requisição veio realmente da plataforma e que o payload não foi adulterado em trânsito [09:19, Sofia]. Há um precedente de cliente que vazou uma secret em log de aplicação [09:22, Diego], o que exigiu pensar em rotação de chave sem downtime para o cliente.

## Decisão

Assinar o corpo de cada requisição de webhook com HMAC-SHA256 usando uma secret exclusiva por endpoint cadastrado. A assinatura é enviada no header `X-Signature`. Cada endpoint tem sua própria secret gerada pela plataforma no momento do cadastro e devolvida ao cliente. Suporte a rotação com grace period de 24 horas: quando o cliente solicita nova secret, a antiga permanece válida por 24 horas em paralelo, permitindo que ele migre seus sistemas sem interrupção.

## Alternativas Consideradas

### Alternativa 1: Secret global da plataforma
- **Descrição:** Usar uma única secret compartilhada entre todos os endpoints e clientes da plataforma.
- **Por que foi descartada:** Se a secret vazar em qualquer cliente, compromete todos os webhooks de todos os clientes simultaneamente. O isolamento por endpoint limita o raio de explosão de um vazamento [09:21, Sofia].

### Alternativa 2: Autenticação por token JWT no header
- **Descrição:** Incluir um JWT assinado como header de autenticação em vez de HMAC sobre o payload.
- **Por que foi descartada:** JWT autentica quem enviou, mas não garante integridade do corpo da mensagem. HMAC-SHA256 sobre o payload oferece as duas garantias simultaneamente com implementação mais simples para o cliente [09:20, Sofia].

## Consequências

**Positivas:**
- HMAC-SHA256 é padrão de mercado amplamente suportado por bibliotecas em todas as linguagens relevantes para clientes B2B.
- Secret por endpoint limita o impacto de um vazamento a um único cliente/endpoint.
- Grace period de 24 horas na rotação elimina downtime durante migrações de chave.
- TLS obrigatório na URL cadastrada (validação no schema Zod) reforça a proteção em trânsito [09:23, Sofia].

**Negativas / Trade-offs:**
- A responsabilidade de verificar a assinatura recai sobre o cliente — requer documentação clara no portal de desenvolvedor.
- A implementação do grace period exige armazenar tanto a secret atual quanto a anterior na tabela de configuração de webhooks, com timestamp de expiração da antiga.
- Sofia reservou pelo menos dois dias úteis para revisão do código de geração de secret e do mecanismo de HMAC antes do deploy [09:46, Sofia].

## Referências

- TRANSCRICAO.md [09:19–09:22] Sofia, Bruno, Diego
- TRANSCRICAO.md [09:46] Sofia
- `src/shared/logger/index.ts` — redação de `*.token` e `*.accessToken` já presente; a secret do webhook deve ser tratada com o mesmo cuidado de redação nos logs
