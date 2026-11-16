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
  - RFC-011-catalogo-terminologia-e-rag
origem: tissia-ai/docs/rfc/RFC-011-catalogo-terminologia-e-rag.md
---

# RFC-011 — Catálogo de Terminologia TUSS/CID-10 (SPK-09) e RAG Híbrido (`pgvector`)

**Fase:** F4+ (Integração PEP Onda 4 & Preparação SFT Vertex) · **Status:** Concluída · **ADRs:** 0006, 0008, 0010, 0011

## Objetivo

1. Expor na Spike as rotas determinísticas de listagem, busca incremental (autocomplete) e detalhamento de códigos **TUSS** e **CID-10** solicitadas pelo PEP em `SPK-09-catalogo-tuss.md`.
2. Unificar o armazenamento da terminologia no Postgres (`catalogo_tuss` e `catalogo_cid`) com suporte às tabelas oficiais da ANS baixadas em `../tissia-ai-tunning` (Tabelas 18, 19, 20, 22, 23, 36, 38, 50 e CID-10) e busca híbrida léxica (`pg_trgm` / BM25) + semântica (`pgvector` com embeddings da Vertex AI via Reciprocal Rank Fusion) para servir de RAG ao modelo Gemini treinado.

## Contrato da API (aditivo em `/v1`)

Todas as rotas exigem a mesma autenticação por integrador (`Authorization: Bearer tsk_...`), não recebem nem retornam PII e incluem `versao_bases` em toda resposta para invalidação de cache no PEP.

### 1. `GET /v1/catalogos/tuss`

Busca códigos TUSS vigentes por prefixo de código ou por termo normalizado (sem acento, sem caixa, com expansão de siglas médicas de `data/siglas.yaml`).

**Query parameters:**
- `q` (`string`, opcional, default `""`, máx. 120 chars): texto livre ou prefixo numérico do código. Se vazio, lista os primeiros itens vigentes ordenados por código.
- `limit` (`int`, opcional, default `20`, mín. `1`, máx. `50`): quantidade máxima de itens retornados (sem cursor).
- `tabela` (`"18" | "19" | "20" | "22" | "23" | "36" | "38" | "50"`, opcional, default `"22"`): tabela TUSS alvo. O PEP consome por padrão a Tabela 22 (Procedimentos e Eventos em Saúde).

**Response `200 OK`:**
```json
{
  "itens": [
    {
      "codigo_tuss": "41101227",
      "termo": "Ressonância magnética de articulação - unilateral",
      "grupo": "IMAGEM",
      "tem_dut": false
    }
  ],
  "versao_bases": "a1b2c3d4e5f6"
}
```

### 2. `GET /v1/catalogos/tuss/{codigo}`

Retorna os metadados de um código TUSS específico e, quando houver Diretriz de Utilização (DUT) curada associada, os critérios resumidos para exibição em tooltip no PEP.

**Query parameters:**
- `tabela` (`string`, opcional, default `"22"`): tabela TUSS alvo.

**Response `200 OK` (ou `404 Not Found` se o código não existir):**
```json
{
  "codigo_tuss": "41001230",
  "termo": "Angiotomografia coronariana",
  "grupo": "IMAGEM",
  "tem_dut": true,
  "numero_dut": 3,
  "titulo_dut": "ANGIOTOMOGRAFIA CORONARIANA",
  "criterios_dut": [
    {
      "id": "c1",
      "texto": "Avaliação inicial de pacientes sintomáticos com probabilidade pré-teste intermediária (10% a 90%) de doença arterial coronariana (DAC) obstrutiva."
    }
  ],
  "versao_bases": "a1b2c3d4e5f6"
}
```

### 3. `GET /v1/catalogos/cid`

Busca subcategorias CID-10 por prefixo de código (com ou sem ponto, ex.: `S83.5` ou `S835`) ou por termo normalizado.

**Query parameters:**
- `q` (`string`, opcional, default `""`, máx. 120 chars)
- `limit` (`int`, opcional, default `20`, mín. `1`, máx. `50`)

**Response `200 OK`:**
```json
{
  "itens": [
    {
      "codigo_cid": "S835",
      "termo": "Entorse e distensão envolvendo ligamento cruzado (anterior) (posterior) do joelho"
    }
  ],
  "versao_bases": "a1b2c3d4e5f6"
}
```

### 4. `GET /v1/catalogos/cid/{codigo}`

Retorna o detalhe de um código CID-10 (aceita código com ou sem ponto na URL, devolvendo sempre o código canônico sem ponto).

**Response `200 OK` (ou `404 Not Found`):**
```json
{
  "codigo_cid": "S835",
  "termo": "Entorse e distensão envolvendo ligamento cruzado (anterior) (posterior) do joelho",
  "restricao_sexo": null,
  "versao_bases": "a1b2c3d4e5f6"
}
```

## Modelo de Dados e Índices (`migrations/versions/0004_catalogo_e_rag.py`)

Extensões habilitadas na migração: `pg_trgm`, `unaccent` e `vector` (`pgvector`).

- **`catalogo_tuss`**:
  - Chave primária composta: `(tabela, codigo)`.
  - Colunas: `termo`, `termo_norm`, `inicio_vigencia`, `fim_vigencia`, `capitulo`, `grupo`, `subgrupo`, `numero_dut`, `tem_dut`, `metadados` (`JSONB` — armazena fabricante, modelo, registro ANVISA, apresentação e laboratório das Tabelas 19 e 20), `versao_bases`, `embedding` (`vector(768)`).
  - Índices:
    - `ix_catalogo_tuss_tabela_codigo_prefix`: B-tree `(tabela, codigo varchar_pattern_ops)` para busca instantânea por prefixo de código.
    - `ix_catalogo_tuss_termo_norm_trgm`: GIN `(termo_norm gin_trgm_ops)` para busca determinística por substring/trigram.
    - `ix_catalogo_tuss_embedding_hnsw`: HNSW `(embedding vector_cosine_ops)` para recuperação semântica do RAG.
- **`catalogo_cid`**:
  - Chave primária: `codigo`.
  - Colunas: `termo`, `termo_norm`, `restricao_sexo`, `versao_bases`, `embedding` (`vector(768)`).
  - Índices equivalentes (prefixo B-tree, GIN `pg_trgm` e HNSW `vector_cosine_ops`).

## Arquitetura do RAG para o Modelo Treinado na Vertex AI

```
Evolução Clínica / Prontuário
       │
       ▼
[1. Modelo Gemini (Base ou Fine-Tuned SFT na Vertex AI)]
   ├─ Preenche campos de domínio fechado da SP/SADT (Tabelas 23, 36, 50)
   ├─ Redige Indicação Clínica (Campo 23) ancorada em fatos literais
   └─ Extrai descritores normalizados de Procedimentos e Diagnósticos
       │
       ▼
[2. Recuperador Híbrido no Catálogo Unificado (app/catalogo.py)]
   ├─ Busca Léxica: BM25/RapidFuzz + pg_trgm (com expansão de siglas de data/siglas.yaml)
   ├─ Busca Semântica: Vertex AI Embeddings (text-multilingual-embedding-002) + pgvector (<=>)
   └─ Fusão: Reciprocal Rank Fusion (RRF, k=60) → Top-50 candidatos vigentes
       │
       ▼
[3. Decisor / Grounded Reranking (app/codificacao.py)]
   └─ Seleciona estritamente entre as opções do catálogo + "nenhum_dos_anteriores"
```

## Scripts de Ingestão (`scripts/ingest_tuss.py` e `scripts/ingest_catalogo_pg.py`)

1. **`scripts/ingest_tuss.py`**: suporta tanto o formato clássico da ANS quanto o formato exportado em `../tissia-ai-tunning/codigos_da_tuss/tuss-*.csv` (`id;source;display_name;extras_*`) e aceita `--tipo 18|22|23|36|50|mensagens`.
2. **`scripts/ingest_catalogo_pg.py`**: sincroniza `data/` e os pacotes completos de `../tissia-ai-tunning/codigos_da_tuss/` (incluindo leitura em streaming dos arquivos `.zip` das Tabelas 19 e 20) diretamente para o Postgres em lotes (`ON CONFLICT DO UPDATE`), com flag opcional `--gerar-embeddings` para popular a coluna `embedding` via Vertex AI.

## Critérios de Aceite

- [x] `GET /v1/catalogos/tuss` e `GET /v1/catalogos/cid` respondem em conformidade com o contrato de `SPK-09`, respeitando `limit` (padrão 20, máx. 50) e expondo `versao_bases`.
- [x] Busca por prefixo de código (`4110`, `S83`, `S83.5`) e busca por termo sem acento/caixa ou com siglas (`rm joelho`, `angiotc`, `lca`) retornam os itens esperados no topo.
- [x] `GET /v1/catalogos/tuss/{codigo}` retorna `tem_dut` e a lista resumida `criterios_dut` (`id`, `texto`) para códigos com DUT curada, e `404` para código inexistente.
- [x] Migração `0004_catalogo_e_rag.py` cria as tabelas `catalogo_tuss` e `catalogo_cid` com suporte a `pg_trgm` e `pgvector`.
- [x] `recuperar_candidatos_rag_tuss` e `recuperar_candidatos_rag_cid` combinam ranking lexical e vetorial (`pgvector`) via RRF sem permitir códigos fora das bases vigentes.

