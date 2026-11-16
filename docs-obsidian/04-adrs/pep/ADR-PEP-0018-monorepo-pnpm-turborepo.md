---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0018-monorepo-pnpm-turborepo.md
---

# ADR-PEP-0018 — Monorepo `pnpm` + `Turborepo` para UI, API e contratos do PEP; Spike permanece separada

**Status:** Aceita — 07/11/2026
**Substitui parcialmente:** [[ADR-PEP-0001-repositorios|ADR-PEP-0001]] (unifica `pep-medscribe` e `pep-medscribe-api`; mantém `tissia-ai` separado)

## Contexto

A [[ADR-PEP-0001-repositorios|ADR-PEP-0001]] separou UI e backend do PEP em dois repositórios com a ressalva explícita de reavaliar caso a sincronia de contratos gerasse atrito. Com 1 desenvolvedor trabalhando com agentes para entregar o MVP em 90 dias (uma RFC vertical por vez: banco + API + tela + E2E), dois repositórios exigem:

1. escrever validações duas vezes (`class-validator` + `@ApiProperty` na API e `zod` nos formulários da UI);
2. exportar e copiar snapshots `openapi.json` entre repositórios a cada alteração de campo;
3. coordenar dois PRs para qualquer entrega de domínio e duplicar funções puras isomórficas (cálculo de escores clínicos, condições `exibir_se`, limites como Campo 23 ≤ 500).

A Spike (`tissia-ai`), por outro lado, é um serviço Python 3.12 vendido de forma independente para outros PEPs ([[PRD-002-spike-standalone|PRD-002]]), com ciclo de vida e regra de neutralidade próprios.

## Decisão

Unificar o produto PEP no repositório `pep-medscribe` usando **`pnpm` workspaces + `Turborepo`**, mantendo a Spike isolada em `tissia-ai`:

| Caminho / Repositório | Pacote | Papel e Deploy |
| :--- | :--- | :--- |
| `apps/ui` | `@pep/ui` | SPA/PWA React + Vite + Tailwind + shadcn ([[ADR-PEP-0004-frontend\|ADR-PEP-0004]]) · Estático (Cloud Storage + CDN) |
| `apps/api` | `@pep/api` | NestJS 12 (API + worker), Drizzle ORM ([[ADR-PEP-0017-orm-drizzle-rls\|ADR-PEP-0017]]), migrações · Contêiner (Cloud Run) |
| `packages/contracts` | `@pep/contracts` | Schemas **Zod** estritos (`.strict()`), DTOs compartilhados, erros RFC 9457, schemas `jsonb` clínicos e funções puras isomórficas |
| `packages/tsconfig` | `@pep/tsconfig` | Configurações TypeScript estritas compartilhadas |
| `docs/` | — | Hub único de documentação técnica do PEP e propostas `SPK-NN` |
| Repo `tissia-ai` (separado) | — | Spike (Python 3.12 + FastAPI) · Inalterado (Cloud Run SP) |

Regras invariantes do monorepo:

- **Fonte única de contrato (`@pep/contracts`):** schemas Zod validam a entrada em `@pep/api` (devolvendo `422 application/problem+json` para campos inválidos ou extras), geram o documento OpenAPI (`openapi/openapi.json` versionado em `apps/api` para auditoria de contrato e futura API pública da P3) e alimentam diretamente o `react-hook-form` e o cliente HTTP tipado em `@pep/ui`.
- **Fronteiras de pacotes:** `@pep/ui` nunca importa de `@pep/api` e vice-versa; `@pep/contracts` depende apenas de `zod` (zero código de servidor, banco ou framework no pacote compartilhado). Verificado na CI.
- **Deploys independentes:** tags e pipelines filtrados (`ui-vX.Y.Z` para `@pep/ui`, `api-vX.Y.Z` para `@pep/api` via `turbo --filter`); mudança restrita à UI não faz redeploy da API.
- **Neutralidade com a Spike preservada:** `@pep/api` continua consumindo apenas a API pública `/v1` da Spike pelo módulo `spike` ([[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]]), validada contra o snapshot `openapi_v1.json` copiado do repositório `tissia-ai`.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Manter UI e API em 2 repos | Duplica schemas DTO/Zod e exige 2 PRs + sincronia manual de artefato OpenAPI em toda RFC da P1 |
| Incluir também a Spike (`tissia-ai`) no monorepo | Mistura Python/`uv` com Node/`pnpm` e acopla o ciclo de release de um produto vendido a concorrentes ([[PRD-002-spike-standalone\|PRD-002]]) |
| Nx em vez de Turborepo | Mais plugins e abstrações que o necessário para 2 apps e 2 pacotes TS; `pnpm` + `Turborepo` é mínimo e transparente |

## Consequências

- Uma RFC inteira (migração + contrato + endpoint + tela + testes) é entregue e validada em um único PR atômico (`pnpm turbo run lint typecheck test build`).
- `class-validator` e `class-transformer` deixam de ser necessários nos DTOs novos de `@pep/api`, substituídos pelos schemas Zod de `@pep/contracts`.
- O repositório legado separado `pep-medscribe-api` fica arquivado após a importação de seu código e testes da P0 para `apps/api`.
