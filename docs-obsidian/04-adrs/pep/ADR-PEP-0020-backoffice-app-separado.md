---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0020-backoffice-app-separado.md
---

# ADR-PEP-0020 — Backoffice interno como app separado no monorepo (`apps/backoffice`)

**Status:** Aceita — 08/11/2026
**Complementa:** [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]] (adiciona um app ao monorepo)

## Contexto

O design do **Tissia Backoffice** (18 telas: acesso SSO + FIDO2, visão geral, clínicas, planos, integradores da API v1, observabilidade/análises/incidentes da Spike, modelos, prompts, avaliações, bases regulatórias, regras, curadoria DUT, feedback, modelo de risco, acessos e LGPD) é uma ferramenta **só da equipe interna**, com público, autenticação, riscos e ciclo de release diferentes dos do PEP usado por clínicas. Misturá-lo ao `@pep/ui` exporia rotas e código de operação aos usuários clínicos e acoplaria os deploys.

## Decisão

1. **App separado `apps/backoffice` (`@pep/backoffice`)**, SPA React + Vite + TypeScript, deploy próprio (`backoffice-vX.Y.Z`), porta de desenvolvimento 5174, `noindex`.
2. **Reaproveita o monorepo**, não o `@pep/ui`: `@pep/backoffice` não importa de `@pep/ui` nem de `@pep/api` (mesma regra de fronteira da ADR-PEP-0018). Compartilha `@pep/tsconfig` e, quando houver backend, `@pep/contracts` (schemas Zod estritos).
3. **Fase atual: telas estáticas** (referência visual fiel ao design, com `Sidebar`/`Topbar` reais e rotas), no mesmo padrão das telas 04–26 do `@pep/ui`; cada tela leva CSS escopado (`.sNN`). A migração para dados reais acontece por domínio, uma RFC por vez.
4. **Tokens visuais copiados por ora** (paleta, tipografia Inter). Extrair um pacote `@pep/ds` só quando houver componentes interativos reais usados pelos dois apps (hoje seriam cópias de CSS estático).

## Decisões derivadas

As questões que ficaram em aberto na primeira versão desta ADR foram tratadas em:

- **Necessidade e escopo por etapa (B0/B1/B2):** [[PRD-003-backoffice|PRD-003]].
- **Backend, processo, ingresso e autenticação da equipe:** [[ADR-PEP-0021-backoffice-backend|ADR-PEP-0021]].
- **Fronteira de dados e acesso de suporte:** [[ADR-PEP-0022-backoffice-sem-dado-clinico|ADR-PEP-0022]].
- **Operação da Spike:** [[ADR-PEP-0023-spike-no-backoffice|ADR-PEP-0023]] e [[SPK-06-api-de-operacao|SPK-06]].

## Consequências

- Deploy e superfície de ataque do backoffice isolados do PEP clínico.
- Custo: CSS base duplicado entre os dois apps até a extração do `@pep/ds`.
