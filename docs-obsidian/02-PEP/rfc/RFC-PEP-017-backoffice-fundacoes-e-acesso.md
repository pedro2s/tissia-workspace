---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-017-backoffice-fundacoes-e-acesso.md
---

# RFC-PEP-017 — Backoffice: fundações, acesso interno e trilha de auditoria

**Fase:** B0 (P1, antes do piloto em produção) · **Status:** Implementada (B0 entregue em homologação em 15/11/2026; ver Notas de implementação) · **ADRs:** ADR-PEP-0020, 0021, 0022 (e ADR-PEP-0010 para o Keycloak)
**Telas do backoffice:** 01 (acesso), barra lateral e barra superior, 18 (papéis e trilha, parcial)
**Requisitos:** RF-BKO-01, 02, 03 · RNF-BKO-01…05 ([[PRD-003-backoffice|PRD-003]])
**Depende de:** RFC-PEP-000 (Db, outbox, erros), RFC-PEP-001 (padrão de sessão e guard)

## Objetivo

Deixar o backoffice pronto para receber domínio: processo próprio, login só da equipe com FIDO2, sessão de ambiente com motivo, papéis internos, trilha interna encadeada e a UI ligada à API. Ao final, um operador entra em Homologação ou Produção e tudo o que fizer fica registrado.

## Escopo

| Dentro | Fora |
| :--- | :--- |
| Processo `backoffice` (`src/backoffice.ts`) e prefixo `/bo/v1` | Domínio de clínicas (RFC-PEP-018) |
| Realm `tissia-equipe-<amb>`, cliente `bo-web`, FIDO2 obrigatório | Federação com IdP corporativo (pergunta aberta do PRD-003) |
| Sessão de ambiente: motivo, ticket, 60 min em Produção | Segunda aprovação genérica (entra com a primeira ação que a exigir, RFC-PEP-020) |
| Papéis internos e matriz de permissões | Tela de gestão de papéis (B0 usa grupos do Keycloak) |
| `auditoria_interna` append-only com hash encadeado | Exportação da trilha |
| Papel `app_plataforma` e `Db.comPlataforma` | Tabelas de plataforma novas (RFC-PEP-018) |
| UI: login OIDC, shell com dados reais da sessão | Migração das telas 02–18 para dados reais |

## Design

### Processo e roteamento

- `src/backoffice.ts` cria a aplicação Nest com `BackofficeModule` (controllers `/bo/v1/*`) mais `ComumModule`. O `AppModule` do processo `api` não importa `BackofficeModule`, e isso é verificado no `dependency-cruiser`.
- Porta local 3001. Em `apps/backoffice/vite.config.ts`, proxy `/bo` → `BO_PROXY_TARGET ?? http://localhost:3001`.
- Cloud Run `pep-backoffice-<amb>`, mesma imagem com `CMD node dist/backoffice.js`. Ingresso: IAP com grupo da equipe **[validar custo/complexidade × allowlist de IP]**.

### Identidade

- Realm `tissia-equipe-dev` em `apps/api/infra/keycloak/` (import automático, como o `pep-dev`). Fluxo do navegador: usuário e senha (ou federação) → **WebAuthn obrigatório** (sem OTP como alternativa). Usuário de dev `operador.exemplo` com credencial WebAuthn virtual nos testes.
- Grupos = papéis: `suporte`, `financeiro`, `eng_ia`, `curadoria`, `dir_medica`, `seguranca`. O mapper emite `papeis` no token.
- O guard `AcessoInternoGuard` (global no processo `backoffice`, deny-by-default) valida o emissor do realm da equipe, a audiência `pep-backoffice` e a prova de chave física (`amr`). Tokens do realm `pep-*` recebem 401.

### Sessão de ambiente

```
POST /bo/v1/sessao   { ambiente: "homologacao" | "producao", motivo?: string(10..500), ticket?: string }
→ 201 { token, expira_em, papeis, ambiente }
```

- Em `producao`, `motivo` e `ticket` são obrigatórios (422 sem eles) e o token dura 60 min, sem renovação silenciosa: depois disso é preciso informar motivo de novo. Em `homologacao`, 8 h.
- O ambiente é o do deploy. O campo existe para registrar a intenção e o motivo e para o aviso visual; não troca de banco. Pedido de `producao` em deploy de homologação → 422.
- Claims: `sub`, `papeis`, `ambiente`, `sessao_id`. HS256 com segredo próprio (`BO_SESSAO_SEGREDO`), diferente do segredo do PEP.

### Permissões

Tipos fechados em `@pep/contracts/src/backoffice/permissoes.ts`, função pura `permissoesDoPapel(papel)`:

| Permissão | suporte | financeiro | eng_ia | curadoria | dir_medica | seguranca |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `clinicas:ler` | ✓ | ✓ | ✓ | | ✓ | ✓ |
| `clinicas:onboarding` | ✓ | | | | | |
| `clinicas:suspender` | ✓ (2ª aprov. em prod, RFC-018) | ✓ | | | | |
| `suporte:solicitar` | ✓ | | ✓ | | | |
| `cobranca:*` (B1) | ver | ✓ | | | | |
| `trilha:ler` | | | | | | ✓ |
| `papeis:administrar` | | | | | | ✓ |

Decorator `@Requer('clinicas:ler')` igual ao do PEP. Rota sem decorator → 403.

### Trilha interna

```sql
CREATE TABLE auditoria_interna (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  em timestamptz NOT NULL DEFAULT now(),
  ator_sub text NOT NULL, papeis text[] NOT NULL, sessao_id uuid NOT NULL,
  ambiente text NOT NULL, motivo_id uuid,           -- sessão de produção
  acao text NOT NULL,                                -- 'sessao.iniciada', 'clinica.lida', 'clinica.criada', …
  alvo_tipo text, alvo_id text,                      -- nunca texto livre de cliente
  dados jsonb NOT NULL DEFAULT '{}',                 -- só ids e valores antes/depois de configuração
  hash bytea                                         -- sha256(hash_anterior || linha), calculado pelo worker
);
```

- Sem `tenant_id` (é tabela de plataforma). `app_plataforma` tem só `INSERT` e `SELECT`; `UPDATE` e `DELETE` são revogados e há trigger que rejeita.
- Encadeamento assíncrono pelo `worker` (mesmo desenho da RFC-PEP-014 §auditoria), sem lock na requisição.
- O interceptor `TrilhaInterna` grava **toda** requisição `/bo/v1` (inclusive leituras) com `acao` vinda do decorator `@Acao('clinica.lida')`. Falha ao gravar → 503 (fail-closed, como a leitura de prontuário).

### Banco

- `db:bootstrap` cria `app_plataforma` (NOBYPASSRLS), com senha `APP_PLATAFORMA_PASSWORD`.
- `drizzle/rls/plataforma.txt` lista as tabelas de plataforma. O `db:rls` aplica a política `plataforma_leitura` nelas e **falha** se `app_plataforma` tiver privilégio em tabela fora da lista (ADR-PEP-0022).
- `Db.comPlataforma(ator, fn)` abre transação no pool `app_plataforma` e grava `app.ator_interno`.

### UI (`apps/backoffice`)

- `oidc-client-ts` com realm da equipe (mesmo padrão de `apps/ui/src/auth`, código copiado, sem importar de `@pep/ui`). A tela 01 vira o fluxo real: etapa 1 SSO (redirect), etapa 2 chave (WebAuthn no Keycloak), etapa 3 ambiente, motivo e ticket (`POST /bo/v1/sessao`).
- `Sidebar` e `Topbar` passam a ler sessão, papel e ambiente. Itens do menu sem permissão ficam ocultos. A linha superior e o selo mostram o ambiente da sessão.
- As telas 02–18 continuam estáticas atrás de `RequerSessaoInterna` até a RFC de cada uma.

## Segurança e LGPD

- Nenhuma rota do backoffice devolve dado clínico (ADR-PEP-0022). A trilha não guarda texto de cliente, só ids.
- O CORS do processo `backoffice` aceita só a origem do backoffice. Cabeçalhos `X-Frame-Options: DENY` e CSP restrita.
- Rate limit de 60 req/min por operador **[validar]**.

## Critérios de aceite

- [ ] Token do realm `pep-dev` em `/bo/v1/*` → 401; token do realm da equipe sem chave física → 401.
- [ ] O processo `api` responde 404 em `/bo/v1/sessao`; o processo `backoffice` responde 404 em `/api/v1/eu`.
- [ ] Sessão de `producao` sem motivo → 422; token expira em 60 min.
- [ ] Toda requisição `/bo/v1` gera uma linha em `auditoria_interna`; com a gravação da trilha falhando, a rota responde 503.
- [ ] `app_plataforma` com `SELECT` em tabela clínica → `db:rls` falha (teste).
- [ ] `UPDATE auditoria_interna` → erro.
- [ ] E2E: operador entra com WebAuthn virtual, escolhe Produção com motivo e vê o selo PRODUÇÃO e o nome no shell.

## Perguntas abertas

1. IAP ou allowlist de IP no ingresso? IAP exige Google Workspace na equipe.
2. Operador com dois papéis (ex.: suporte e seguranca): a união das permissões é aceitável?

## Notas de implementação

- `APP_GUARD` (`AcessoGuard`) foi movido de `ComumModule` para `AppModule`, permitindo que `BackofficeModule` importe `ComumModule` usando exclusivamente `AcessoInternoGuard` sem conflito de guards.
- `auditoria_interna` recebeu `GRANT UPDATE (hash) ON auditoria_interna TO app_admin` (mantendo `REVOKE UPDATE, DELETE, TRUNCATE` para `app_rw` e `app_plataforma`) e trigger `BEFORE UPDATE OR DELETE` que só permite preencher `hash` quando `OLD.hash IS NULL`.
- `tissia-equipe-dev-realm.json` utiliza mapper de `amr` (`["pwd","hwk"]`) combinado ao fluxo `bo-browser` com `webauthn-authenticator` obrigatório (sem OTP), seguindo o achado da ADR-PEP-0010 para o Keycloak 26.0.
- `permissoesDosPapeis` implementa a união das permissões quando o operador possui múltiplos papéis, com `cobranca:ler` e `cobranca:escrever` no lugar de `cobranca:*`.
