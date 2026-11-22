---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0023-spike-no-backoffice.md
---

# ADR-PEP-0023 — Spike no backoffice: só pela API de operação da Spike; conteúdo editorial continua em git

**Status:** Aceita — 08/11/2026
**Relacionadas:** [[PRD-002-spike-standalone|PRD-002 §4]] (neutralidade), [[SPK-06-api-de-operacao|SPK-06]], Spike ADR-0006 (bases versionadas), Spike ADR-0009 (regras em YAML), Spike RFC-005 (harness e portão)

## Contexto

As telas 06–17 do backoffice operam a Spike: integradores e chaves, saúde, análises, incidentes, roteamento de modelos, prompts, avaliações, bases, regras, DUT, feedback e risco. A Spike mora em outro repositório, tem banco próprio e é vendida a terceiros. Nela, regras, bases e prompts são arquivos versionados em git, revisados em PR e validados no deploy.

## Decisão

1. **O backoffice não acessa o banco da Spike nem a infra dela.** Lê e age só pela **API de operação da Spike** (`/ops/v1`, [[SPK-06-api-de-operacao|SPK-06]]). Essa API é da Spike, documentada e testada no repositório dela, e separada da `/v1` dos integradores.
2. **Autenticação pelo operador, não pelo PEP:** o processo `backoffice` repassa o token do usuário interno (realm da equipe, audiência `spike-operacao`). A Spike valida o JWT pela JWKS, aplica os próprios papéis e audita o ator. Não existe chave de serviço do PEP com poder de operação, e a neutralidade fica preservada: o `@pep/api` do produto continua usando só `/v1`.
3. **Git continua sendo a fonte de verdade** de regras, bases, DUT e prompts. O backoffice mostra versão em produção, diff, resultado do portão, aprovações e link para o PR. Não há editor. Ações operacionais com efeito imediato e reversível (percentual de roteamento entre provedores, canário de prompt já aprovado, rotação e revogação de chave) entram pela `/ops/v1`, com segunda aprovação quando o design pede.
4. **Observabilidade:** latências, erros e alertas vêm do Cloud Monitoring/Logging (a Spike já loga em JSON por requisição, Spike RFC-000). A `/ops/v1` expõe um resumo e o backoffice linka para os painéis. Incidentes ficam na status page e no canal; o backoffice só mostra o incidente aberto.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Backoffice lê o banco da Spike (réplica ou credencial) | Quebra a neutralidade e acopla o PEP ao schema interno da Spike |
| Editor de regras, prompts e DUT no backoffice | Cria uma segunda fonte de verdade ao lado dos YAML e prompts versionados e contorna PR e CI; a dupla aprovação já é a revisão de PR |
| Backoffice próprio dentro do `tissia-ai` | Daria dois consoles para a mesma equipe. Pode ser revisto se a Spike virar empresa ou equipe separada |
| Construir painéis de métricas e incidentes | Duplica o Cloud Monitoring sem ganho com o volume atual |

## Consequências

- A operação da Spike no backoffice (B2) depende da SPK-06 ser aceita e implementada no `tissia-ai`.
- As telas 11, 13, 14 e 15 do design mudam de "editar" para "ver e aprovar" (ajustes no PRD-003 §6).
- Se a Spike ganhar uma equipe própria, a `/ops/v1` continua servindo os dois lados.
