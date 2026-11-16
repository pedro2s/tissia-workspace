---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0003-postgres-rls-multitenant.md
---

# ADR-PEP-0003 — PostgreSQL 16 com Row-Level Security por tenant

**Status:** Aceita — 07/11/2026

## Contexto

O PEP é multi-clínica: um tenant (grupo/clínica contratante) tem várias unidades, e um usuário pode atuar em mais de um tenant (tela 01b, "Seu perfil tem acesso a 2 clínicas"). Vazamento de prontuário entre clínicas é o pior incidente possível (LGPD Art. 11). Filtro de tenant só no código da aplicação falha no primeiro `WHERE` esquecido.

## Decisão

- **PostgreSQL 16**, banco único, schema por módulo ([[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]]).
- **`tenant_id uuid NOT NULL` em toda tabela de dado de cliente** (incluindo tabelas filhas), com política RLS `USING (tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid)` e `WITH CHECK` igual (`nullif` evita falha de cast quando o GUC volta para `''` após `COMMIT` em conexão reaproveitada do pool). Tabelas globais (`usuarios`, TUSS, CID, catálogo de operadoras) não têm `tenant_id` e são só leitura para a aplicação; tabelas mistas com catálogo curado (`modelos` com `tenant_id IS NULL`) usam política `SELECT` explícita permitindo `tenant_id IS NULL` e escrita restrita ao tenant da sessão.
- **Contexto por transação:** o backend abre transação (`Db.comTenant` com Drizzle ORM + `AsyncLocalStorage`, ver [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]]) e executa `SELECT set_config('app.tenant_id', $1, true)` (e `app.usuario_id`, `app.unidade_id` para auditoria) a partir do token. Nenhuma query fora de transação com contexto.
- **Papéis:** a aplicação conecta com um papel **sem `BYPASSRLS`** e sem ser dono das tabelas. Migrações rodam com outro papel. Jobs de manutenção cross-tenant usam um papel separado, auditado.
- `FORCE ROW LEVEL SECURITY` em todas as tabelas com `tenant_id`.
- Isolamento por unidade (ex.: faturista só da Unidade Centro) é regra de autorização na aplicação, não RLS.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Banco por tenant | Migrações e conexões multiplicam; difícil para BI e para usuário em 2 tenants |
| Schema por tenant | Mesmo problema de migração, sem ganho de isolamento real sobre RLS |
| Só filtro na aplicação | Um bug vaza prontuário; RLS é a segunda barreira |

## Consequências

- **Testes de isolamento obrigatórios:** para cada tabela com `tenant_id`, um teste tenta ler e escrever com outro tenant e espera zero linhas/erro; um teste falha se aparecer tabela com `tenant_id` sem política ([[test-plan-pep|test plan]]).
- Pool de conexões precisa resetar o contexto (`SET LOCAL` resolve, por ser escopo de transação).
- Índices começam por `tenant_id` nas tabelas grandes (atendimentos, guias, auditoria).
- Modelo detalhado em [[modelo-de-dados-pep|modelo de dados]].
