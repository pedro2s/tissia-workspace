---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0017-orm-drizzle-rls.md
---

# ADR-PEP-0017 — Acesso a dados com Drizzle ORM mantendo isolamento RLS por transação

**Status:** Aceita — 07/11/2026 (a cláusula de migrações foi substituída pela [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]: Drizzle Kit gera e aplica as migrações)

## Contexto

O PEP crescerá para cerca de 40 tabelas distribuídas em 17 módulos ([[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]]), com colunas `jsonb` estruturadas (evoluções SOAP/modelos, autoria de IA, análises da Spike), filtros dinâmicos combinados com paginação por cursor e regras estritas de imutabilidade ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]). Escrever todas as consultas e mapeamentos em SQL puro sobre `pg` reduz a velocidade de entrega na P1/P2 e perde verificação estática de colunas e tipos.

Por outro lado, um ORM tradicional baseado em repositórios injetados globalmente arrisca quebrar o isolamento multi-tenant ([[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]]): se qualquer consulta rodar em uma conexão avulsa do pool fora de `Db.comTenant`, o `set_config('app.tenant_id', $1, true)` não estará ativo.

## Decisão

- **Drizzle ORM (`drizzle-orm/node-postgres`)** sobre o driver `pg` já utilizado em `apps/api`.
- **Zero instância global de ORM para tabelas de domínio:** nenhum módulo injeta um cliente Drizzle solto. Todo acesso a dados de tenant passa por `Db.comTenant(ctx, async (tx) => ...)`, que abre transação no pool `app_rw` (sem `BYPASSRLS`), define `app.tenant_id`, `app.usuario_id` e `app.unidade_id` via `set_config(..., true)` e entrega a transação tipada `tx: TenantTx`.
- **Propagação intra-requisição com `AsyncLocalStorage`:** quando um módulo chama o serviço público de outro dentro da mesma requisição ([[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]]), `Db.comTenant` reutiliza a `TenantTx` já aberta para o mesmo `tenantId`, garantindo atomicidade e um único `COMMIT`/`ROLLBACK` com a `outbox`.
- **Schemas isolados por módulo:** cada módulo declara apenas suas próprias tabelas em `src/modulos/<dominio>/schema.ts` (qualificadas pelo schema PostgreSQL do módulo quando aplicável), sem importar tabelas nem declarar chaves estrangeiras (`REFERENCES`) para outros módulos. Checado por `dependency-cruiser`.
- **Colunas `jsonb` tipadas com `@pep/contracts`:** colunas estruturadas usam `.$type<TipoDoContrato>()` validado pelos schemas Zod compartilhados ([[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]]).
- **Imutabilidade explícita:** tabelas append-only (`evolucao_versoes`, `adendos`, `guia_revisoes`, `auditoria_eventos`) permanecem protegidas no banco (permissões + trigger) e só expõem operações de inserção e leitura nos repositórios.
- **Migrações:** SQL versionado em `apps/api/migrations/` (compatível com recursos nativos do PostgreSQL 16 como `aplicar_rls()`, `EXCLUDE USING gist`, índices parciais e partições). O teste de integração que varre `pg_class` continua como gate obrigatório de CI.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| TypeORM | `@InjectRepository` padrão roda fora de transação e contorna o contexto `set_config`; gerador de migrações conflita com `EXCLUDE USING gist`, partições, triggers e políticas RLS; `save()` mascara `UPDATE` implícito; decorators incentivam importar entidades entre módulos |
| Prisma | Motor binário extra, menor controle sobre `FOR UPDATE SKIP LOCKED`/índices parciais/GIST e maior atrito para garantir `SET LOCAL` por transação sem extensões complexas |
| Manter só SQL puro (`pg`) | Seguro para RLS, mas verboso e propenso a erros de digitação/tipagem em projeções, filtros opcionais e cursores ao escalar para 40 tabelas |

## Consequências

- Consultas SQL continuam previsíveis (1:1 com o SQL gerado) e `tx.execute(sql`...`)` permanece disponível na mesma transação para casos específicos do PostgreSQL.
- `DbAdmin` (papel `app_admin` com `BYPASSRLS`) continua restrito ao relay do `outbox` e jobs administrativos auditados.
- Detalhes de implementação atualizados na [[RFC-PEP-000-fundacoes|RFC-PEP-000]].
