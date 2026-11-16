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
  - RFC-001-bases-de-conhecimento
origem: tissia-ai/docs/rfc/RFC-001-bases-de-conhecimento.md
---

# RFC-001 — Bases de conhecimento: TUSS, CID-10, Correlação TUSS×Rol, DUT

**Fase:** F0 · **Status:** Concluída · **ADRs:** 0006, 0009

## Objetivo

Ter em `data/` bases oficiais normalizadas, versionadas e rastreáveis até a fonte, carregadas em memória por `app/bases.py`.

## Fontes

| Base | Fonte oficial | Script | Saída |
| :--- | :--- | :--- | :--- |
| TUSS 22 — Procedimentos | ANS, Padrão TISS › Componente de Representação de Conceitos em Saúde (planilhas da Terminologia TUSS) | `scripts/ingest_tuss.py` | `data/tuss/22_procedimentos.csv` (`codigo, termo, inicio_vigencia, fim_vigencia`) |
| TUSS — Mensagens/glosas | Mesma fonte (tabela de mensagens; nº **[validar]**) | idem | `data/tuss/mensagens.csv` (usada na F2) |
| Correlação TUSS × Rol | ANS, página do Rol de Procedimentos (planilha de correlação) | `scripts/ingest_rol.py` | `data/rol/correlacao.csv` (`codigo_tuss, procedimento_rol, capitulo, grupo, subgrupo, dut`) |
| CID-10 | DATASUS (CID10CSV: capítulos, categorias, subcategorias) | `scripts/ingest_cid10.py` | `data/cid10/subcategorias.csv` (`codigo, descricao, restricao_sexo`) |
| DUT (critérios) | ANS, Anexo II da RN nº 465/2021 + RNs de atualização | Curadoria assistida (abaixo) | `data/dut/<numero>.yaml` |

Os scripts recebem o arquivo oficial baixado como argumento (sem download automático na F0), normalizam encoding (os CSVs do DATASUS vêm em latin-1), datas e códigos (sem pontos, zero-padding), e atualizam `data/MANIFEST.yaml`:

```yaml
tuss_22:
  fonte: "ANS - Terminologia TUSS"
  url: "<url do arquivo>"
  publicado_em: "2026-xx-xx"
  sha256_original: "…"
  arquivo: tuss/22_procedimentos.csv
```

`versao_bases` = os 12 primeiros caracteres do sha256 do `MANIFEST.yaml`.

## DUT: formato e curadoria

As DUTs vêm em PDF, com texto jurídico-clínico. Para cada DUT usada pelas especialidades-piloto (Q3 do PRD; default Ortopedia + Cardiologia):

1. `scripts/extrair_dut.py <numero> <trecho.txt>` usa o Gemini para propor critérios estruturados (só como rascunho).
2. O CMO revisa e corrige o YAML no PR. **Nenhuma DUT entra sem `revisado_por`.**

```yaml
numero: 0                       # nº da DUT no Anexo II
titulo: "…"
fonte: "RN 465/2021, Anexo II, item X; atualizada pela RN …"   # [validar]
codigos_tuss: ["…"]             # redundante com a correlação; o teste verifica se bate
logica: qualquer                # qualquer | todos — como os grupos se combinam
grupos:
  - id: g1
    descricao: "Cobertura obrigatória quando preenchido pelo menos um dos critérios:"
    logica: qualquer
    criterios:
      - id: c1
        texto: "…"              # texto fiel à DUT
        pergunta: "A evolução clínica documenta …?"   # pergunta noul enviada ao decisor (opcional se condicao_estruturada bastar)
        condicao_estruturada:   # opcional: avaliada deterministicamente antes do decisor
          idade_min: null
          idade_max: null
          sexo: null            # M | F
          cids_permitidos: null # ex.: ["S835", "M17"]
revisado_por: "CRM-XX 00000"
revisado_em: "2026-11-xx"
```

A estrutura `grupos` + `logica` cobre o formato típico das DUTs ("pelo menos um dos critérios do grupo I **e** …"). `# ponytail:` dois níveis de aninhamento. Se uma DUT exigir mais, achata em grupos ou adiciona um nível, nunca uma árvore genérica.

## `app/bases.py`

```python
@dataclass(frozen=True)
class Bases:
    versao: str
    tuss: dict[str, ProcedimentoTuss]  # codigo -> termo, vigência
    cid: dict[str, Cid]  # codigo -> descricao, restricao_sexo
    dut_por_tuss: dict[str, Dut]  # codigo_tuss -> Dut (só DUTs curadas)
    indice_tuss: IndiceLexical  # usado pela RFC-006
    indice_cid: IndiceLexical


def carregar(dir: Path = Path("data")) -> Bases: ...
```

Carregado uma vez no startup (`lifespan`). Atualizar a base = novo deploy.

## Critérios de aceite

- [x] Os scripts são idempotentes: rodar duas vezes sobre o mesmo arquivo não gera diff.
- [x] Teste: todo `codigos_tuss` de cada DUT YAML existe na TUSS 22 vigente e aponta para a mesma DUT na correlação.
- [x] Teste: todo YAML de DUT valida contra o schema Pydantic e tem `revisado_por`.
- [x] `carregar()` em < 3 s e < 300 MB RSS no Cloud Run.
- [x] `/healthz` expõe `versao_bases`.

## Perguntas abertas

- Q2 (versão TISS/operadoras predominantes) define se precisamos manter mais de uma versão da TUSS em paralelo. Default: só a vigente.
- O exemplo do PRD v2 (TUSS 41101227, RM de joelho, com exigência de Lachman/McMurray) deve ser checado na correlação. Se não houver DUT, essa exigência é regra de operadora (fonte `OPERADORA`), não DUT.

## Notas de implementação

- Na planilha de correlação TUSS × Rol da ANS (RN 465/2021), o código `41101227` (RM de articulação unilateral) tem cobertura obrigatória no Rol sem DUT no Anexo II; portanto, exigências como Lachman/McMurray ou radiografia prévia para RM de joelho entram como regra de operadora/boa prática (`OPERADORA` / `BOA_PRATICA` na RFC-003), enquanto as DUTs curadas em `data/dut/*.yaml` cobrem os procedimentos com diretriz explícita no Anexo II (ex.: DUTs 3, 10, 38, 54, 65 e 110).
- O `IndiceLexical` em `app/bases.py` já implementa a ponderação híbrida BM25 (IDF) + `rapidfuzz` (`WRatio`) com expansão de siglas via `data/siglas.yaml`.
