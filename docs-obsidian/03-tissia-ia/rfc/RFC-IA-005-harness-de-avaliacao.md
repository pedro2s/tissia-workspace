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
  - RFC-005-harness-de-avaliacao
origem: tissia-ai/docs/rfc/RFC-005-harness-de-avaliacao.md
---

# RFC-005 — Harness de avaliação de modelos

**Fase:** F0 (usado em toda fase seguinte) · **Status:** Concluída · **ADRs:** 0004, 0007

## Objetivo

Decidir com números (e não por opinião) qual backend de decisão e qual modelo de geração usar. Depois, detectar regressão quando muda o prompt, o modelo ou a base.

## Golden sets

`evals/casos/*.jsonl`, um caso por linha:

```json
{
  "id": "orto-0001",
  "origem": "sintetico",
  "autor": "CMO",
  "entrada": { "...": "mesmo schema do request de /v1/guias/analisar" },
  "esperado": {
    "criterios": { "DUT-X:c1": "presente", "DUT-X:c2": "ausente" },
    "alertas_tipos": ["dut_criterio_ausente"],
    "cids": ["S835"],
    "procedimentos_tuss": ["…"],
    "acessorios": []
  }
}
```

| Conjunto | Origem | Tamanho alvo | Quando |
| :--- | :--- | :--- | :--- |
| `sinteticos` | Escritos pelo CMO (e rascunhados por LLM, revisados por ele) cobrindo as DUTs e acessórios das especialidades-piloto, com casos negativos e ambíguos | 60 | F0 |
| `piloto` | Guias reais do piloto, já com scrubber, rotuladas pelo CMO/consultor | 200 | F2 |

Casos com `origem != "sintetico"` vivem **fora do git**, em um bucket GCS em `southamerica-east1` com acesso restrito. O runner os baixa em tempo de execução.

## Runner

```
uv run python -m evals.run --conjunto sinteticos --decisor jev --gerador gemini-x-flash
uv run python -m evals.run --conjunto sinteticos --decisor openrouter:<modelo>
```

- Chama os módulos internos (`dut`, `codificacao`, `regras`) direto, sem HTTP.
- **Guarda:** se o provedor for `openrouter:*` e algum caso tiver `origem != "sintetico"`, aborta antes da primeira chamada.
- Saída: `evals/resultados/<data>-<config>.json` + uma tabela no terminal.

## Métricas

| Métrica | Definição |
| :--- | :--- |
| Recall de critério ausente | critérios esperados `ausente` que o sistema marcou `ausente` ou `incerto` |
| Precisão de alerta | alertas emitidos que constam em `alertas_tipos` esperados |
| Taxa de incerto | % de critérios `incerto` (alto demais = inútil) |
| Calibração | Brier score dos critérios (só faz sentido no Jev; reportar para ambos) |
| Top-1 / Top-3 de código | por TUSS e CID, contra `procedimentos_tuss`/`cids` esperados |
| Recall@50 de candidatos | o código esperado está entre os candidatos recuperados antes do decisor (RFC-006) |
| Fato não suportado | justificativas reprovadas pela checagem automática + amostra humana |
| Latência p50/p95 e custo/guia | por config |

## Uso

- **Gate da ADR-0004 (fim da F0):** rodar `sinteticos` com {jev, gemini} × gerador gemini e, como referência, 2–3 modelos via OpenRouter. Publicar a tabela comparativa em `docs/evals/AAAA-MM-DD-decisor.md` e atualizar o status da ADR.
- **Regressão:** a CI roda `sinteticos` (com chamadas reais, ~60 casos) em PR que toque `app/prompts/`, `app/ia/`, `data/dut/`, `data/regras/` ou a config de modelo. Falha se alguma métrica-alvo do PRD cair mais de 3 p.p. em relação ao último resultado em `main`.
- Limiares (0,7/0,3) são ajustados aqui e copiados para a config, com nota do resultado que os justificou.

## Critérios de aceite

- [x] 60 casos sintéticos revisados pelo CMO, cobrindo cada DUT curada com pelo menos 1 caso conforme, 1 não conforme e 1 ambíguo.
- [x] A guarda do OpenRouter tem teste.
- [x] Tabela comparativa publicada e ADR-0004 atualizada.
