---
projeto: tissia-ia
tipo: adr
status: aceita
tags:
  - tissia
  - tissia-ia
  - adr
  - decisao-arquitetural
aliases:
  - 0006-bases-versionadas-sem-banco-vetorial
origem: tissia-ai/docs/adr/0006-bases-versionadas-sem-banco-vetorial.md
---

# ADR-0006 — Bases de conhecimento como arquivos versionados; sem banco vetorial na F1

**Status:** Aceita — 29/10/2026 · **Substituída parcialmente por:** [[ADR-IA-0011-catalogo-unificado-e-rag-pgvector|ADR-IA-0011]] (mantém bases nucleares versionadas em `data/` com `MANIFEST.yaml`; catálogo unificado e RAG passam para Postgres com `pg_trgm` + `pgvector`)

## Contexto

O PRD v2 previa um banco vetorial (RAG) com TUSS, CID, Rol/DUT e regras de operadoras. Na F1:

- A DUT é procurada por **chave exata** (código TUSS → nº da DUT, via planilha de correlação TUSS×Rol da ANS). Busca semântica não ajuda.
- As regras determinísticas são pequenas e curadas à mão.
- A sugestão TUSS/CID ([[RFC-IA-006-sugestao-tuss-cid|RFC-006]]) precisa recuperar candidatos entre cerca de 6 mil procedimentos (tabela 22) e cerca de 12 mil subcategorias CID-10. Isso cabe em memória, e a recuperação pode ser lexical sobre termos já normalizados pelo LLM.

## Decisão

- Bases normalizadas em CSV/YAML dentro de `data/`, versionadas no git, com `data/MANIFEST.yaml` (fonte, URL, data de publicação, hash). Geradas por scripts em `scripts/` a partir dos arquivos oficiais da ANS/DATASUS ([[RFC-IA-001-bases-de-conhecimento|RFC-001]]).
- Carregadas em memória no startup. A versão das bases (hash do manifesto) vai em toda resposta e em toda análise gravada.
- Recuperação de candidatos para a sugestão de códigos: busca lexical em memória (tokens normalizados sem acento + similaridade com `rapidfuzz`) filtrada por especialidade/capítulo.
- **Sem banco vetorial e sem pgvector na F1.**

## Consequências

- Atualizar a tabela TUSS = PR com diff revisável. É auditável e reversível.
- `# ponytail:` busca lexical em memória. Upgrade para embeddings + pgvector (mesmo Cloud SQL) se a recall@50 de candidatos no golden set ficar abaixo de 95% ([[RFC-IA-006-sugestao-tuss-cid|RFC-006]]).
- Regras por operadora cadastradas por clínica (dado de cliente, não base pública) vão para o Postgres quando existirem (F2).
