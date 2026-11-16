# AGENTS.md — Ecossistema Unificado Tissia (`PEP` + `tissia-ia`)

Este é o arquivo mestre de contexto e orquestração multi-agente do **Workspace Unificado Tissia**.
Ele consolida as diretrizes dos dois projetos ativos (**PEP** e **tissia-ia**), define fronteiras de contrato entre TypeScript e Python, protege projetos de referência como somente leitura (`Read-Only`) e aponta para a Fonte Única da Verdade documental compatível com Obsidian.

---

## 1. Mapa do Ecossistema

```text
/home/pedro/dev/pedro/tissia-workspace/
├── docs-obsidian/                   # Single Source of Truth (Vault Obsidian)
│   ├── MOC-Index.md                 # Índice mestre com WikiLinks [[...]]
│   ├── 00-arquitetura/              # Visão macro, Mermaid, modelo de dados, glossários, conformidade
│   ├── 01-contratos-api/            # Manual da API v1, mapeamento PEP <-> tissia-ia, SPK-01..13
│   ├── 02-PEP/                      # PRDs, RFCs (RFC-PEP-*) e operação do PEP
│   ├── 03-tissia-ia/                # PRD-IA, RFCs (RFC-IA-*), runbook, evals e incidentes
│   └── 04-adrs/                     # ADRs centralizadas (pep/ADR-PEP-* e ia/ADR-IA-*)
├── projetos/                        # Código-fonte ativo (symlinks para os repositórios Git)
│   ├── PEP/                         # -> /home/pedro/dev/pedro/tissia (Monorepo TypeScript)
│   └── tissia-ia/                   # -> /home/pedro/dev/pedro/tissia-ai (Motor de IA FastAPI)
├── referencias/                     # Projetos de referência e insumos (READ-ONLY ESTRITO)
│   ├── tiss/                        # -> /home/pedro/dev/pedro/tiss (Biblioteca/MCP/Portal TISS)
│   ├── tissia-ai-tunning/           # -> /home/pedro/dev/pedro/tissia-ai-tunning (Tabelas ANS/XSDs)
│   └── pep-medscribe-api-legado/    # -> /home/pedro/dev/pedro/pep-medscribe-api (Legado arquivado)
└── AGENTS.md                        # Este arquivo mestre
```

### Stacks e Localização dos Componentes

| Peça | Caminho no Workspace | Stack Principal | Papel |
| :--- | :--- | :--- | :--- |
| **Documentação Central** | `docs-obsidian/` (via `docs/` nos repos) | Markdown + Obsidian WikiLinks + YAML | Fonte única da verdade para arquitetura, contratos, RFCs e ADRs |
| **PEP — Backend (`@pep/api`)** | `projetos/PEP/apps/api` | NestJS 12 (monólito modular: `api` + `worker`), Drizzle ORM, PostgreSQL 16 + RLS, Redis/BullMQ | Domínio clínico, agenda, prontuário imutável, faturamento TISS, `SpikeAdapter` (`modulos/tissia-ai`) |
| **PEP — UI (`@pep/ui`)** | `projetos/PEP/apps/ui` | React 18, Vite 5, TypeScript, Tailwind CSS, shadcn/ui, TanStack Query, React Router, `nuqs`, PWA | Interface clínica e faturamento (telas 01–26) |
| **PEP — Backoffice (`@pep/backoffice`)** | `projetos/PEP/apps/backoffice` | React, Vite, TypeScript | Operação interna da plataforma e da Spike (telas 01–18), sem acesso a dado clínico |
| **PEP — Contratos (`@pep/contracts`)** | `projetos/PEP/packages/contracts` | TypeScript + Zod `.strict()` | Schemas isomórficos compartilhados entre `@pep/ui` e `@pep/api`, erros RFC 9457 |
| **Motor de IA (`tissia-ia`)** | `projetos/tissia-ia` | Python 3.12, FastAPI, Pydantic v2 (`extra="forbid"`), SQLAlchemy 2 + Alembic, PostgreSQL + `pgvector`/`pg_trgm`, Vertex AI (Gemini 3.x) | API REST `v1` anti-glosa, validador DUT, justificativa Campo 23, sugestão TUSS/CID, preditivo, Scribe, catálogos |

> **Leitura inicial recomendada para qualquer agente:** comece por `docs-obsidian/MOC-Index.md` e `docs-obsidian/00-arquitetura/arquitetura-macro-ecossistema.md`.

---

## 2. Protocolo de Execução Multi-Agente

### 2.1 Regra Obrigatória *Contract-First* (`docs-obsidian/01-contratos-api/`)

**ANTES** de criar, modificar ou remover qualquer endpoint em `projetos/tissia-ia/app/api/` ou `app/schemas.py`, **OU** alterar qualquer client HTTP / DTO de integração em `projetos/PEP/packages/contracts/` ou `projetos/PEP/apps/api/src/modulos/tissia-ai/` (`SpikeAdapter`):

1. Leia obrigatoriamente:
   - `docs-obsidian/01-contratos-api/manual-integracao-tissia-ia-v1.md`
   - `docs-obsidian/01-contratos-api/mapeamento-pep-tissia-ia.md`
   - `docs-obsidian/01-contratos-api/indice-contratos-e-schemas.md`
   - As propostas `SPK-*` aplicáveis em `docs-obsidian/01-contratos-api/propostas-spk/`
   - O snapshot OpenAPI correspondente (`projetos/tissia-ia/tests/snapshots/openapi_v1.json` e/ou `projetos/PEP/apps/api/openapi/openapi.json`).
2. Mudanças na API `v1` do `tissia-ia` devem ser **aditivas** (nunca quebrar consumidores standalone). Todos os objetos de entrada no `tissia-ia` usam `extra="forbid"` (HTTP `422` para chaves inesperadas); logo, o client no `PEP` deve enviar estritamente os campos previstos.
3. Nomenclatura oficial da integração: prefixos de chave `tsk_test_` / `tsk_live_`, headers `X-Tissia-*`, módulo `modulos/tissia-ai` e variáveis `TISSIA_AI_*` (nunca usar os nomes legados `msk_*` ou `X-Spike-*`).
4. Lacuna de contrato descoberta durante o trabalho no `PEP` deve ser proposta em `docs-obsidian/01-contratos-api/propostas-spk/` (`SPK-NN`) ou evoluída como `RFC-IA-*` no `tissia-ia` — nunca altere `tissia-ia` implicitamente sem sincronizar o contrato.

---

### 2.2 Papéis e Regras Invariantes por Projeto

#### Papel A — Ao editar o `PEP` (`projetos/PEP`)
- **Fluxo de implementação:** implemente **uma RFC por vez**, na ordem do mapa de fases em `docs-obsidian/00-arquitetura/hub-pep-tissia.md`. Leia a `RFC-PEP-*` e as `ADR-PEP-*` citadas antes de escrever código.
- **Multi-tenancy e Banco (PostgreSQL RLS + Drizzle):** todo acesso a tabela de domínio passa por `Db.comTenant` (`AsyncLocalStorage` + `SET LOCAL`, `ADR-PEP-0017`). Tabela nova precisa de `tenant_id uuid NOT NULL` e política RLS em `drizzle/rls`. App roda como `app_rw` (sem `BYPASSRLS`). Migrações apenas aditivas (`ADR-PEP-0019`).
- **Fronteiras do monorepo (`ADR-PEP-0018`):** `@pep/ui` nunca importa de `@pep/api`; `@pep/backoffice` nunca importa de `@pep/ui` nem de `@pep/api`; `@pep/contracts` contém apenas schemas Zod `.strict()` e funções puras isomórficas; módulos em `apps/api/src/modulos/` nunca importam `schema.ts` de outro módulo (validar com `pnpm deps`).
- **Isolamento da IA (`SpikeAdapter` / `ADR-PEP-0005`):** apenas `apps/api/src/modulos/tissia-ai/` comunica com o `tissia-ia`. Envie **somente** `clinica_ref` (`un_<uuid>`), `guia_ref` (`gu_<uuid>`), `paciente.ref` (`pa_` + HMAC-SHA256 com sal por clínica), `paciente.sexo` e `paciente.idade`. **Nunca** envie nome, CPF, CNS, carteirinha, telefone ou data de nascimento.
- **Resiliência e Imutabilidade:** falha, timeout (6s) ou `degradado: true` do `tissia-ia` **nunca** bloqueia atendimento ou faturamento. Prontuário é imutável (`ADR-PEP-0007`: correção só por adendo). Nada sugerido pela IA vai à operadora ou ao prontuário sem aceite humano registrado.
- **Segurança, Logs e Códigos ANS:** nunca logue corpo de requisição, texto clínico ou PII. Nunca invente código TUSS/CID/Tabela 38 — leia das bases em `apps/api/ref/` (`ADR-PEP-0024`) ou use `<TUSS_…>`.
- **Idioma:** domínio, comentários no código e mensagens de commit em Português do Brasil (pt-BR).

#### Papel B — Ao editar o `tissia-ia` (`projetos/tissia-ia`)
- **Fluxo de implementação:** implemente **uma RFC por vez**, conforme `docs-obsidian/00-arquitetura/hub-tissia-ia.md`. Leia a `RFC-IA-*` e as `ADR-IA-*` citadas antes de começar.
- **Minimização de Dados e PII (`ADR-IA-0007`):** nunca logue corpo de requisição nem texto clínico (registre apenas IDs, códigos e latências). Todo texto clínico deve passar pelo scrubber em `app/pii.py` antes de ser enviado a qualquer provedor de IA.
- **Provedores de LLM (`ADR-IA-0010`):** Vertex AI com `VERTEX_LOCATION=global` e somente modelos Gemini 3.x vigentes. OpenRouter é permitido apenas em `evals/` com casos `origem: sintetico`.
- **Continuidade Operacional:** falha ou timeout de IA **nunca** gera HTTP `5xx` — devolva `200 OK` com `degradado: true` e o resultado determinístico/preditivo intacto.
- **Rigor Clínico e Regulatório (`ADR-IA-0008`):** a justificativa gerada nunca pode afirmar critério clínico ausente da evolução. Nenhum código TUSS/CID/DUT sugerido fora das bases vigentes em `data/` (validadas por SHA-256 em `data/MANIFEST.yaml`). Nunca invente códigos em exemplos ou testes; use `<TUSS_…>` se necessário.
- **Idioma:** domínio (campos de API, regras, módulos), comentários e commits em Português do Brasil (pt-BR).

---

### 2.3 Regra Estrita `Read-Only` para `/referencias` e `.docs/`

É **TERMINANTEMENTE PROIBIDO** editar, criar, mover ou deletar qualquer arquivo dentro de:
- `referencias/tiss/` (`/home/pedro/dev/pedro/tiss`)
- `referencias/tissia-ai-tunning/` (`/home/pedro/dev/pedro/tissia-ai-tunning`)
- `referencias/pep-medscribe-api-legado/` (`/home/pedro/dev/pedro/pep-medscribe-api`)
- `projetos/tissia-ia/.docs/`

Use esses diretórios **exclusivamente para leitura e consulta** de padrões XML/XSD TISS, regras ANS, tabelas TUSS oficiais, scripts de ingestão ou histórico executivo.

---

## 3. Protocolo de Atualização Contínua de Documentação (`/docs-obsidian`)

A documentação em `docs-obsidian/` só tem valor se permanecer sincronizada com o código. Ao finalizar **qualquer** implementação de feature, correção relevante ou alteração de contrato:

1. **Atualizar a RFC correspondente:**
   - Atualize o campo `status` no frontmatter YAML e no cabeçalho da nota (`RFC-PEP-*` em `docs-obsidian/02-PEP/rfc/` ou `RFC-IA-*` em `docs-obsidian/03-tissia-ia/rfc/`).
   - Caso haja desvio em relação ao desenho original, registre-o na seção `## Notas de implementação` no final da RFC.
2. **Atualizar Contratos de API e Snapshots:**
   - Se alterou rota, header, payload, enum ou regra de validação entre `PEP` e `tissia-ia`, atualize `docs-obsidian/01-contratos-api/manual-integracao-tissia-ia-v1.md` e `docs-obsidian/01-contratos-api/mapeamento-pep-tissia-ia.md`.
   - Regenere o snapshot OpenAPI correspondente (`pnpm --filter @pep/api openapi` no `PEP` e snapshot em `projetos/tissia-ia/tests/snapshots/openapi_v1.json` no `tissia-ia`).
3. **Registrar Decisões Arquiteturais (ADRs):**
   - ADRs com status `aceita` são **imutáveis**.
   - Toda nova decisão arquitetural relevante exige criar uma nova ADR (`ADR-PEP-NNNN-*.md` em `docs-obsidian/04-adrs/pep/` ou `ADR-IA-NNNN-*.md` em `docs-obsidian/04-adrs/ia/`) e adicioná-la ao `docs-obsidian/MOC-Index.md` e ao respectivo hub (`hub-pep-tissia.md` / `hub-tissia-ia.md`).
4. **Padrão Obsidian:**
   - Toda nova nota em `docs-obsidian/` deve conter frontmatter YAML (`projeto`, `tipo`, `status`, `tags`, `aliases`) e usar WikiLinks (`[[nome-da-nota]]` ou `[[nome-da-nota|Texto]]`, escapando `\|` dentro de tabelas Markdown).

---

## 4. Comandos Rápidos de Verificação

### No `PEP` (`cd /home/pedro/dev/pedro/tissia-workspace/projetos/PEP`)
- `pnpm dev` · `pnpm build` · `pnpm typecheck` · `pnpm test` · `pnpm deps` · `pnpm e2e`
- `pnpm --filter @pep/ui dev` · `pnpm --filter @pep/api infra:up` · `pnpm --filter @pep/api db:setup` · `pnpm --filter @pep/api test` · `pnpm --filter @pep/api openapi`

### No `tissia-ia` (`cd /home/pedro/dev/pedro/tissia-workspace/projetos/tissia-ia`)
- `uv run pytest` · `uv run ruff check . && uv run ruff format --check .` · `uv run python -m evals.run --conjunto sinteticos`

