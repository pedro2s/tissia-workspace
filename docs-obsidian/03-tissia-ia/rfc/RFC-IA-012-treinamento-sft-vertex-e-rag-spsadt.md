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
  - RFC-012-treinamento-sft-vertex-e-rag-spsadt
origem: tissia-ai/docs/rfc/RFC-012-treinamento-sft-vertex-e-rag-spsadt.md
---

# RFC-012 — Pipeline de Treinamento Supervisionado (SFT) na Vertex AI + RAG para Guias SP/SADT e Análise do Projeto `tiss`

**Fase:** F4+ (Especialização do Motor Anti-Glosa) · **Status:** Concluída (`scripts/tunning/`) · **ADRs:** 0004, 0007, 0008, 0010, 0011

## 1. Objetivo e Meta de Qualidade

Projetar e disponibilizar em [`scripts/tunning/`](file:///home/pedro/dev/pedro/tissia-ai/scripts/tunning) o pipeline executável de **Supervised Fine-Tuning (SFT)** de um modelo Gemini na **Vertex AI**, capaz de analisar a evolução clínica do paciente, cruzar com os candidatos recuperados pelo **RAG Híbrido (`pg_trgm` + `pgvector` — RFC-011)** e sugerir o preenchimento estruturado da **Guia SP/SADT (Padrão TISS v4.03.00)** com:

- **Assertividade $\ge 80\%$** (Top-1/Top-3 em procedimentos TUSS e subcategorias CID-10 no Golden Set e validação);
- **Recall de critérios ausentes da DUT $\ge 85\%$** e **Precisão de alertas $\ge 80\%$**;
- **0% de códigos inventados ou fora de vigência** (garantido por construção via RAG + seleção ancorada e validação pós-inferência contra `bases` / `catalogo_tuss`);
- **0% de fatos clínicos alucinados** na Indicação Clínica, mantendo a guarda determinística de `trecho_literal` e `palavras_novas` ([[ADR-IA-0008-human-in-the-loop|ADR-0008]]).

---

## 2. Análise Completa do Repositório Open-Source `~/dev/pedro/tiss` (`voa-tiss`)

Vasculhamos integralmente o repositório `~/dev/pedro/tiss` para identificar **(a)** artefatos XML/JSON/CSV aproveitáveis no treinamento SFT e **(b)** inspirações arquiteturais para evoluir o **Spike (`tissia-ai`)** e o **PEP (`tissia`)**.

### 2.1 Artefatos de `~/dev/pedro/tiss` Aproveitados no Dataset de Treinamento (`scripts/tunning/`)

| Fonte em `~/dev/pedro/tiss` | Conteúdo Encontrado | Conformidade TISS 4.03.00 e Tratamento em `01_extrator_xml_e_tiss.py` |
|---|---|---|
| `src/tiss/examples/autorizacao-solicitacao.xml` e `tests/fixtures/autorizacao-solicitacao.json` | Mensagem `SOLICITACAO_PROCEDIMENTOS` (`<ans:solicitacaoSP-SADT>`) completa na v4.03.00 com `<ans:indicacaoClinica>`, `<ans:caraterAtendimento>` e `<ans:procedimentosSolicitados>`. | **100% conforme XSD 4.03.00.** Extraída diretamente pelo script `01_extrator_xml_e_tiss.py`. |
| `src/tiss/examples/lote-valido.xml` e `tests/fixtures/xml/4.02.00-valido.xml` | Lotes `<ans:loteGuias>` contendo guias `<ans:guiaSP-SADT>` válidas contra o XSD oficial da ANS. | **Estruturalmente conforme XSD**, porém o autor usou `tipoAtendimento = 01` (Remoção) e `indicacaoAcidente = 0` (Trabalho) como placeholders genéricos para consultas (`10101012`). Nosso extrator corrige automaticamente para `tipo_atendimento = "04"` (Consulta) ou `"23"` (Exame) e `indicacao_acidente = "9"` (Não Acidente) para não contaminar a semântica clínica do SFT. |
| `bench/tarefas/gerar-sp-sadt-01..04.json` | 4 tarefas de benchmark de geração de lotes SP/SADT v4.03.00 (incluindo guias multi-procedimento com `10101012` e `10101039`). | Aproveitado o campo `"referencia"` (XML gabarito com hash MD5 válido) e descartado o campo `"errada"` (usado pelo benchmark para testar detecção de erro). |
| `agents/faturista/faturista/exemplos/atendimentos.csv` | Planilha de faturamento com guias SP/SADT multi-item combinando exames laboratoriais (`40301630` Hemograma Completo + `40302040` Glicose) e consultas (`10101012`). | Agrupado por `numero_guia` e pareado no Passo 2 com evolução clínica sintética de investigação de anemia e diabetes. |
| `src/tiss/_data/sadt/fields.json` | Catálogo completo de **958 linhas** mapeando todos os **68 campos impressos** da Guia SP/SADT v4.03.00 (Nov/2025), tamanhos máximos (`st_texto500`), obrigatoriedade (`obrigatorio`, `condicional`, `opcional`) e tabelas de domínio fechadas (Tabelas 23, 26, 35, 36, 39, 48, 50, 52, 61, 75, 76, 77, 87). | Utilizado como referência canônica para o schema [`scripts/tunning/schemas.py`](file:///home/pedro/dev/pedro/tissia-ai/scripts/tunning/schemas.py) (inclusive confirmando que na Tabela 50 o código vigente para Exame é `"23"` e que o formulário SP/SADT v4.03.00 não exporta CID-10 no XML). |

### 2.2 Inspirações de `~/dev/pedro/tiss` para Evoluir o Spike (`tissia-ai`) e o PEP (`tissia`)

#### A. Evoluções Recomendadas para o Spike (`tissia-ai`)

1. **Enriquecimento de `POST /v1/glosas/recurso` (`SPK-02`) com a Tabela 38 comentada (`src/tiss/_data/glosas.json`):**
   - O projeto `tiss` possui os **470 motivos oficiais da Tabela 38 da ANS** (competência `2026-09`) mapeados em 19 grupos de prefixo (`10` elegibilidade, `12` cadastro prestador, `14` autorização, `17` cobrança/cálculo, `18` procedimento/auditoria clínica, `28` pacote, `50` erro de estrutura XSD), cada um com uma **orientação processual determinística** de quais documentos anexar ao recurso.
   - **Ação sugerida no Spike:** Ingerir os 470 registros de `glosas.json` na base do `tissia-ai` para que o endpoint de Recurso de Glosa (`SPK-02-recurso-de-glosa.md`) combine a justificativa clínica (DUT + evolução) com a diretriz administrativa exata do grupo da Tabela 38.
2. **Prior Estatístico Real por Operadora na Predição de Risco (`RFC-008`) via Painel de Glosas ANS (`src/tiss/operadoras/_data/indicadores.json`):**
   - O arquivo `indicadores.json` traz a série histórica oficial da ANS (2019 a 2025) para **853 operadoras** (indexadas por `registro_ans`), contendo: `% glosa_inicial`, `% glosa_final`, `tempo_medio_pagamento` (dias) e `% guias_sem_retorno`.
   - **Ação sugerida no Spike:** Incorporar `indicadores.json` ao cálculo de `risco_glosa` (`RFC-008`) para calibrar o risco base pelo `registro_ans` da operadora mesmo quando a clínica ainda está em *cold start* (sem histórico próprio de retornos).
3. **Auditoria de Preço Unitário via Referência D-TISS (`src/tiss/referencia/`):**
   - O módulo `tiss.referencia` consulta os percentis `P25`, `P50` e `P75` pagos pelas operadoras por UF e código TUSS (dados abertos D-TISS da ANS), permitindo alertar quando o valor orçado na guia diverge drasticamente da mediana regional.

#### B. Evoluções Recomendadas para o PEP (`tissia`)

1. **Gerador de XML TISS 4.03.00 Determinístico + Hash MD5 ISO-8859-1 (`src/tiss/_lote_codec.py` e `_autorizacao_xml.py`):**
   - O PEP (`tissia`) pode adotar diretamente a lógica de `_autorizacao_xml.py` (para gerar o XML de `SOLICITACAO_PROCEDIMENTOS` / `solicitacaoSP-SADT`) e `_lote_codec.py` (para `ENVIO_LOTE_GUIAS` / `guiaSP-SADT`), que garante a ordem exata das tags exigida pelo XSD (`_ORDERS`) e calcula o `<ans:hash>` MD5 sobre a concatenação das folhas em `ISO-8859-1`.
2. **Renderizador HTML/PDF Fiel ao Formulário Oficial SP/SADT (`src/tiss/_sadt.py` e `_data/sadt/fields.json`):**
   - O pacote `tiss` possui o template HTML/CSS oficial da Guia SP/SADT com todos os 68 campos numerados do layout da ANS e validação estrita de domínios fechados (`_resolve_enum`), ideal para impressão ou exportação PDF no PEP.
3. **Auto-Correção de Lotes com JSON Patch RFC 6902 (`src/tiss/_patch.py`):**
   - Quando a validação pré-envio detecta divergência aritmética (ex.: `valorTotalGeral` diferente da soma dos procedimentos, como nas tarefas `corrigir-01..07.json`), o validador anexa um `fix` em formato JSON Patch RFC 6902 que o PEP pode aplicar automaticamente antes de exportar o XML.
4. **Conciliação Automática de Demonstrativo de Retorno (`src/tiss/_demonstrativo.py`):**
   - Lê o XML `DEMONSTRATIVO_ANALISE_CONTA` devolvido pela operadora, classifica cada guia em `pago`, `glosado` ou `glosa_parcial` e já extrai os códigos da Tabela 38 para alimentar o endpoint de ingestão `POST /v1/tiss/lotes/ingerir` (`RFC-008`) e o fluxo de recurso de glosa (`SPK-02`).

---

## 3. Estrutura Implementada em `tissia-ai/scripts/tunning/`

Conforme definido, todos os scripts de preparação de dados, validação, disparo do treinamento SFT na Vertex AI e avaliação residem diretamente em [`scripts/tunning/`](file:///home/pedro/dev/pedro/tissia-ai/scripts/tunning):

```text
scripts/tunning/
├── README.md                         # Guia passo a passo de execução do pipeline
├── schemas.py                        # Schemas Pydantic (GuiaSPSADTExtraida, ProcedimentoSolicitado) TISS 4.03.00
├── 01_extrator_xml_e_tiss.py         # Extrai 75 guias base de evals/casos/sinteticos.jsonl + repo ../tiss
├── 02_sintetizador_anamnese.py       # Síntese reversa, Data Augmentation (149 pares), scrub PII e RAG Híbrido
├── 03_formatador_jsonl_vertex.py     # Validação estrita (0 erros) e split 80/10/10 no formato Vertex AI SFT
├── 04_disparar_sft_vertex.py         # Disparo do job client.tunings.tune() via google-genai (com --dry-run)
└── 05_avaliar_modelo_tuned.py        # Avaliação de Schema Compliance, Zero Hallucination, Domínios e F1 TUSS
```

### Resultado da Execução Local do Pipeline (`data/tunning/`)

O pipeline completo já foi executado e validado contra o banco PostgreSQL (`medscribe`) com o catálogo unificado:

- **`01_extrator_xml_e_tiss.py`**: Extraiu **75 guias SP/SADT** (**60** de `evals/casos/sinteticos.jsonl` + **15** de XMLs, benchmarks `gerar-sp-sadt-01..04.json` e `atendimentos.csv` de `~/dev/pedro/tiss`).
- **`02_sintetizador_anamnese.py`**: Gerou **149 exemplos enriquecidos** (incluindo variações de escrita médica abreviada/telegráfica), aplicou `app.pii.scrub()` em 100% dos textos e injetou os candidatos recuperados via `buscar_contexto_rag()` (`app/catalogo.py`).
- **`03_formatador_jsonl_vertex.py`**: Validou **149 exemplos válidos (0 erros)** e gerou:
  - `data/tunning/train_sft_vertex.jsonl` (**119 exemplos**)
  - `data/tunning/val_sft_vertex.jsonl` (**14 exemplos**)
  - `data/tunning/test_sft_vertex.jsonl` (**16 exemplos**)
- **`05_avaliar_modelo_tuned.py`**: Self-check no conjunto de teste confirmou `schema_compliance_rate = 1.0`, `zero_hallucination_rate = 1.0`, `domain_exact_match_rate = 1.0` e `tuss_f1_score = 1.0`.

> **Nota de auditoria (RFC-013 — set/2026):** Conferência contra o pacote oficial ANS 202609 identificou dois desvios que exigem regeneração do dataset e evals antes do tuning final: (1) em `atendimentos.csv` / `01_extrator_xml_e_tiss.py`, o código `40301630` foi tratado como *Hemograma Completo*, porém na TUSS 22 oficial `40301630` é *Creatinina - pesquisa e/ou dosagem* (Hemograma é `40304361`); (2) `evals/casos/sinteticos.jsonl` (casos `orto-acessorio-doppler-01/02` e similares) herdou códigos deslocados/inexistentes de `data/tuss/22_procedimentos.csv` (ex.: `40901760`). Ver plano de saneamento na [[RFC-IA-013-refinamento-vigencia-ans-e-tiss-403|RFC-013]].
