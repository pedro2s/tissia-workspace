---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0022-backoffice-sem-dado-clinico.md
---

# ADR-PEP-0022 — Backoffice sem dado clínico (papel de banco próprio) e suporte só por acesso consentido no PEP

**Status:** Aceita — 08/11/2026
**Relacionadas:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]]

## Contexto

O backoffice precisa ver **todos** os tenants (carteira, onboarding, uso), portanto lê dados de tenants diferentes. Hoje o único papel que faz isso é o `app_admin` (`BYPASSRLS`), que enxerga tudo, inclusive prontuário. A equipe de suporte também precisa, às vezes, entrar numa clínica para resolver um problema. O design (telas 04 e 18) promete: "o prontuário nunca é exibido no backoffice; o acesso de suporte exige aprovação da clínica, tem prazo e fica na trilha das duas partes".

## Decisão

1. **Papel `app_plataforma`** (`NOBYPASSRLS`), criado no `db:bootstrap`:
   - `GRANT SELECT, INSERT, UPDATE` apenas nas tabelas de plataforma: `tenants`, `clinicas`, `unidades`, `convites`, `acessos_suporte`, `assinaturas` (B1) e as tabelas de uso agregado.
   - Política RLS adicional `plataforma_leitura` nessas tabelas, `TO app_plataforma USING (true)`, gerada pela convenção de `drizzle/rls`. Tabelas de plataforma são as listadas em `drizzle/rls/plataforma.txt`.
   - **Nenhum grant** em tabelas clínicas (pacientes, prontuário, guias, documentos, auditoria do tenant). O `db:rls` falha se `app_plataforma` tiver privilégio em tabela fora da lista. É uma garantia estrutural: um bug no código do backoffice não consegue ler prontuário.
2. **`Db.comPlataforma(ator, fn)`**: transação no pool `app_plataforma` que grava `app.ator_interno`. É o único acesso a dados do módulo `backoffice`; `DbAdmin` continua restrito a outbox e jobs.
3. **Uso agregado sem RLS de tenant:** um job do `worker` (`app_admin`, auditado) consolida contagens diárias por tenant (análises, aceite, médicos ativos) em `uso_diario`, que é tabela de plataforma. O backoffice lê só esse agregado.
4. **Suporte entra pelo PEP, não pelo backoffice:**
   - O backoffice cria `acessos_suporte` (tenant, escopo `configuracoes | faturamento_metadados`, duração ≤ 4 h, motivo, solicitante) com estado `pendente`.
   - O administrador da clínica aprova ou recusa no PEP (tela 25). A aprovação cria um vínculo temporário do usuário interno com o perfil `suporte_<escopo>` e `expira_em`, no realm do PEP, onde o usuário interno tem uma conta espelho sem senha, só WebAuthn.
   - Os escopos não incluem leitura de prontuário. Quebra-vidro (RF-ACE-10) não vale para usuário interno.
   - Cada ação do suporte vai para a trilha do tenant (RFC-PEP-014) **e** para a trilha interna do backoffice.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Backoffice usa `app_admin` | Enxerga prontuário; a garantia ficaria só no código |
| Backoffice abre `Db.comTenant` em cada tenant | Para listar 48 clínicas abriria 48 transações, e o papel `app_rw` dá acesso a dado clínico do tenant |
| Banco ou schema separado para plataforma | Quebraria FKs e transações com `tenants`/`unidades`; é complexidade sem ganho nesta escala |
| Suporte com "personificar usuário" | O suporte agiria como o usuário: a trilha mente e o consentimento fica implícito |

## Consequências

- Mais um papel, mais um pool e uma checagem nova no `db:rls`.
- Metadados de plataforma (nome da clínica, CNPJ, e-mail do administrador) ficam visíveis à equipe interna. São dados da pessoa jurídica e do contato contratual, não de paciente; precisam constar no RIPD.
- O acesso de suporte depende da RFC-PEP-014 (tela 25 e trilha do tenant). Até lá, suporte em produção não tem caminho e só usa homologação.
