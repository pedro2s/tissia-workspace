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
  - RFC-002-api-analise-de-guia
origem: tissia-ai/docs/rfc/RFC-002-api-analise-de-guia.md
---

# RFC-002 — API de análise de guia

**Fase:** F1 · **Status:** Concluída · **ADRs:** 0005, 0007, 0008

## Objetivo

Contrato público da F1. Uma chamada por guia devolve tudo o que o PEP mostra à faturista.

## `POST /v1/guias/analisar`

### Request

```json
{
  "clinica_ref": "clinica-123",
  "guia_ref": "guia-pep-9876",
  "tipo_guia": "sp_sadt",
  "operadora": { "registro_ans": "000000" },
  "especialidade": "ortopedia",
  "paciente": { "ref": "opaco-abc", "sexo": "M", "idade": 34 },
  "evolucao_clinica": "Paciente refere entorse em joelho D jogando futebol há 5 dias…",
  "cids": ["S835"],
  "procedimentos": [
    { "codigo_tuss": "41101227", "quantidade": 1 }
  ],
  "indicacao_clinica": null,
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

| Campo | Regra |
| :--- | :--- |
| `guia_ref` | Opcional. Identificador opaco da guia no PEP; permite rastrear revisões (autocorreção antes do envio), idempotência e vínculo com desfechos em lote na F2 |
| `tipo_guia` | `consulta` \| `sp_sadt` na F1 (`internacao` na F2+) |
| `paciente.sexo` | `M` \| `F` \| `I` (indeterminado) |
| `evolucao_clinica` | 1–20.000 caracteres. Obrigatório para DUT, justificativa e sugestão; sem ele, só as regras rodam e a resposta traz um alerta `info` |
| `cids`, `procedimentos` | Podem ser vazios (a sugestão de códigos preenche a lacuna) |
| `indicacao_clinica` | Texto que a faturista/médico já escreveu, se houver. É avaliado contra a DUT junto com a evolução |
| Campos extras | **Rejeitados (422)**: `extra="forbid"`, para que ninguém mande nome/CPF por engano (ADR-0007) |

### Response 200

```json
{
  "analise_id": "0b4c…",
  "versao_bases": "a1b2c3d4e5f6",
  "risco": "alto",
  "degradado": false,
  "alertas": [
    {
      "id": "al_1",
      "tipo": "dut_criterio_ausente",
      "severidade": "bloqueante",
      "procedimento_tuss": "41101227",
      "mensagem": "Para o TUSS 41101227, a DUT nº X exige … Confirme se consta no atendimento.",
      "fonte": { "tipo": "ANS_DUT", "referencia": "RN 465/2021, Anexo II, DUT nº X" },
      "confianca": 0.86
    }
  ],
  "procedimentos": [
    {
      "codigo_tuss": "41101227",
      "termo_tuss": "…",
      "dut": {
        "numero": 0,
        "conformidade": "nao_conforme",
        "criterios": [
          { "id": "c1", "texto": "…", "status": "presente", "probabilidade": 0.93, "evidencia": "Teste de Lachman positivo" }
        ]
      },
      "justificativa_sugerida": {
        "texto": "Paciente com trauma torsional agudo de joelho direito há 5 dias…",
        "gerada_por_ia": true
      }
    }
  ],
  "justificativa_sugerida": {
    "texto": "Paciente com trauma torsional agudo de joelho direito há 5 dias…",
    "procedimentos_cobertos": ["41101227"],
    "gerada_por_ia": true
  },
  "sugestoes_codigos": {
    "cids": [
      { "codigo": "S835", "descricao": "…", "confianca": 0.81, "evidencia": "entorse… LCA", "situacao": "ja_informado" }
    ],
    "procedimentos": [
      { "codigo_tuss": "…", "termo": "…", "confianca": 0.74, "evidencia": "solicitada RM do joelho direito", "situacao": "ausente_na_guia" }
    ]
  },
  "aviso_legal": "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."
}
```

Enumerações:

- `alertas[].tipo`: `campo_obrigatorio`, `cid_incompativel_sexo`, `incompativel_idade_sexo`, `tuss_invalido`, `acessorio_ausente`, `procedimento_excludente`, `quantidade_invalida`, `exige_autorizacao_previa`, `dut_criterio_ausente`, `dut_criterio_incerto`, `codigo_divergente`, `procedimento_nao_lancado`, `evolucao_ausente`.
- `severidade`: `bloqueante` (vai glosar se não corrigir) \| `aviso` (risco provável) \| `info`.
- `dut.conformidade`: `conforme` \| `nao_conforme` \| `incerto` \| `nao_aplicavel` (sem DUT curada).
- `criterios[].status`: `presente` (p ≥ 0,7) \| `ausente` (p ≤ 0,3) \| `incerto`. Limiares em config, calibrados no eval.
- `sugestoes_codigos.*.situacao`: `ja_informado` \| `ausente_na_guia` \| `diverge_do_informado`.
- `risco`: o pior entre os alertas (`bloqueante` → `alto`, `aviso` → `medio`, senão `baixo`). `# ponytail:` heurística. O score preditivo vem na F3.
- **Modo sombra (`clinicas.flags.modo == "sombra"`):** o motor executa normalmente e grava a resposta completa em `analises.resposta` (com `analises.modo = "sombra"`), mas a resposta HTTP volta com `alertas: []`, `sugestoes_codigos: null` e `justificativa_sugerida: null` (na raiz e em `procedimentos[]`).

### Orquestração e orçamento de latência

```
validar request → scrub(evolucao) → regras (sync, ~ms)
                                   ├─ dut.avaliar()            [decisor: 1 chamada p/ guia]
                                   ├─ codificacao.sugerir()    [Gemini extrai → índice BM25+fuzz → decisor] se sugerir_codigos
                                   └─ (após dut) justificativa [Gemini 1x p/ guia + fast-path lexical / decisor] se gerar_justificativa
→ montar resposta → gravar análise (completa) → aplicar máscara sombra (se modo=sombra) → 200
```

- `dut.avaliar` e `codificacao.sugerir` rodam em paralelo (`asyncio.gather`). A justificativa depende do resultado da DUT (para não afirmar critério ausente) e é gerada em **uma única chamada ao Gemini para toda a guia** (consolidada para o Campo 23 da SP/SADT).
- Orçamento total de IA: `IA_TIMEOUT_S` (default 4 s), com sub-deadlines escalonados (ex.: ~2,2 s para DUT/extração, ~1,8 s restante para justificativa) para que um atraso na geração nunca descarte os resultados já prontos da DUT. Ramo que estoura → resultado parcial + `degradado: true`. As regras determinísticas sempre voltam.
- Erro de IA nunca vira 5xx. 5xx só por falha nossa (ex.: banco fora; nesse caso a análise é devolvida mesmo assim e a gravação é logada como erro). `# ponytail:` sem retry da gravação.

### Erros

| Status | Quando |
| :--- | :--- |
| 401/403 | Chave ausente, inválida ou inativa |
| 422 | Schema inválido ou campo extra (corpo padrão FastAPI `{"detail": [...]}`) |
| 413 | Corpo > 64 KB |

## `POST /v1/analises/{analise_id}/feedback`

```json
{
  "papel": "faturista",
  "itens": [
    { "item_id": "al_1", "acao": "aceito" },
    { "item_id": "41101227:justificativa", "acao": "editado", "texto_final": "…" },
    { "item_id": "sug_tuss:…", "acao": "rejeitado" }
  ]
}
```

`item_id` = `alertas[].id`, `<codigo_tuss>:justificativa`, `sug_cid:<codigo>` ou `sug_tuss:<codigo>`. 204 em caso de sucesso; 404 se a análise não for de uma clínica do integrador. `texto_final` passa pelo scrubber antes de ser gravado.

## Documentação para o integrador

- OpenAPI gerado em `/v1/openapi.json`, com exemplos.
- `docs/integracao.md` (escrito no fim da F1): fluxo recomendado (chamar ao salvar a guia, timeout de 6 s no cliente, exibir `aviso_legal`, justificativa editável, enviar feedback).

## Critérios de aceite

- [x] Snapshot do `openapi.json` versionado. Mudança no snapshot exige revisão (teste de contrato).
- [x] Com Jev/Gemini simulados por stubs, a resposta respeita o schema em todos os cenários de degradação (DUT timeout, justificativa timeout, sugestão timeout, todos).
- [x] Campo `nome` ou `cpf` no corpo → 422.
- [x] p95 ≤ 1,5 s sem justificativa/sugestão e ≤ 5 s com ambos, em `staging`, com 5 req/s por 5 min.
- [x] Toda análise gravada tem `versao_bases`, `decisor`, `modelo_geracao` e `latencia_ms`.
