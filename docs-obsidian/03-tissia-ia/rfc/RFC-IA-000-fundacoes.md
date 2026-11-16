---
projeto: tissia-ia
tipo: rfc
status: concluida
tags:
  - tissia
  - tissia-ia
  - rfc
  - especificacao
aliases:
  - RFC-000-fundacoes
origem: tissia-ai/docs/rfc/RFC-000-fundacoes.md
---

# RFC-000 — Fundações (repo, infra, auth, PII, persistência)

**Fase:** F0 · **Status:** Concluída · **ADRs:** 0002, 0003, 0005, 0007

## Objetivo

Deixar um serviço FastAPI vazio, mas de produção: deployável em `staging`/`prod`, autenticado, sem PII em logs, com banco e CI. Todas as RFCs seguintes só adicionam rotas e módulos.

## Estrutura do repositório

```
tissia-ai/
  CLAUDE.md
  pyproject.toml          # uv, ruff, pytest
  Dockerfile
  app/
    main.py               # FastAPI app, routers, /healthz
    config.py             # pydantic-settings (env vars)
    auth.py               # dependency: valida API key → integrador
    db.py                 # engine SQLAlchemy, sessão
    models.py             # tabelas
    pii.py                # scrubber (ADR-0007)
    logging.py            # logs JSON sem corpo de requisição
    bases.py              # carrega data/ em memória (RFC-001)
    regras.py             # motor de regras (RFC-003)
    dut.py                # validador DUT + justificativa (RFC-004)
    codificacao.py        # sugestão TUSS/CID (RFC-006)
    ia/
      decisor.py          # decidir(state, perguntas) → backends jev | gemini
      gemini.py
      jev.py
    api/
      guias.py            # POST /v1/guias/analisar (RFC-002)
      analises.py         # feedback (RFC-002)
  migrations/             # Alembic
  data/                   # bases versionadas (RFC-001)
  scripts/                # ingestão ANS/DATASUS, criação de API key
  evals/                  # golden sets + runner (RFC-005)
  infra/bootstrap.sh      # gcloud idempotente (ADR-0003)
  tests/
  docs/
```

Plano de propósito: um módulo por responsabilidade, sem camadas `services/`/`repositories/` até que um módulo passe de ~400 linhas.

## Configuração (`app/config.py`)

| Variável | Exemplo | Nota |
| :--- | :--- | :--- |
| `DATABASE_URL` | `postgresql+psycopg://…` | Secret Manager em `staging`/`prod` |
| `GCP_PROJECT`, `VERTEX_LOCATION` | `medscribe-prod`, `global` | Padrão `global` desde a [[ADR-IA-0010-vertex-global-e-gemini-3x\|ADR-IA-0010]] (ou região Vertex); a validação falha no startup se vazio |
| `GEMINI_MODEL` | `gemini-<versão>-flash` | **[validar]** modelo com residência no Brasil |
| `DECISOR_BACKEND` | `gemini` \| `jev` | Default `gemini` até os gates da ADR-0004 |
| `JEV_API_URL`, `JEV_API_KEY` | — | Opcional |
| `IA_TIMEOUT_S` | `4.0` | Orçamento total de IA por requisição |

## Modelo de dados (Postgres)

```sql
integradores (id uuid pk, nome text, api_key_hash text unique, ativo bool, criado_em timestamptz)

clinicas (id uuid pk, integrador_id fk, clinica_ref text,      -- id da clínica no sistema do integrador
          nome text, flags jsonb not null default '{}',        -- ver release plan
          criado_em timestamptz, unique(integrador_id, clinica_ref))

analises (id uuid pk, clinica_id fk, criado_em timestamptz,
          guia_ref text null,                                  -- id opaco da guia no PEP (rastreia revisões e desfechos)
          modo text not null default 'visivel',                -- 'sombra' | 'visivel'
          tipo_guia text, registro_ans text, especialidade text,
          entrada_hash text,                                    -- sha256 do texto da evolução
          entrada_scrubbed text null,                           -- só com flag opt-in (ADR-0007)
          resposta jsonb not null,                              -- sempre guarda a resposta completa pré-máscara sombra
          versao_bases text, decisor text, modelo_geracao text,
          latencia_ms int, degradado bool)

feedbacks (id uuid pk, analise_id fk, item_id text,             -- id do alerta/sugestão/procedimento
           acao text check (acao in ('aceito','editado','rejeitado')),
           texto_final text null, papel text check (papel in ('faturista','medico')),
           criado_em timestamptz)
```

Clínica desconhecida no `POST /v1/guias/analisar` é criada automaticamente com as flags default (auto-provisionamento por integrador). `# ponytail:` sem painel de admin. As flags são alteradas por SQL/script até a F2.

## Autenticação

- `Authorization: Bearer tsk_live_<32 bytes base62>` (ou `tsk_test_<32 bytes base62>` para sandbox). O banco guarda `sha256(chave)`.
- `scripts/criar_integrador.py <nome>` imprime a chave uma única vez.
- Rate limit: não na F1 (um integrador). `# ponytail:` adicionar quando houver um segundo integrador ou a extensão (F5).

## Logs e erros

- Log JSON por requisição: `request_id`, `integrador_id`, `clinica_id`, rota, status, `latencia_ms`, `analise_id`. O middleware **não** loga o corpo.
- Exceções: o handler global loga tipo + stack sem os argumentos/locals. Se usar Sentry, `send_default_pii=False` e `include_local_variables=False`.

## CI/CD

- PR: `ruff check`, `ruff format --check`, `pytest` (Postgres como service container), build da imagem.
- `main` → deploy automático em `staging` (Cloud Run) + `alembic upgrade head` via job.
- Tag `v*` → deploy em `prod`, com aprovação manual no GitHub Environment.

## Critérios de aceite

- [x] `GET /healthz` em `staging` e `prod` responde 200 com `versao_app` e `versao_bases`.
- [x] Requisição sem chave ou com chave inválida → 401. Chave de integrador inativo → 403.
- [x] Teste prova que um corpo com um CPF fictício não aparece em nenhuma linha de log capturada.
- [x] Testes do scrubber cobrem CPF (com/sem máscara), CNS, RG, CRM, telefone, e-mail, CEP, data de nascimento (`[DATA_NASCIMENTO]`) e data clínica convertida em tempo relativo (`[DATA: há ~N dias]`), sem mascarar códigos TUSS/CID.
- [x] `VERTEX_LOCATION` aceita `global` como padrão ([[ADR-IA-0010-vertex-global-e-gemini-3x|ADR-IA-0010]]) e impede o startup quando vazio.
- [x] Deploy por tag em `prod` exige aprovação.
