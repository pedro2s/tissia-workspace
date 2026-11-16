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
  - 0011-catalogo-unificado-e-rag-pgvector
origem: tissia-ai/docs/adr/0011-catalogo-unificado-e-rag-pgvector.md
---

# ADR-0011 — Catálogo unificado no Postgres (`pg_trgm` + `pgvector`), ingestão em camadas e RAG para modelo treinado

**Status:** Aceita — 11/11/2026
**Substitui parcialmente:** [[ADR-IA-0006-bases-versionadas-sem-banco-vetorial|ADR-0006]] (mantém as bases nucleares versionadas em `data/` com `MANIFEST.yaml`, mas introduz o armazenamento unificado no Postgres com `pg_trgm` e `pgvector` para o catálogo do PEP e o RAG de códigos).

## Contexto

Três necessidades convergiram após a entrega das fases F0–F3:

1. **Catálogo único para o PEP (SPK-09):** O PEP (`tissia`) precisa consultar e buscar códigos TUSS e CID-10 diretamente na Spike (`GET /v1/catalogos/tuss` e `GET /v1/catalogos/cid`) para eliminar listas fixas no frontend e garantir que qualquer código lançado no pedido de exame ou na guia SP/SADT pertença à mesma base(`versao_bases`) que o motor anti-glosa valida.
2. **Escala real das tabelas TUSS da ANS (`../tissia-ai-tunning`):** Ao incorporar o pacote oficial completo da ANS (Setembro/Novembro de 2026), os volumes reais por tabela são altamente assimétricos:
   - **Tabelas nucleares e de domínio da guia:** Tabela 22 — Procedimentos (5.969 itens vigentes), Tabela 18 — Diárias e Taxas (3.595 itens), Tabela 23 — Caráter do Atendimento (2 itens), Tabela 36 — Indicação de Acidente (4 itens), Tabela 50 — Tipo de Atendimento (9 itens), Tabela 38 — Mensagens de Glosa e CID-10 (~12.400 subcategorias). Juntas somam ~22 mil registros (< 2,5 MB em CSV).
   - **Tabelas massivas de insumos/despesas:** Tabela 20 — Medicamentos (45.355 itens) e Tabela 19 — Materiais e OPME (1.535.881 itens, ~390 MB descompactado, onde cada modelo/fabricante/registro ANVISA é um código individual). Colocar 1,58 milhão de linhas em `data/` no Git e carregá-las em RAM no startup violaria a invariante da [[RFC-IA-001-bases-de-conhecimento|RFC-001]] (`carregar() < 3 s` e `< 300 MB RSS` no Cloud Run).
3. **Treinamento de modelo Gemini na Vertex AI (Supervised Fine-Tuning) + Prevenção de alucinação de códigos:**
   - Estamos preparando o treinamento supervisionado (SFT) de um modelo Gemini na Vertex AI para sugerir o preenchimento estruturado de guias SP/SADT a partir do prontuário.
   - Modelos de linguagem (mesmo após *fine-tuning*) alucinam dígitos ao tentar memorizar dezenas de milhares de códigos numéricos de 8 a 10 dígitos e não acompanham janelas de vigência trimestrais da ANS.
   - Portanto, o catálogo exposto ao PEP e o índice de recuperação (RAG) que abastece o modelo treinado precisam partir da **mesma fonte de verdade**.

## Decisão

1. **Tabelas unificadas no Postgres (`catalogo_tuss` e `catalogo_cid`) com `pg_trgm` e `pgvector`:**
   - Uma única tabela `catalogo_tuss` (particionada logicamente pela coluna `tabela`: `'18'`, `'19'`, `'20'`, `'22'`, `'23'`, `'36'`, `'38'`, `'50'`) e uma tabela `catalogo_cid` servem simultaneamente:
     - **Busca léxica e determinística do PEP (`SPK-09`):** índices B-tree de prefixo em `codigo` e índices GIN `pg_trgm` em `termo_norm` (minúsculas, sem acento).
     - **RAG híbrido para o modelo de IA:** coluna `embedding vector(768)` indexada via `HNSW (embedding vector_cosine_ops)` gerada com o modelo `text-multilingual-embedding-002` da Vertex AI, combinada com o score lexical via **Reciprocal Rank Fusion (RRF)**.
2. **Ingestão em duas camadas (`data/` + Postgres):**
   - **Camada 1 (Versionada no Git em `data/` + Memória + Postgres):** As bases nucleares de validação (`tuss/22_procedimentos.csv`, `tuss/18_taxas_diarias.csv`, `tuss/23_carater_atendimento.csv`, `tuss/36_indicacao_acidente.csv`, `tuss/50_tipo_atendimento.csv`, `tuss/mensagens.csv`, `cid10/subcategorias.csv`, `rol/correlacao.csv` e `dut/*.yaml`) continuam versionadas com `data/MANIFEST.yaml` e carregadas em `app/bases.py`. Se a tabela do Postgres estiver vazia (ex.: ambiente de teste efêmero ou bootstrap), o catálogo responde em memória a partir de `app/bases.py`, garantindo disponibilidade contínua.
   - **Camada 2 (Carga direta no Postgres via `scripts/ingest_catalogo_pg.py`):** Os pacotes completos de `../tissia-ai-tunning/codigos_da_tuss/` — incluindo a Tabela 22 completa (5.969 itens), Tabela 18 (3.595 itens), Tabela 20 (45.355 medicamentos) e Tabela 19 (1,53 milhão de materiais/OPME lidos em streaming direto do `.zip`) — são carregados no Postgres em lote.
   - **Seletividade de embeddings (`pgvector`):** Embeddings vetoriais são gerados para as tabelas de decisão clínica e codificação de procedimentos/diagnósticos (Tabela 22, Tabela 18 e CID-10). Tabelas de domínio minúsculo (23, 36, 50) vão direto no prompt/schema do modelo, e SKUs comerciais da Tabela 19/20 são buscados por filtro léxico/trigram e metadados ANVISA.
3. **Papel do modelo Gemini treinado (SFT) vs. RAG na codificação SP/SADT:**
   - **No treinamento (SFT na Vertex AI):** O modelo é treinado para:
     1. Classificar os campos de domínio fechado da guia SP/SADT (`carater_atendimento` — Tabela 23, `indicacao_acidente` — Tabela 36, `tipo_atendimento` — Tabela 50);
     2. Redigir a Indicação Clínica (Campo 23, ≤ 500 caracteres) em linguagem técnica de auditoria médica, ancorada exclusivamente nos fatos da evolução;
     3. Extrair e normalizar os termos canônicos de diagnósticos e procedimentos realizados/solicitados (sem emitir códigos de 8 dígitos livres).
   - **Na inferência com RAG (Grounded Selection):** Para cada termo extraído pelo modelo, a Spike executa a busca híbrida (BM25/`pg_trgm` + `pgvector` via RRF) no catálogo unificado e entrega os candidatos vigentes (`"<codigo> - <termo>"`) ao decisor/modelo para escolha fechada. Nenhum código fora do catálogo vigente pode sair da API.

## Consequências

- O catálogo consultado pelo médico no PEP (`/v1/catalogos/tuss` e `/v1/catalogos/cid`) e os candidatos do RAG usados pela IA nunca divergem e compartilham `versao_bases`.
- O container Cloud Run mantém inicialização rápida (`< 3 s`) e baixo consumo de RAM (`< 300 MB`), delegando o armazenamento de 1,58 milhão de itens de OPME/medicamentos para o Postgres com índices `pg_trgm`.
- O pipeline de *fine-tuning* na Vertex AI fica desacoplado de atualizações trimestrais da tabela TUSS: quando a ANS publica novos códigos, basta atualizar o catálogo/RAG sem retreinar o modelo.

