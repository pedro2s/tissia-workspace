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
  - RFC-014-api-de-operacao-interna
origem: tissia-ai/docs/rfc/RFC-014-api-de-operacao-interna.md
---

# RFC-014 — API de operação interna `/ops/v1`, Fase 1 (SPK-06)

**Fase:** Operação (backoffice B2 do PEP) · **Status:** Concluída (Fase 1 entregue; `GET /avaliacoes` em prod pendente) · **ADRs:** [[ADR-IA-0007-minimizacao-de-dados|0007]], [[ADR-IA-0012-ops-v1-keycloak-jwks|0012]]

## Objetivo

Dar à equipe interna (backoffice do PEP, [[SPK-06-api-de-operacao|SPK-06]]) observabilidade da Tissia AI e a revogação de emergência de chave, sem acesso ao banco e sem texto clínico. Autenticação e regras gerais na [[ADR-IA-0012-ops-v1-keycloak-jwks|ADR-0012]].

## Escopo

| Fase 1 (esta RFC) | Fase 2 (fora) |
| :--- | :--- |
| Autenticação JWKS, `ops_auditoria`, rotas `GET` abaixo e `POST …/revogacao` | `PUT /ops/v1/roteamento` (não existirá enquanto o roteamento for config de deploy), `POST /ops/v1/prompts/{familia}/canario`, `GET /ops/v1/risco/calibracao`, `GET /ops/v1/prompts/{familia}/diff`, `GET /ops/v1/bases/{versao}/diff` |

## Rotas da Fase 1

Todas sob `/ops/v1`, só leitura salvo a revogação. "todos" = qualquer um dos 5 papéis.

| Rota | Papéis | Conteúdo |
| :--- | :--- | :--- |
| `GET /versoes` | todos | `versao_app`, ambiente, `versao_bases`, provedor/decisor/modelos de IA, sha256 dos prompts |
| `GET /saude?janela=1h\|6h\|24h` | todos | Total de requisições, contagem por faixa de status (`2xx`/`4xx`/`5xx`), taxa de `degradado` e p50/p95 por etapa (de `analises.tempos`) |
| `GET /analises?clinica_ref=&degradada=&de=&ate=&cursor=&limite=` | eng_ia, suporte | Metadados paginados por cursor: `analise_id`, `criado_em`, `tipo_guia`, `risco`, `score_risco`, `degradado`, `latencia_ms`, `qtd_feedbacks`, `desfecho_status` |
| `GET /analises/{id}` | eng_ia, suporte | Metadados + `tempos` + alertas (`id`, `tipo`, `severidade`, `fonte`) + feedbacks (`item_id`, `acao`) + desfecho (status, valores, código da Tabela 38) |
| `GET /integradores` | suporte, eng_ia | Integradores (sem hash de chave), status, plano, nº de clínicas, chaves (`id`, `prefixo`, escopos, ambiente, ativa, expiração, último uso) |
| `GET /integradores/{id}/uso?janela=24h` | suporte, eng_ia | Volume por rota e status; `422` agregado só por caminho `loc` |
| `POST /integradores/{id}/chaves/{chave_id}/revogacao` | seguranca | Corpo `{motivo, aprovacao_ref}`; `aprovacao_ref` obrigatório com `AMBIENTE=prod` (422). Desativa a chave; chamadas seguintes ao `/v1` com ela → `401`. Auditoria com `antes`/`depois` |
| `GET /roteamento` | eng_ia | Provedor, decisor, modelos, timeout e kill switches efetivos (só leitura) |
| `GET /prompts` | eng_ia, curadoria | Famílias de prompt do deploy com sha256 e tamanho |
| `GET /avaliacoes` | eng_ia, curadoria | Resultados do harness (`evals/resultados/*.json` do deploy) só com métricas agregadas |
| `GET /bases` | curadoria, dir_medica | `MANIFEST.yaml` efetivo (mesma fonte de `GET /v1/bases/versoes`) |
| `GET /regras?fonte=` | curadoria | Regras carregadas: `id`, `tipo`, `fonte`, `severidade`, nº de casos de teste |
| `GET /feedback/agregado?de=&ate=` | eng_ia, curadoria | Aceite/edição/rejeição por tipo de alerta e por regra (contagens) |

## Modelo de dados

- `ops_auditoria` (migração `0009`): `ator_sub`, `papeis`, `acao`, `alvo_tipo`, `alvo_id`, `aprovacao_ref`, `antes`, `depois`, `request_id`, `criado_em`.
- `ops_requisicoes_hora` (migração `0010`): contagem agregada por hora, integrador, rota (template), status e, para `422`, caminho `loc`. Alimenta `/saude` e `/integradores/{id}/uso`.
- Tempos por etapa: usa a coluna `analises.tempos` já existente (Onda 6).

## Critérios de aceite

- [x] API key de integrador em `/ops/v1/*` → 401; JWT sem o papel → 403; `aud`/`iss`/`exp` inválidos → 401.
- [x] `/v1/openapi.json` sem nenhuma rota `/ops`; com `OPS_HABILITADO=false` o `/ops/v1` responde 404.
- [x] Nenhuma resposta do `/ops/v1` contém texto livre (`justificativa`, `evidencia`, `mensagem`, `motivo_glosa_texto`, `entrada_scrubbed`, `paciente_ref`) — teste recursivo nas chaves.
- [x] Revogação desativa a chave, o `/v1` passa a responder 401 com ela e a ação fica em `ops_auditoria` com `antes`/`depois`.
- [x] Toda chamada autorizada gera linha em `ops_auditoria`.

## Notas de implementação

- **Desvios do desenho técnico da Fase 1 (15/11/2026):** (1) não se cria `analises.tempos_ms`: a coluna `analises.tempos` já existe; (2) faltava fonte para a contagem por status HTTP, por rota e de `422` por `loc` (o `uso_mensal` só conta por mês), daí `ops_requisicoes_hora`; (3) `OPS_JWKS_CACHE_TTL_S` não virou config (5 min fixo no `PyJWKClient`); (4) leituras são auditadas na própria dependência de papel; escritas gravam `antes`/`depois`.
- **Rotas de observabilidade (15/11/2026):** `GET /analises` aceita também `integrador_id` e cada item traz `integrador_id` e `clinica_ref`, porque `clinica_ref` não é única entre integradores. O cursor é base64url de `criado_em|id`, e cursor inválido dá `422`. Os feedbacks trazem também `papel` e `criado_em`, e o desfecho traz `valor_cobrado` e `valor_glosado` (nunca `motivo_glosa_texto`). `/saude` e `/uso` têm granularidade de hora: a janela pode cobrir até 1 h a mais. Percentis e taxa de `degradado` usam o `criado_em` exato.
- **`ops_requisicoes_hora`:** uma linha por hora, integrador, rota (template), status e `loc`. Cada requisição soma 1 na linha com `loc=''`; num `422`, cada caminho de erro soma 1 numa linha própria. A unicidade usa `NULLS NOT DISTINCT` (Postgres 15+; o Cloud SQL é 16). O upsert roda no `MiddlewareLogRequisicao` (cerca de 1 ms, fora da `latencia_ms`); falha vira só log. Conta apenas rotas `/v1/*` casadas: ficam de fora `/ops`, `/healthz`, docs, OpenAPI e 404 de rota inexistente. Os `loc` vêm de um handler de `RequestValidationError` que delega ao padrão do FastAPI, sem mudar a resposta.
- **Rotas de estado do deploy:**
  - `GET /regras` não devolve o nº de casos de teste, porque as regras YAML não têm esse dado.
  - Chaves com a palavra `justificativa` viraram `geracao_justificativa` e `gerar_justificativa` (o teste de texto livre proíbe a chave `justificativa`).
  - `/feedback/agregado` classifica cada item como o tipo do alerta, `geracao_justificativa`, `sugestao_codigo` ou `outro`. `por_regra` usa o id do alerta: nos alertas de DUT, codificação e preditivo esse id é dinâmico (ex.: `dut_<n>_<tuss>_ausente`). O agregado carrega o `analises.resposta` de cada feedback da janela (`ponytail:`); para janelas grandes, agregar em SQL.
  - `GET /bases` devolve o mesmo corpo de `GET /v1/bases/versoes`, via `calcular_versoes_bases()` extraída de `app/api/bases.py`, sem mudar o `/v1`.
- **Pendência — `GET /avaliacoes` vazio em produção:** a rota lê `evals/resultados/*.json`, mas esse diretório está no `.gitignore` e o Dockerfile não copia `evals/`. Para ter dados é preciso versionar um resumo dos resultados ou publicá-los num bucket.
- **Teste de memória da RFC-001:** `test_carregar_performance_e_memoria` passou a medir `carregar()` num subprocesso, pelo `VmHWM`. O RSS do processo do pytest acumulava as apps de outros testes e passou de 300 MB com os testes do `/ops/v1`; o `ru_maxrss` no filho herda o pico do pai.

