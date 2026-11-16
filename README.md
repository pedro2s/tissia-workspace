# Tissia Workspace — Ecossistema Unificado (`PEP` + `tissia-ia`)

Repositório mestre de orquestração, documentação centralizada e integração multi-repositório do ecossistema **Tissia**, reunindo o **Prontuário Eletrônico e Faturamento TISS (`PEP`)**, o **Motor de IA Anti-Glosa (`tissia-ia`)** e o **Vault de Documentação (`docs-obsidian`)**.

---

## Estrutura do Repositório

```text
tissia-workspace/
├── AGENTS.md                 # Diretrizes mestras de arquitetura, contratos e orquestração multi-agente
├── CLAUDE.md                 # Symlink para AGENTS.md
├── README.md                 # Visão geral do workspace
├── tissia.code-workspace     # Definição de multi-root workspace para VS Code / Cursor
├── docs-obsidian/            # Single Source of Truth documental (Vault Obsidian)
│   ├── MOC-Index.md          # Índice central (Map of Content)
│   ├── 00-arquitetura/       # Arquitetura macro, modelo de dados, conformidade e glossários
│   ├── 01-contratos-api/     # Manual da API v1, mapeamento PEP <-> tissia-ia e propostas SPK-*
│   ├── 02-PEP/               # PRDs, RFCs (RFC-PEP-*) e planos operacionais do PEP
│   ├── 03-tissia-ia/         # PRD-IA, RFCs (RFC-IA-*), runbook, evals e incidentes
│   └── 04-adrs/              # Decisões arquiteturais (ADR-PEP-* e ADR-IA-*)
├── projetos/                 # Symlinks para os repositórios de código ativo
│   ├── PEP/                  # -> /home/pedro/dev/pedro/tissia (Monorepo TypeScript)
│   └── tissia-ia/            # -> /home/pedro/dev/pedro/tissia-ai (Motor de IA FastAPI)
└── referencias/              # Projetos e tabelas de referência (READ-ONLY ESTRITO)
    ├── tiss/                 # -> /home/pedro/dev/pedro/tiss (Biblioteca/MCP/CLI e schemas TISS/ANS)
    ├── tissia-ai-tunning/    # -> /home/pedro/dev/pedro/tissia-ai-tunning (Tabelas TUSS/ANS e XSDs TISS v4.03.00)
    └── pep-medscribe-api-legado/ # -> /home/pedro/dev/pedro/pep-medscribe-api (API NestJS legada arquivada)
```

> **Observação:** Os diretórios dentro de `projetos/` e `referencias/` são *symlinks* para os repositórios Git irmãos locais. Os diretórios em `referencias/` e `projetos/tissia-ia/.docs/` são estritamente **somente leitura (`Read-Only`)**.

---

## Componentes e Stacks

| Componente | Caminho | Stack Principal | Papel no Ecossistema |
| :--- | :--- | :--- | :--- |
| **Documentação Central** | `docs-obsidian/` | Markdown + Obsidian WikiLinks + YAML | Fonte única da verdade para arquitetura, contratos HTTP, PRDs, RFCs e ADRs |
| **PEP — Backend (`@pep/api`)** | `projetos/PEP/apps/api` | NestJS 12 (`api` + `worker`), Drizzle ORM, PostgreSQL 16 + RLS, Redis/BullMQ | Domínio clínico, agenda, prontuário imutável, faturamento TISS e `SpikeAdapter` (`modulos/tissia-ai`) |
| **PEP — UI (`@pep/ui`)** | `projetos/PEP/apps/ui` | React 18, Vite 5, TypeScript, Tailwind CSS, shadcn/ui, TanStack Query, React Router, `nuqs`, PWA | Interface clínica e de faturamento |
| **PEP — Backoffice (`@pep/backoffice`)** | `projetos/PEP/apps/backoffice` | React, Vite, TypeScript | Operação interna da plataforma e da Spike, sem acesso a dados clínicos |
| **PEP — Contratos (`@pep/contracts`)** | `projetos/PEP/packages/contracts` | TypeScript + Zod `.strict()` | Schemas isomórficos compartilhados entre `@pep/ui` e `@pep/api` e erros RFC 9457 |
| **Motor de IA (`tissia-ia`)** | `projetos/tissia-ia` | Python 3.12, FastAPI, Pydantic v2 (`extra="forbid"`), SQLAlchemy 2 + Alembic, PostgreSQL (`pgvector`/`pg_trgm`), Vertex AI (Gemini 3.x) | API REST `v1` anti-glosa, validador DUT, justificativa Campo 23, sugestão TUSS/CID, preditivo, Scribe e catálogos |

---

## Como Usar o Workspace

### VS Code / Cursor

Abra o arquivo `tissia.code-workspace` diretamente pelo editor ou via terminal:

```bash
code tissia.code-workspace
```

O arquivo configura as pastas do ecossistema, exclui diretórios de build/dependências da busca (`node_modules`, `.venv`, `dist`, `__pycache__`, `.ruff_cache`) e protege `referencias/**` e `**/.docs/**` como somente leitura.

### Documentação no Obsidian

Abra a pasta `docs-obsidian/` como um **Vault** no Obsidian. O ponto de entrada principal é [`docs-obsidian/MOC-Index.md`](docs-obsidian/MOC-Index.md).

---

## Comandos Rápidos de Verificação

### PEP (`projetos/PEP`)

```bash
cd projetos/PEP

# Desenvolvimento, build e checagens gerais do monorepo
pnpm dev
pnpm build
pnpm typecheck
pnpm test
pnpm deps
pnpm e2e

# Comandos específicos por app
pnpm --filter @pep/ui dev
pnpm --filter @pep/api infra:up
pnpm --filter @pep/api db:setup
pnpm --filter @pep/api test
pnpm --filter @pep/api openapi
```

### Motor de IA (`projetos/tissia-ia`)

```bash
cd projetos/tissia-ia

# Testes unitários e de integração
uv run pytest

# Lint e verificação de formatação
uv run ruff check . && uv run ruff format --check .

# Execução de evals sintéticos
uv run python -m evals.run --conjunto sinteticos
```

---

## Governança e Referências Principais

- **Guia mestre de orquestração e contratos:** [`AGENTS.md`](AGENTS.md)
- **Índice mestre da documentação:** [`docs-obsidian/MOC-Index.md`](docs-obsidian/MOC-Index.md)
- **Arquitetura macro e fluxos:** [`docs-obsidian/00-arquitetura/arquitetura-macro-ecossistema.md`](docs-obsidian/00-arquitetura/arquitetura-macro-ecossistema.md)
- **Manual da API `v1` (`PEP` ↔ `tissia-ia`):** [`docs-obsidian/01-contratos-api/manual-integracao-tissia-ia-v1.md`](docs-obsidian/01-contratos-api/manual-integracao-tissia-ia-v1.md)
