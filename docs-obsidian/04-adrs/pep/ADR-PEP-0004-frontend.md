---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0004-frontend.md
---

# ADR-PEP-0004 — Frontend: React + Vite + TS, PWA, Tailwind + shadcn com os tokens do design

**Status:** Aceita — 07/11/2026

## Contexto

A UI atual (`pep-medscribe`) já é React 18 + Vite + TypeScript, com rotas por hash, CSS próprio em `src/styles.css` (tokens do design: teal `#0B6E78`, navy `#0F1B2D`, Inter) e as telas 04–26 convertidas automaticamente do design como JSX estático com CSS escopado por tela (`src/pages/screens/`). O design de produto pede Tailwind + shadcn, PWA para tablet/celular e offline no prontuário. As telas geradas reproduzem o visual, mas não têm componentes, estado nem dados.

## Decisão

- **Base mantida:** React + Vite + TS, SPA instalável como **PWA** (`vite-plugin-pwa`).
- **Estilo:** Tailwind com o tema configurado a partir dos tokens do design (cores, raios, tipografia, espaçamentos); componentes **shadcn/ui** (Radix) recoloridos com esses tokens. `src/styles.css` vira a fonte dos tokens (CSS variables) até a migração terminar.
- **Dados:** **TanStack Query** sobre um cliente tipado gerado do OpenAPI do backend ([[ADR-PEP-0001-repositorios|ADR-PEP-0001]]). Estado de formulário com React Hook Form + Zod.
- **Rotas:** **react-router** substitui o hash router de `src/App.tsx`, com rotas aninhadas no shell e guarda por permissão.
- **Migração progressiva:** uma tela gerada só é reescrita em componentes quando ganha comportamento real na RFC dela. Até lá, continua como referência visual. A tela reescrita precisa bater com o design (teste visual, [[test-plan-pep|test plan]]) e o arquivo gerado é apagado.
- Acessibilidade: o que o design já marca (rótulos, `aria-*`, alvos de 44 px no tablet/mobile) é requisito, não sugestão.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Manter CSS próprio sem Tailwind | Funciona, mas o design de produto escolheu Tailwind + shadcn; componentes acessíveis prontos economizam a P1 |
| Next.js | SSR não traz ganho num app autenticado; complica PWA/offline |
| Reescrever todas as telas antes da P1 | Esforço sem dados reais; a migração junto da RFC evita retrabalho |

## Consequências

- Convivem por um tempo dois estilos (telas geradas e componentes); o CSS escopado (`.s05` etc.) evita colisão.
- Detalhes (estrutura de pastas, design system, estados de carregamento/erro, offline, tablet/mobile) na [[RFC-PEP-015-frontend|RFC-PEP-015]]; offline em [[ADR-PEP-0015-offline-prontuario|ADR-PEP-0015]].
