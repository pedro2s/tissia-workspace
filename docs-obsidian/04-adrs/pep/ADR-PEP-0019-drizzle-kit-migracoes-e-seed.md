---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0019-drizzle-kit-migracoes-e-seed.md
---

# ADR-PEP-0019 — Drizzle Kit para migrações e seed

**Status:** Aceita — 08/11/2026
**Substitui parcialmente:** [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]] (somente a cláusula "Migrações: SQL versionado com `node-pg-migrate`")

## Contexto

A [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]] adotou o Drizzle ORM para consultas, mas manteve as migrações em SQL escrito à mão (`node-pg-migrate`) e o seed em SQL bruto sobre `pg`. Isso deixou dois formatos de verdade para o mesmo schema (os `schema.ts` e os `.sql`), que podiam divergir sem aviso, e três ferramentas no banco (Drizzle, `node-pg-migrate`, scripts `pg`). O projeto ainda está na P0 e sem dado real, o que torna esta a hora mais barata de unificar. O projeto `knowee-v2` já opera esse desenho.

## Decisão

- **Fonte única de schema:** os `schema.ts` por módulo. `drizzle-kit generate` produz as migrações em `apps/api/drizzle/` e `drizzle-kit migrate` as aplica (histórico em `drizzle.__drizzle_migrations`, fora do schema `public`). A baseline anterior foi reescrita como `0000_baseline`.
- **O que o schema não expressa fica em SQL idempotente ao lado, em duas pastas:**
  - `drizzle/prelude/*.sql`: grants dos papéis, aplicado por `pnpm db:bootstrap` antes das migrações. Os papéis `app_rw` (NOBYPASSRLS) e `app_admin` (BYPASSRLS) são criados pelo script, com senha vinda de `APP_RW_PASSWORD`/`APP_ADMIN_PASSWORD` (padrão `dev` só fora de produção); o contêiner do Postgres deixa de montar `init-roles.sql`.
  - `drizzle/rls/*.sql`: RLS **por convenção**. Toda tabela de `public` com `tenant_id uuid NOT NULL` recebe RLS habilitada, forçada e a política `tenant_isolation`; `tenants` tem política própria sobre `id`. Aplicado por `pnpm db:rls`, encadeado em `pnpm db:migrate`, que também reafirma os grants e **falha** se alguma tabela de tenant ficou sem RLS. Substitui a função `aplicar_rls()` chamada a cada migração.
- **SQL que o schema não descreve** (trigger append-only, `EXCLUDE USING gist`, partições): `drizzle-kit generate --custom` cria uma migração vazia na mesma sequência. Nunca editar migração já aplicada em ambiente compartilhado.
- **Seed em TypeScript com Drizzle** (`scripts/seed-dev.ts`), idempotente, com o papel `app_admin` (semear é operação administrativa entre tenants). Não roda em produção.
- **CI:** `pnpm --filter @pep/api db:generate` seguido de `git diff --exit-code apps/api/drizzle` falha o PR quando o schema mudou sem a migração correspondente.
- **FKs entre módulos continuam proibidas** (ADR-PEP-0017); o schema é a verdade e a baseline não tem FK de `tenant_id` nas tabelas de `comum` e de `identidade`.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Manter `node-pg-migrate` com SQL manual | Dois formatos para o mesmo schema, sem checagem de que batem; mais ferramentas |
| Políticas via `pgPolicy` no schema Drizzle | Não há `FORCE ROW LEVEL SECURITY` no schema, e cada tabela nova dependeria de alguém declarar a política (tabela sem política vaza) |
| Função `aplicar_rls()` por migração | Depende de lembrar de chamá-la; a convenção por `tenant_id` mais a verificação falham fechado |

## Consequências

- Comandos: `pnpm db:generate`, `db:bootstrap`, `db:migrate`, `db:rls`, `db:setup` (bootstrap + migrate), `db:check`, `seed:dev`. `pnpm test` e o E2E rodam `db:setup`.
- Tabelas existentes só ganham RLS por convenção se tiverem `tenant_id NOT NULL`; uma tabela de tenant sem essa coluna precisa de política própria em `drizzle/rls` e de entrada na verificação.
- Ambientes locais criados antes desta ADR precisam recriar o banco (`docker compose down -v`), porque a baseline foi reescrita.
