---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-000-fundacoes.md
---

# RFC-PEP-000 — Fundações e arquitetura de referência

**Fase:** P0 · **Status:** Implementada (monorepo `pnpm` + `Turborepo` com `@pep/api`, `@pep/ui` e `@pep/contracts`; faltam infra GCP Terraform e deploy) · **ADRs:** ADR-PEP-0002, 0003, 0005, 0008, 0009, 0017, 0018
**Telas:** nenhuma diretamente (habilita todas)
**Requisitos:** RNF-* do PRD-001; RF-FND-01…RF-FND-08 (abaixo; família `FND` separada de `RF-PLT-*` do PRD-001)

## Objetivo

Estruturar o monorepo `tissia` (`apps/api`, `apps/ui`, `packages/contracts`, `packages/tsconfig`) e a base que toda RFC da P1 usa: NestJS 12 modular e regra de dependência, multitenancy com RLS + Drizzle ORM, contratos compartilhados em Zod, outbox e eventos, observabilidade sem PII, padrões de API, CI/CD e ambientes. Nenhuma regra de domínio entra aqui.

## Escopo

| Dentro | Fora |
| :--- | :--- |
| Monorepo `pnpm` + `Turborepo`, esqueleto NestJS 12 (`@pep/api`), módulos vazios com fronteiras | Regras de negócio dos módulos (RFCs 002+) |
| Tenancy (Tenant/Clínica/Unidade), RLS e `Db.comTenant` com Drizzle ORM + `AsyncLocalStorage` | Identidade e MFA (RFC-PEP-001) |
| Pacote `@pep/contracts` (schemas Zod estritos, erros RFC 9457, paginação) | Consumidores de eventos específicos |
| Outbox + BullMQ, contrato de eventos | Painéis de BI (RFC-PEP-012) |
| Logs, traces, métricas, health | Telas (RFC-PEP-015) |
| Padrões REST, OpenAPI gerado de `@pep/contracts` + Swagger | Multi-região, DR ativo-ativo |
| Infra GCP `southamerica-east1`, CI/CD, `staging`/`prod` | |

## Requisitos

| ID | Requisito |
| :--- | :--- |
| RF-FND-01 | Todo dado de negócio pertence a um tenant; uma consulta sem tenant definido não retorna linhas (RLS). |
| RF-FND-02 | Todo evento de domínio é gravado na mesma transação da mudança (outbox) e publicado ao menos uma vez. |
| RF-FND-03 | Nenhum log, trace ou métrica contém PII ou texto clínico. |
| RF-FND-04 | API REST versionada em `/api/v1`, com OpenAPI gerado e versionado a cada build. |
| RF-FND-05 | Erros no formato RFC 9457 (`application/problem+json`). |
| RF-FND-06 | POST que cria recurso aceita `Idempotency-Key`. |
| RF-FND-07 | Listagens paginadas por cursor. |
| RF-FND-08 | `@pep/ui` e `@pep/api` compartilham contratos tipados e schemas Zod em `@pep/contracts`; nenhum DTO ou schema de validação duplicado entre front e back. |

## Design

### Arquitetura de referência

```
 [PWA React (@pep/ui, CDN)]        [Webhooks/API pública (P3)]
            │ HTTPS + OIDC (Keycloak)          │
            ▼                                   ▼
 ┌──────────────────────── @pep/api (NestJS 12 + Drizzle, Cloud Run) ──────────────────┐
 │ identidade · organizacao · pacientes · agenda · prontuario · documentos · modelos   │
 │ faturamento · retorno · contratos · financeiro · assinatura · notificacoes          │
 │ auditoria · integracoes · bi · spike (SpikeAdapter)                                 │
 │            │ transação Drizzle = escrita + outbox (set_config RLS)                  │
 └────────────┼───────────────────────────────┬───────────────────────────────────────┘
              ▼                               ▼
   [Cloud SQL Postgres 16 + RLS]     [Memorystore Redis] ◄── BullMQ
              ▲                               │
              │                  ┌────────────┴─────────────┐
              │                  ▼                          ▼
   [GCS: anexos, XML, PDFs]  [workers (mesma imagem,   [relay do outbox]
                              processo separado)]
                                 │
        ┌────────────────────────┼─────────────────────────────┐
        ▼                        ▼                              ▼
 [Spike / Tissia IA v1]  [Operadoras TISS (P2)]   [WhatsApp, PSC ICP, e-mail]
```

- Uma imagem para `@pep/api`, dois modos de execução: `api` (HTTP) e `worker` (consumidores BullMQ + relay do outbox). Ambos no Cloud Run; o worker com instância mínima 1.
- Toda chamada externa sai de um módulo `integracoes`/`spike` e roda em worker, exceto `POST /v1/guias/analisar` da Spike, que é síncrono por desenho (RFC-PEP-005).

### Monorepo e módulos (`ADR-PEP-0018`)

```
tissia/
├── apps/
│   ├── ui/                         # @pep/ui — React + Vite + Tailwind + shadcn
│   └── api/                        # @pep/api — NestJS 12 + Drizzle ORM
│       ├── src/
│       │   ├── main.ts             # modo api
│       │   ├── worker.ts           # modo worker
│       │   ├── comum/              # tenancy + Drizzle, auth guard, erros, paginação, idempotência, outbox, logger, zod-dto
│       │   └── modulos/<dominio>/  # controller, service, schema.ts (Drizzle), repositório, eventos
│       ├── drizzle/                # migrações geradas (Drizzle Kit) + prelude/ (grants) + rls/ (políticas)
│       ├── test/                   # integração com Postgres e Redis reais
│       └── openapi/openapi.json    # snapshot versionado
├── packages/
│   ├── contracts/                  # @pep/contracts — schemas Zod estritos, DTOs, erros RFC 9457, funções puras
│   └── tsconfig/                   # @pep/tsconfig — base TypeScript estrita
├── docs/                           # hub único de documentação técnica
├── pnpm-workspace.yaml
└── turbo.json
```

Regras de dependência:

- `@pep/ui` e `@pep/api` nunca importam um do outro; ambos importam de `@pep/contracts`.
- `@pep/contracts` depende apenas de `zod` (zero dependência de NestJS, React, banco ou Node).
- Dentro de `@pep/api`, um módulo só importa de outro módulo pela **API pública** dele (o `*.module.ts` / `*.service.ts` exportado), nunca `schema.ts`, repositório ou controller. Verificado por `dependency-cruiser` na CI.
- Sem ciclos. Reação a outro domínio vai por evento, não por chamada.
- `spike` só é chamado por `faturamento` e `prontuario` (síncrono) e por consumidores de eventos (assíncrono).
- `comum` não depende de nenhum módulo de domínio.

### Tenancy, RLS e Drizzle ORM (`ADR-PEP-0003`, `ADR-PEP-0017`)

| Nível | O que é | Isolamento |
| :--- | :--- | :--- |
| Tenant | Contrato comercial (grupo de clínicas) | RLS em todas as tabelas (`tenant_id`) |
| Clínica | CNPJ prestador | Coluna `clinica_id`; filtro por permissão e sal do HMAC da Spike |
| Unidade | Local de atendimento; `clinica_ref` da Spike | Coluna `unidade_id`; escopo do perfil |

```sql
ALTER TABLE <tabela> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <tabela> FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <tabela>
  USING (tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid)
  WITH CHECK (tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid);
```

- Todo acesso a tabela de domínio passa por `Db.comTenant(ctx, async (tx) => ...)`, que abre transação no pool `app_rw`, executa `SELECT set_config('app.tenant_id', $1, true), set_config('app.usuario_id', $2, true), set_config('app.unidade_id', $3, true)` e entrega `tx` (instância transacional do Drizzle ORM sobre o `PoolClient`, além do cliente `pg` subjacente quando necessário).
- `Db.comTenant` usa `AsyncLocalStorage` para reutilizar a mesma transação `tx` quando um módulo chama o serviço público de outro dentro da mesma requisição. Fora de `comTenant`, nenhuma instância do ORM é exposta aos módulos.
- A aplicação conecta com o papel `app_rw` (sem `BYPASSRLS`). Migrações usam `app_owner`. Jobs administrativos entre tenants (como o relay do outbox) usam `app_admin` (`DbAdmin`), auditado.
- Teste obrigatório: para cada tabela, uma consulta sem `app.tenant_id` retorna 0 linhas e um `INSERT` com outro tenant falha.
- Escopo de unidade não é RLS (um gestor vê várias): é filtro na camada de serviço a partir das permissões do token (RFC-PEP-001).

### Configuração e segredos

- Configuração tipada e validada no startup com `zod`. Variável faltando = processo não sobe.
- Segredos no Secret Manager, montados como variável de ambiente pelo Cloud Run. Nada em repo nem em imagem.
- Chave de API da Spike é por ambiente; sal do `paciente.ref` é por clínica (`clinica_id`) e chaves de criptografia de campo são por tenant no Secret Manager/KMS (ADR-PEP-0005, RFC-PEP-014).

### Migrações

- Drizzle Kit ([[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]): `pnpm db:generate` gera a migração em `apps/api/drizzle/` a partir dos `schema.ts` por módulo e `pnpm db:migrate` a aplica. Uma migração por PR que muda schema; a CI falha se o schema mudou sem migração.
- Só aditivas enquanto houver dado real: nada de `DROP`/`RENAME` de coluna em uso. Remoção = duas releases (parar de usar → remover).
- Toda tabela nova (inclusive filhas) nasce com `tenant_id uuid NOT NULL`: o RLS forçado e a política `tenant_isolation` são aplicados por convenção (`drizzle/rls`, `pnpm db:rls`), que falha se alguma ficou sem. Checado também por teste que varre `pg_class`.

### Outbox e eventos

```sql
CREATE TABLE outbox (
  id            uuid PRIMARY KEY,           -- uuid v7
  tenant_id     uuid NOT NULL,
  tipo          text NOT NULL,              -- ex.: AtendimentoFechado
  versao        int  NOT NULL DEFAULT 1,
  agregado_id   uuid NOT NULL,
  payload       jsonb NOT NULL,             -- só ids e códigos, nunca texto clínico
  criado_em     timestamptz NOT NULL DEFAULT now(),
  publicado_em  timestamptz
);
CREATE INDEX ix_outbox_pendentes ON outbox (criado_em) WHERE publicado_em IS NULL;
```

- O serviço grava a mudança e a linha do outbox na mesma transação (`TenantTx`).
- O relay (worker) lê pendentes com `FOR UPDATE SKIP LOCKED`, publica em uma fila BullMQ por tipo de evento e marca `publicado_em`.
- Entrega ao menos uma vez. Consumidores são idempotentes pela chave `outbox.id` (tabela `eventos_processados (consumidor, evento_id)`).
- Retry com backoff exponencial (5 tentativas); depois disso, fila de mortos e alerta.
- `# ponytail:` relay por polling (1 s). Trocar por `LISTEN/NOTIFY` se a latência entre evento e reação importar.

Eventos canônicos (payload mínimo: `tenant_id`, ids, códigos):

| Evento | Emitido por | Consumidores principais |
| :--- | :--- | :--- |
| AtendimentoFechado | agenda/prontuario | faturamento (cria rascunho), receita esquecida (P2) |
| EvolucaoVersionada | prontuario | auditoria, bi |
| GuiaRascunhoCriada | faturamento | spike (analisar, se não veio síncrono) |
| GuiaRevisada | faturamento | spike (reanálise com o mesmo `guia_ref`) |
| AnaliseTissiaAiConcluida | spike | faturamento, notificacoes, bi |
| FeedbackRegistrado | spike | bi |
| GuiaAssinada | assinatura | faturamento (libera para lote) |
| LoteGerado | faturamento | integracoes, spike (ingerir lote) |
| LoteEnviado | integracoes | bi, notificacoes |
| RetornoImportado | retorno | spike (ingerir/desfechos), financeiro, bi |
| DesfechoRegistrado | retorno | spike (desfecho), financeiro (repasse), bi |

### Padrões de API

| Tema | Padrão |
| :--- | :--- |
| Base | `/api/v1`, JSON, `camelCase` nos DTOs públicos, datas ISO 8601 com fuso |
| Auth | `Authorization: Bearer <access token OIDC>`; tenant e unidade ativa vêm do token, nunca do corpo |
| Erros | RFC 9457: `{ type, title, status, detail, instance, errors? }`; `type` é URI estável por erro de domínio |
| Validação | Schemas `zod` com `.strict()` vindos de `@pep/contracts` validados por `ZodValidationPipe` global (campo extra ou inválido = 422 com `{ campo, mensagem }`, mesmo espírito da Spike) |
| Paginação | `?limite=50&cursor=<opaco>` → `{ itens, proximoCursor }`; sem `offset` |
| Idempotência | `Idempotency-Key` em POST de criação; resposta guardada 24 h em Redis por (tenant, rota, chave) |
| Concorrência | `ETag`/`If-Match` em recursos editáveis (guia, agenda) → 412 em conflito |
| OpenAPI | Gerado pelo `@nestjs/swagger` a partir dos DTOs Zod (`z.toJSONSchema()`); snapshot em `apps/api/openapi/openapi.json`; mudança exige revisão |
| Contratos na UI | `@pep/ui` importa diretamente schemas e tipos de `@pep/contracts` (formulários e cliente HTTP tipado), sem código duplicado |
| Correlação | `X-Request-Id` aceito ou gerado; propagado para logs, traces e chamadas à Spike |

### Observabilidade

- Logs JSON (pino) com `request_id`, `tenant_id`, `usuario_id`, rota, status, latência. Redação por lista de permissão: só campos marcados como seguros são logados. Corpo de request/response nunca.
- Traces OpenTelemetry → Cloud Trace. Spans de chamada à Spike com atributos `analise_id`, `degradado`, `latencia_ms`.
- Métricas (OpenTelemetry → Cloud Monitoring): latência p50/p95 por rota, erros 5xx, profundidade das filas, idade do outbox mais antigo, latência e taxa de `degradado` da Spike, taxa de aceite de sugestões.
- Health: `/healthz` (processo) e `/readyz` (Postgres e Redis).
- Alertas iniciais: outbox pendente > 5 min; fila de mortos > 0; 5xx > 1% em 15 min; p95 da Spike > 6 s em 15 min.

### CI/CD e ambientes

| Ambiente | Uso | Deploy |
| :--- | :--- | :--- |
| local | Desenvolvimento | `docker compose` (Postgres, Redis, Keycloak); Spike local ou `staging` |
| `staging` | Integração, E2E, homologação com a Spike `staging` | Automático a cada merge em `main` (filtrado por app alterado via Turborepo) |
| `prod` | Clínicas | Tags `ui-vX.Y.Z` e `api-vX.Y.Z` + aprovação manual |

Pipeline (GitHub Actions + Workload Identity Federation): `pnpm install --frozen-lockfile`, `pnpm turbo run lint typecheck deps test build`, testes de integração com Postgres e Redis reais, checagem de RLS, diff do snapshot OpenAPI, `pnpm audit`, build da imagem no Artifact Registry, migração e deploy.

## UI

Sem tela própria. `@pep/ui` importa schemas, tipos e constantes de `@pep/contracts` (RF-FND-08); detalhes na RFC-PEP-015.

## Segurança/LGPD

- RLS forçado em toda tabela de domínio; papel da aplicação sem `BYPASSRLS`.
- Criptografia em trânsito (TLS) e em repouso (Cloud SQL/GCS). Campos de PII de alto risco (CPF, CNS, carteirinha) com criptografia de campo por tenant (KMS) e hash para busca.
- Nada de PII em logs, traces, métricas, payload de evento ou mensagem de fila.
- Dados e processamento em `southamerica-east1` (ADR-PEP-0009).

## Critérios de aceite

- [x] Monorepo `tissia` com `apps/api` (`api` + `worker`), `apps/ui` e `packages/contracts` compilando via `pnpm` + `Turborepo`.
- [x] Teste que varre todas as tabelas: RLS habilitado e forçado; consulta sem tenant retorna 0 linhas; escrita cruzada falha.
- [x] Evento gravado no outbox na mesma transação; teste derruba o processo entre commit e publicação e o evento é publicado depois.
- [x] Consumidor idempotente: o mesmo evento entregue 2× produz um efeito.
- [x] Teste de log: request com CPF e texto clínico fictícios → nenhuma linha de log contém os valores.
- [x] Erro de validação devolve 422 `application/problem+json`; campo extra devolve 422.
- [x] `Idempotency-Key` repetida devolve a mesma resposta sem criar duplicata.
- [x] Snapshot OpenAPI versionado e contratos compartilhados em `@pep/contracts` consumidos por `@pep/api` e `@pep/ui`.
- [ ] Deploy automático em `staging` e manual por tag em `prod`, com migração antes do tráfego.

## Perguntas abertas

| # | Pergunta | Bloqueia |
| :--- | :--- | :--- |
| Q1 | _(Resolvida: ADR-PEP-0009 aceita em GCP `southamerica-east1`)_ | — |
| Q2 | Tenant = grupo econômico ou CNPJ? (impacta faturamento por clínica e contratos; sal da Spike já fixado por `clinica_id`) | Modelo de tenancy |
| Q3 | Workers no Cloud Run bastam para o volume de lotes XML, ou GKE/Cloud Run Jobs para tarefas longas? | Ingestão de retorno (P2) |

## Notas de implementação

07/11/2026:
- Base inicial criada e em seguida migrada para o monorepo `pnpm` + `Turborepo` (`apps/api`, `apps/ui`, `packages/contracts`, `packages/tsconfig` — ADR-PEP-0018), atualizando o backend para **NestJS 12 (`12.4`)**, **Drizzle ORM** sobre `Db.comTenant` com `AsyncLocalStorage` (ADR-PEP-0017) e contratos **Zod 4** em `@pep/contracts` (`ZodValidationPipe` + geração de schema OpenAPI nativo via `z.toJSONSchema()`).
- **Política RLS:** `current_setting('app.tenant_id', true)::uuid` falha com `invalid input syntax for type uuid: ""` numa conexão reaproveitada, porque o GUC fica `''` (não NULL) depois do COMMIT. A política usa `nullif(..., '')`; sem tenant devolve 0 linhas, como exigido.
- **`SET LOCAL` não aceita parâmetro:** `Db.comTenant` usa `set_config('app.tenant_id', $1, true)` (e `app.usuario_id`, `app.unidade_id`), equivalente e seguro contra SQL injection.
- **Papel `app_admin`** (`BYPASSRLS`) criado em `apps/api/infra/init-roles.sql` para o relay; em GCP, usuário separado no Cloud SQL.
- **Portas locais:** 55442 (Postgres) e 56381 (Redis), para não colidir com outros projetos.
- **Auth P0:** cabeçalho `X-Dev-Tenant` só em `AUTH_MODE=dev` (recusado em produção); OIDC entra na RFC-PEP-001.
- **Pendente nesta RFC:** infra GCP (Terraform, ADR-PEP-0009), deploy `staging`/`prod`, traces/métricas OpenTelemetry, `ETag`/`If-Match` (sem recurso editável ainda), ESLint e `osv-scanner`.

08/11/2026 — migrações e seed passaram para o Drizzle Kit ([[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]): baseline reescrita (`drizzle/0000_baseline.sql`), papéis e grants por `db:bootstrap`, RLS por convenção em `drizzle/rls`, seed em TypeScript com Drizzle. `node-pg-migrate`, `migrations/`, `aplicar_rls()` e `infra/init-roles.sql` foram removidos. A baseline não tem FK de `tenant_id` em `outbox`, `eventos_processados` e `vinculos` (fronteira entre módulos, ADR-PEP-0017); antes havia.
