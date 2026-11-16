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
  - RFC-006-sugestao-tuss-cid
origem: tissia-ai/docs/rfc/RFC-006-sugestao-tuss-cid.md
---

# RFC-006 — Sugestão TUSS/CID

**Fase:** F1 · **Status:** Concluída · **ADRs:** 0004, 0006, 0008

## Objetivo

A partir da evolução clínica, sugerir os CIDs e os procedimentos TUSS correspondentes, com confiança e trecho de evidência, e cruzar com o que a guia já tem:

- `ausente_na_guia`: citado na evolução, não lançado → alerta `procedimento_nao_lancado` (receita perdida) ou `codigo_divergente` (CID).
- `diverge_do_informado`: a guia tem um código, mas a evolução aponta outro da mesma família → alerta `codigo_divergente`, `aviso`.
- `ja_informado`: confirma o que está na guia.

Funciona com a guia vazia (`cids: []`, `procedimentos: []`): devolve só sugestões, sem alertas de divergência.

**Invariante:** nenhum código fora das bases vigentes é sugerido. A escolha final é sempre entre candidatos tirados das bases.

## Pipeline (`app/codificacao.py`)

```
1. Extração (Gemini, saída estruturada)
   evolução → { diagnosticos: [{termo, evidencia}], procedimentos: [{termo, tipo: realizado|solicitado, lateralidade?, evidencia}] }
   Termos normalizados em linguagem de tabela ("ressonância magnética de joelho", "entorse de LCA").
   Ignorar procedimento negado/cancelado ("não será necessário RM").

2. Recuperação de candidatos (em memória, ADR-0006)
   para cada termo: top 50 de bases.indice_tuss / bases.indice_cid
   filtro: só códigos vigentes; TUSS restrito a capítulos/grupos compatíveis com a especialidade quando o mapa existir

3. Escolha (decisor, uma chamada para tudo)
   para cada termo: choice(termo + evidência, opções = candidatos + "nenhum_dos_anteriores")
   → top-3 com probabilidade

4. Cruzamento com a guia → situacao + alertas
```

- Extração e escolha são as únicas chamadas de IA: 1 Gemini + 1 decisor, em paralelo com o validador DUT (RFC-002).
- `confianca < 0,5` ou `nenhum_dos_anteriores` vencendo → o termo não gera sugestão (e não gera alerta).
- Só a 1ª opção vira `sugestoes_codigos[]`. As opções 2 e 3 vão em `alternativas` dentro do item, para a UI oferecer troca.

## Índice lexical (`IndiceLexical` em `app/bases.py`)

- Normalização: minúsculas, sem acento, sem stopwords pt, expansão de siglas por dicionário curado (`data/siglas.yaml`: `rm` → `ressonancia magnetica`, `tc` → `tomografia computadorizada`, `us`/`usg` → `ultrassonografia`, `lca` → `ligamento cruzado anterior`…).
- Ranking híbrido: ponderação **BM25 / IDF** sobre os tokens normalizados (para dar peso alto a termos clínicos específicos e evitar empates de subconjunto em `100.0`) combinada com `rapidfuzz` (`WRatio` / `token_sort_ratio`) como score de similaridade e desempate, retornando `limit=50`.
- `# ponytail:` lexical (BM25 + rapidfuzz) + siglas. Upgrade para embeddings (Vertex text-embedding em SP) + pgvector se a recall@50 no golden set ficar abaixo de 95% (ADR-0006).

## Mudanças no contrato

Já refletidas na [[RFC-IA-002-api-analise-de-guia|RFC-002]]: `opcoes.sugerir_codigos`, `sugestoes_codigos`, alertas `procedimento_nao_lancado` e `codigo_divergente`, `item_id` de feedback `sug_cid:<codigo>`/`sug_tuss:<codigo>`.

Cada item de sugestão:

```json
{ "codigo_tuss": "…", "termo": "…", "confianca": 0.74, "evidencia": "solicitada RM do joelho direito",
  "situacao": "ausente_na_guia", "alternativas": [ { "codigo_tuss": "…", "termo": "…", "confianca": 0.12 } ] }
```

## Relação com outras regras

- Se o motor de regras já emitiu `acessorio_ausente` para um código, a sugestão desse mesmo código não gera um segundo alerta (deduplicação por código).
- Procedimento sugerido com DUT curada **não** dispara validação DUT na mesma análise (só procedimentos que estão na guia são validados). `# ponytail:` evita uma segunda rodada de decisor. Revisitar se a faturista pedir.

## Critérios de aceite

- [x] Teste de propriedade: com qualquer resposta dos stubs de IA, todo código devolvido existe em `bases.tuss`/`bases.cid` e está vigente.
- [x] Golden set: top-3 TUSS ≥ 80%, top-3 CID ≥ 80%, recall@50 de candidatos ≥ 95% **[metas a revisar após o primeiro baseline]**.
- [x] Negação ("não há necessidade de RM") não gera sugestão de RM em 100% dos casos negativos do golden set.
- [x] Guia vazia → só sugestões, nenhum alerta de divergência.
- [x] Timeout da extração → `sugestoes_codigos: null`, `degradado: true`, restante da resposta intacto.
