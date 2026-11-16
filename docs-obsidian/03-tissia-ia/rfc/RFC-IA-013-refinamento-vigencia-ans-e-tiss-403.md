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
  - RFC-013-refinamento-vigencia-ans-e-tiss-403
origem: tissia-ai/docs/rfc/RFC-013-refinamento-vigencia-ans-e-tiss-403.md
---

# RFC-013 — Refinamento de Vigência ANS (Set/2026), TISS 04.03.00 e Governança de Bases

**Fase:** Refinamento Pré-Produção · **Status:** Concluída para TUSS (`202609`, tabelas 18/22/23/36/38/50) e TISS `04.03.00`; pendentes para o portão de produção: carga completa de `cid10` (`57` registros, `piloto_parcial_vigente_v2008`), atualização de `correlacao_rol` (`34` registros, `defasada`) e recuradoria CMO das 4 DUTs (`038`, `054`, `065`, `110`) · **Referências:** [[RFC-IA-001-bases-de-conhecimento|RFC-001]], [[RFC-IA-008-preditivo-historico-e-contratos|RFC-008]], [[RFC-IA-011-catalogo-terminologia-e-rag|RFC-011]], [[RFC-IA-012-treinamento-sft-vertex-e-rag-spsadt|RFC-012]], `ADR-PEP-0024` (`tissia`)

---

## 1. Contexto e Referência Regulatória (Competência Set/2026)

Esta RFC documenta a auditoria completa das bases regulatórias, schemas XML TISS, migrações, regras clínicas e pipeline de fine-tuning do `tissia-ai` (Spike), usando como fonte primária os pacotes oficiais da ANS disponíveis em `../tissia-ai-tunning/padrao_tiss_setembro_2026/`:

| Componente ANS | Competência / Versão | Arquivo Oficial Verificado | SHA-256 Oficial |
| :--- | :--- | :--- | :--- |
| **Comunicação** (prestador ↔ operadora) | `202511` / **`04.03.00`** | `PadroTISSComunicao_202511.zip` (102.554 B) | `db8640e1c3b87085892f54f838bfcea9934439ff365798c8428559f88c13d62d` |
| **Conteúdo e Estrutura** | `202511` | `PadroTISS_ComponentedeContedoeEstrutura_202511.zip` | `6a720ed27da8397fef584e30d794aa56f18906d253ccdf460c30b1019a3a4d3b` |
| **Organizacional** | `202609` | `PadroTISS_ComponenteOrganizacional_202609.pdf` | `fc973a683f772143d3355604e6e0566491756c49c657e1b12b0b33564d63d2d3` |
| **Representação de Conceitos em Saúde** (TUSS / Tab. 38) | `202609` | `Padrao_TISS_Representacao_de_Conceitos_em_Saude_202609.zip` (408.125.735 B) | `7f3d7a9794a8929fddd6f13af26dd9496087bf8b8670f275bd7ecd9a2f64172d` |
| **Arquivos Auxiliares** (Histórico TUSS + DE-PARA Tab. 38) | `202609` | `Padrao_TISS_arquivos_auxiliares_202609.zip` | `7c5809408876d1bd37ddaf98e5e5460b6974f86d5b06baa59f0d1e8d1095c3b` |

> **Fora de escopo:** Componente de Monitoramento TISS `01.06.00` / `01.06.01` (envio operadora → ANS).

---

## 2. Diagnóstico por Base e Componente

### 2.1 Tabelas em `data/MANIFEST.yaml`, `data/` e `data/dut/*.yaml`

| Base / Artefato Local | Estado Observado | Diagnóstico vs. Fonte Oficial Vigente | Licença Oficial | Status |
| :--- | :--- | :--- | :--- | :--- |
| **CID-10** (`data/cid10/subcategorias.csv`) | `57` linhas; manifesto tinha `publicado_em: '2026-01-01'` | O padrão TISS permanece exigindo exclusivamente **CID-10** (`DATASUS V2008`, sem data definida pela ANS para migração à CID-11; transição gradual prevista pelo MS apenas no SUS a partir de jan/2027). O CSV local é uma amostra piloto de 57 subcategorias, não a tabela V2008 completa. | Domínio público / DATASUS (sem arquivo de licença explícito no pacote V2008) | **Parcial (amostra piloto sobre versão vigente V2008)** |
| **TUSS Tab. 22** (`data/tuss/22_procedimentos.csv`) | `48` linhas; manifesto tinha `publicado_em: '2026-01-01'` | **Defasada e com códigos divergentes.** A fonte oficial (`TUSS 22 - VERSÃO 202609.xlsx`, SHA-256 `d64d044e...`) possui **5.969 códigos**. Das 48 linhas locais, apenas **7** têm termo exato; **37** têm descrições deslocadas/incorretas para o código (ex.: `20102020` no local é MAPA, mas na ANS é *Holter de 24 horas - 3 canais - digital*; `40301630` no local é CK-MB, mas na ANS é *Creatinina*; `40701018` no local é Cintilografia do miocárdio estresse, mas na ANS é *Angiografia radioisotópica*; `30724058` no local é Reconstrução ligamentar do joelho, mas na ANS é *Artroplastia de quadril*); e **4** não existem na Tab. 22 (`40901387`, `40901476`, `40901760`, além do sentinela sintético `99990001`). Na TUSS 22 vigente, Doppler de órgão isolado é `40901386` e Doppler arterial de membro inferior é `40901475`. | **CC BY-ND 4.0** (ANS / TUSS) | **Defasada e divergente** |
| **TUSS Tab. 38** (`data/tuss/mensagens.csv`) | `11` linhas; manifesto tinha `publicado_em: '2026-01-01'` | **Defasada e divergente.** A fonte oficial (`Tab 38` de `TUSS - Demais terminologias - VERSÃO 202609.xlsx`, SHA-256 `ad62541b...`) possui **470 termos** (além do arquivo auxiliar `TUSS38 - DE-PARA_tabela 38_fev2026_final.xlsx` com 641 mapeamentos). Das 11 linhas locais: `1006` e `1805` têm significados incorretos (`1006` na ANS é *ATENDIMENTO APÓS O DESLIGAMENTO DO BENEFICIÁRIO*, não autorização prévia; `1805` é *VALOR TOTAL DO PROCEDIMENTO DIFERENTE DO VALOR PROCESSADO*, não CID) e **7 códigos (`1203`, `1204`, `1701`, `1807`, `1812`, `1815`, `1820`) não existem na Tab. 38**. | **Sem licença explícita na publicação ANS** | **Defasada e divergente** |
| **Correlação TUSS × Rol** (`data/rol/correlacao.csv`) | `47` linhas; base citada `RN 465/2021`; `publicado_em: '2026-01-01'` | **Defasada.** A `RN 465/2021` segue como matriz central, mas é alterada em fluxo contínuo por RNs subsequentes (séries de 2025 como `RN 643..655` e de 2026 como `RNs 668, 671, 678, 679, 680 e 681/2026`). Além disso, herdou os mesmos códigos deslocados/inexistentes de `22_procedimentos.csv`. | **CC BY 4.0** (Dados Abertos ANS) | **Defasada** |
| **DUTs curadas** (`data/dut/*.yaml`) | `6` arquivos (`003`, `010`, `038`, `054`, `065`, `110`) | **4/6 com numeração incompatível com o Anexo II oficial da RN 465/2021.** `003.yaml` (Angiotomografia Coronariana) e `010.yaml` (Cintilografia do Miocárdio — porém inclui código `40701018`, que é *Angiografia radioisotópica*) conferem com as DUTs 3 e 10. Já no Anexo II oficial: **DUT 38** é *Estimulação Cerebral Profunda (DBS)* (não Holter `20102011`), **DUT 54** é *Medicamentos para efeitos adversos antineoplásicos* (não MAPA `20102020`), **DUT 65** é *Terapia imunobiológica* (não infiltração articular `30713030`), e **DUT 110** é *Análise molecular de DNA* (não artroscopia/reconstrução ligamentar `30715016`/`30724058`). Exigem recuradoria pelo CMO sobre o Anexo II oficial vigente. | Conteúdo regulatório ANS (`CC BY`) + curadoria clínica própria | **Parcialmente defasada / requer recuradoria CMO** |
| **Schemas XML TISS** (`app/tiss_xml.py`, `data/tiss/xsd/04.03.00/`) | Antes aceitava só `3.05.00..4.01.00`, assumia `4.01.00` sem tag e lia CID em SP/SADT | **Corrigido nesta entrega.** Versões `4.02.00` e `4.03.00` adicionadas em `VERSOES_TISS_SUPORTADAS`; XSDs oficiais `04.03.00` extraídos com `provenance.json`; ausência de tag de versão agora gera `ValueError` explícito; `sp_sadt` e `consulta` não extraem CID estruturado. | **Sem licença explícita na publicação ANS** | **Vigente (corrigido)** |

### 2.2 Duas Fontes da Mesma Base (`data/` vs. Postgres `catalogo_tuss` / `catalogo_cid`)

Diagnóstico empírico no banco local `medscribe`:
- `catalogo_tuss` contém **54.949 linhas** ingeridas via `scripts/ingest_catalogo_pg.py` a partir de `../tissia-ai-tunning/codigos_da_tuss/` (CSVs/ZIPs externos `tuss-{18,19,20,22,23,36,50}_2026111*.csv|zip`), enquanto `data/tuss/22_procedimentos.csv` tem apenas **48 linhas** (41 divergentes) e é usado em memória por [`app/bases.py`](file:///home/pedro/dev/pedro/tissia-ai/app/bases.py), [`app/regras.py`](file:///home/pedro/dev/pedro/tissia-ai/app/regras.py) e [`app/codificacao.py`](file:///home/pedro/dev/pedro/tissia-ai/app/codificacao.py).
- Mesmo entre `codigos_da_tuss/tuss-22_20261111.csv` (5.969 códigos = 100% dos códigos da ANS 202609) e a planilha oficial `TUSS 22 - VERSÃO 202609.xlsx`, há **157 diferenças menores de espaçamento duplo** no texto do termo, e `codigos_da_tuss` **não inclui a Tabela 38**.
- `catalogo_cid` contém **57 linhas** oriundas de `data/cid10/subcategorias.csv`.

**Fonte única proposta:** o pipeline oficial passa a extrair **diretamente dos pacotes oficiais ANS** (`Padrao_TISS_Representacao_de_Conceitos_em_Saude_<comp>.zip` e CSV oficial Rol/DATASUS) para gerar artefatos canônicos versionados (CSV/Parquet normalizado), atualizando `data/MANIFEST.yaml` e alimentando `scripts/ingest_catalogo_pg.py` exclusivamente a partir dessa mesma fonte (eliminando a bifurcação `../tissia-ai-tunning/codigos_da_tuss` × `data/`).

### 2.3 Regras, Parser e Sigilo de CID no TISS `04.03.00`

Verificação no schema oficial [`tissGuiasV4_03_00.xsd`](file:///home/pedro/dev/pedro/tissia-ai/data/tiss/xsd/04.03.00/tissGuiasV4_03_00.xsd):
- `ctm_consultaGuia` e `ctm_sp-sadtGuia` **não possuem elemento estruturado de CID** (`codigoDiagnostico`, `cid10` ou `diagnostico`); apenas guias de internação (`ctm_internacaoDadosSaida` / `ctm_internacaoDados`) trazem `diagnostico` / `diagnosticoObito`.
- No motor de regras ([`app/regras.py`](file:///home/pedro/dev/pedro/tissia-ai/app/regras.py) e [`data/regras/ans_tiss.yaml`](file:///home/pedro/dev/pedro/tissia-ai/data/regras/ans_tiss.yaml)), `TISS-SADT-INDICACAO` exige apenas `indicacao_clinica` (nunca `cids`), e `TISS-CID-SEXO` itera sobre `guia.cids` sem exigir preenchimento quando ausente.
- **Defeito de sigilo encontrado e corrigido:** [`app/ia/gemini_real.py`](file:///home/pedro/dev/pedro/tissia-ai/app/ia/gemini_real.py) (linhas 240–315) instruía o modelo a citar *"os CID informados"* no Campo 23 (Indicação Clínica) e o fallback local em [`app/ia/gemini.py`](file:///home/pedro/dev/pedro/tissia-ai/app/ia/gemini.py) (linha 600) concatenava ` (CID ...)` ao texto da justificativa. Ambos foram corrigidos com guarda determinística contra vazamento de CID no texto sugerido.

### 2.4 Migração `0004_catalogo_e_rag`

- **Causa raiz observada:** os bancos locais `medscribe` e `medscribe_test` estavam estacionados em `0003_preditivo_f3` enquanto `catalogo_tuss` (54.949 linhas) e `catalogo_cid` (57 linhas) já existiam no schema `public` (criados previamente fora do Alembic via `Base.metadata.create_all` em fixtures ou ingestão direta). Ao executar `alembic upgrade head`, [`migrations/versions/0004_catalogo_e_rag.py`](file:///home/pedro/dev/pedro/tissia-ai/migrations/versions/0004_catalogo_e_rag.py) falhava com `DuplicateTable`.
- **Correção aplicada:** migração refatorada com `sa.inspect(op.get_bind()).has_table(...)` e índices/extensões com `IF NOT EXISTS`, validada em banco limpo (`tissia_mig_fresh_test`, `0001 -> 0004`) e nos bancos pré-populados `medscribe` e `medscribe_test` (`0003 -> 0004`).

### 2.5 Dataset SFT (`RFC-012`) e `evals/casos/sinteticos.jsonl`

- **Rota corrigida na `RFC-012`:** substituída a citação inexistente `POST /v1/guias/retorno` por `POST /v1/tiss/lotes/ingerir`.
- **Códigos inválidos/deslocados no dataset e evals:**
  - `data/tunning/guias_extraidas.jsonl` contém `40901760` (2 registros: `evals-orto-acessorio-doppler-01/02`, código inexistente na TUSS 22) e `scripts/tunning/01_extrator_xml_e_tiss.py` trata `40301630` como *Hemograma Completo*, quando na TUSS 22 `202609` `40301630` é *Creatinina* (Hemograma é `40304361`).
  - `evals/casos/sinteticos.jsonl` usa códigos herdados da tabela piloto (`20102011`, `20102020`, `30713030`, `30715016`, `30724058`, `40701018`, `40803074`, `40804038`, `40901220`, `40901760`, `41101170`, `41101227`) cuja semântica está parcialmente deslocada em relação à TUSS 22 oficial.
  - **Regra de processo:** toda atualização da competência TUSS invalida os candidatos recuperados pelo RAG e exige regeneração dos pares (`02_sintetizador_anamnese.py` / `03_formatador_jsonl_vertex.py`) e re-execução completa dos evals (`evals/harness.py` e `05_avaliar_modelo_tuned.py`) antes de promover novo modelo ou release.

---

## 3. Lista Priorizada de Correções (Executadas vs. Planejadas)

| Prioridade | Item | Status | Esforço Estimado | Escopo / Arquivos |
| :--- | :--- | :--- | :--- | :--- |
| **P0** | Tornar migração `0004_catalogo_e_rag` idempotente e validá-la em banco limpo e pré-existente | **Executado** | — | [`migrations/versions/0004_catalogo_e_rag.py`](file:///home/pedro/dev/pedro/tissia-ai/migrations/versions/0004_catalogo_e_rag.py) |
| **P0** | Suportar TISS `04.02.00` e `04.03.00`, versionar XSDs `04.03.00` com `provenance.json`, exigir tag de versão e remover extração de CID em SP/SADT e consulta | **Executado** | — | [`app/tiss_xml.py`](file:///home/pedro/dev/pedro/tissia-ai/app/tiss_xml.py), [`data/tiss/xsd/04.03.00/`](file:///home/pedro/dev/pedro/tissia-ai/data/tiss/xsd/04.03.00/provenance.json), [`tests/test_rfc008_preditivo.py`](file:///home/pedro/dev/pedro/tissia-ai/tests/test_rfc008_preditivo.py) |
| **P0** | Garantir sigilo de CID na geração de Indicação Clínica (remover instrução/concatenação de CID) | **Executado** | — | [`app/ia/gemini_real.py`](file:///home/pedro/dev/pedro/tissia-ai/app/ia/gemini_real.py), [`app/ia/gemini.py`](file:///home/pedro/dev/pedro/tissia-ai/app/ia/gemini.py), [`tests/test_gemini_real.py`](file:///home/pedro/dev/pedro/tissia-ai/tests/test_gemini_real.py) |
| **P0** | Corrigir `data/MANIFEST.yaml` com competência real, SHA-256 reais e licenças (`CC BY`, `CC BY-ND`, Tabela 38) + preservar campos em `_ingest_utils.py` | **Executado** | — | [`data/MANIFEST.yaml`](file:///home/pedro/dev/pedro/tissia-ai/data/MANIFEST.yaml), [`scripts/_ingest_utils.py`](file:///home/pedro/dev/pedro/tissia-ai/scripts/_ingest_utils.py) |
| **P0** | Corrigir rota `POST /v1/tiss/lotes/ingerir` e registrar desvios de TUSS na `RFC-012` | **Executado** | — | [[RFC-IA-012-treinamento-sft-vertex-e-rag-spsadt\|RFC-012]] |
| **P0** | Unificar fonte única das tabelas TUSS (`202609`): extrair `tuss_18` (`3.595`), `tuss_22` (`5.969`), `tuss_23` (`2`), `tuss_36` (`4`), `tuss_38`/`tuss_mensagens` (`470`) e `tuss_50` (`9`) diretamente do `.zip`/`.xlsx` oficial ANS para `data/tuss/` e `Postgres` (`status: vigente`), e ativar teste automático de divergência (`data/` == `Postgres`) | **Executado** (`9f395a7`) | — | `scripts/ingest_ans_xlsx.py`, `scripts/ingest_catalogo_pg.py`, `data/tuss/`, `data/MANIFEST.yaml`, `tests/test_consistencia_fonte_unica.py` |
| **P0** | Completar carga oficial do **DATASUS CID-10 V2008** (`cid10` em `piloto_parcial_vigente_v2008`, `57` registros) e atualizar **Correlação TUSS × Rol** (`correlacao_rol` em `defasada`, `34` registros, para `RN 465/2021` + RNs contínuas 2025–2026) | **Pendente (portão de produção)** | **1–2 dias** | `data/cid10/subcategorias.csv`, `data/rol/correlacao.csv`, `data/MANIFEST.yaml`, `scripts/ingest_catalogo_pg.py` |
| **P0** | Recuradoria clínica do CMO nas 4 DUTs divergentes (`038`, `054`, `065`, `110`, além da revisão da `010`) e alinhamento dos códigos TUSS em `data/regras/boas_praticas.yaml`, `data/regras/operadora_005711.yaml` e `tests/test_rfc001/003/004/006` | **Parcial / Pendente de revisão humana CMO (portão de produção)** (`9f395a7`: TUSS alinhada à `202609` nas DUTs/regras/testes; numeração das DUTs `038`, `054`, `065`, `110` segue marcada "pendente de recuradoria CMO" no campo `fonte`) | **1–2 dias** (+ revisão CMO) | `data/dut/*.yaml`, `data/regras/*.yaml`, `tests/` |
| **P1** | Implementar contrato aditivo `GET /v1/bases/versoes` e enriquecimento opcional de proveniência para o portão do PEP (`ADR-PEP-0024`) | **Executado** (`9f395a7`) | **0,5–1 dia** | `app/api/bases.py`, `app/schemas.py`, `app/bases.py`, `tests/snapshots/openapi_v1.json` |
| **P1** | Regenerar dataset SFT (`scripts/tunning/`) e `evals/casos/sinteticos.jsonl` no leiaute `04.03.00` e códigos `202609` (corrigindo `40301630`→`40304361` e `40901760`) e reexecutar evals | **Executado** (`9f395a7`) | **1 dia** | `scripts/tunning/`, `data/tunning/`, `evals/casos/sinteticos.jsonl` |
| **P2** | Criar job agendado de alerta de defasagem ANS (sem auto-atualização) no Cloud Scheduler/CI | **Executado** (`f13ebe6`, `.github/workflows/vigencia-ans.yml`) | **0,5–1 dia** | `scripts/verificar_vigencia_ans.py`, workflow/Cloud Scheduler |

---

## 4. Contrato Aditivo Proposto em `/v1` para o Portão do PEP (`ADR-PEP-0024`)

Para não quebrar integradores que já consomem `versao_bases: str` (hash de 12 caracteres em `/healthz`, `/v1/guias/analisar` e `/v1/catalogo/*`), propõe-se uma **rota aditiva de leitura**:

### `GET /v1/bases/versoes`

- **Autenticação:** mesmo padrão público/interno de leitura de saúde/catálogo ou `X-API-Key` (a definir no PEP; `GET` idempotente com `Cache-Control: public, max-age=300`).
- **Payload de resposta (`200 OK`, estado real em `data/MANIFEST.yaml`):**

```json
{
  "versao_bases": "a1b2c3d4e5f6",
  "versao_tiss_suportada_max": "04.03.00",
  "alinhado_fonte_unica": true,
  "bases": {
    "cid10": {
      "competencia_ans": "V2008",
      "versao_tiss": null,
      "publicado_em": "2008-01-01",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "9dd5326fa5394803eff9ab86e4db41c8339a74adc76fa5358e631ebade3d65cd",
      "sha256_arquivo_local": "f09452b81bb5f2261607c234b6a33267c69eacf8ff334c15d7c9629da0345142",
      "licenca": "Domínio Público / DATASUS (sem arquivo de licença explícito nos artefatos V2008)",
      "total_registros_memoria": 57,
      "total_registros_pg": 57,
      "status": "piloto_parcial_vigente_v2008"
    },
    "correlacao_rol": {
      "competencia_ans": "RN 465/2021 (piloto parcial sem RNs contínuas 2025-2026)",
      "versao_tiss": null,
      "publicado_em": "2021-04-01",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "3dca7b89eaf7643cc50be289c241e5f020eb00c78ad3cc1c3bbc57063903ed2e",
      "sha256_arquivo_local": "6d6eea44924701f6510d79e384bf8c81979f6128f7df57cc8006b653f2caaeb3",
      "licenca": "CC BY 4.0 (Dados Abertos ANS)",
      "total_registros_memoria": 34,
      "total_registros_pg": null,
      "status": "defasada"
    },
    "tiss_comunicacao": {
      "competencia_ans": "202511",
      "versao_tiss": "04.03.00",
      "publicado_em": "2025-12-03",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "db8640e1c3b87085892f54f838bfcea9934439ff365798c8428559f88c13d62d",
      "sha256_arquivo_local": "d4c421c7e3cf936551b70d6bd794a419867b6daea554d557b4d928363c54044c",
      "licenca": "Sem licença explícita na publicação ANS",
      "total_registros_memoria": null,
      "total_registros_pg": null,
      "status": "vigente"
    },
    "tuss_18": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "e8222ac4d8eee6b39013e835b1a979abf84183880e5fe8359502af1159626f9c",
      "sha256_arquivo_local": "f00dceeffa3771864d6802206ecddf65db3f1768376d911b5b851ac2cdeaac4e",
      "licenca": "CC BY-ND 4.0 (ANS / TUSS)",
      "total_registros_memoria": null,
      "total_registros_pg": 3595,
      "status": "vigente"
    },
    "tuss_22": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "d64d044efa4367a1e932ec28bd2dfea92d2288c914622c9b5b9bf76724892dbb",
      "sha256_arquivo_local": "2462919455e5a2fbb1e458ab448b32f4e0ff608ea3d3c083e5597c05d47a9ad7",
      "licenca": "CC BY-ND 4.0 (ANS / TUSS)",
      "total_registros_memoria": 5969,
      "total_registros_pg": 5969,
      "status": "vigente"
    },
    "tuss_23": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "ad62541b18a4a7652bb883c9aa89940f712d54630c3610c067626acc3fba13c7",
      "sha256_arquivo_local": "cf79ac2d4eb07b8754942d9f2f810e516853e4f53a2613bb4dc69062908663e3",
      "licenca": "CC BY-ND 4.0 (ANS / TUSS)",
      "total_registros_memoria": null,
      "total_registros_pg": 2,
      "status": "vigente"
    },
    "tuss_36": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "ad62541b18a4a7652bb883c9aa89940f712d54630c3610c067626acc3fba13c7",
      "sha256_arquivo_local": "5ebb232759b2502ffb3c5dbcb2435ee6bcf33207fee15278e3efbf9fccfdda44",
      "licenca": "CC BY-ND 4.0 (ANS / TUSS)",
      "total_registros_memoria": null,
      "total_registros_pg": 4,
      "status": "vigente"
    },
    "tuss_50": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "ad62541b18a4a7652bb883c9aa89940f712d54630c3610c067626acc3fba13c7",
      "sha256_arquivo_local": "5b0984459774ee5c4ea985b8edee4fffb36c8f27ede597dcdf3ef8c182d3fa60",
      "licenca": "CC BY-ND 4.0 (ANS / TUSS)",
      "total_registros_memoria": null,
      "total_registros_pg": 9,
      "status": "vigente"
    },
    "tuss_mensagens": {
      "competencia_ans": "202609",
      "versao_tiss": null,
      "publicado_em": "2026-09-28",
      "carregado_em": "2026-11-16T00:00:00Z",
      "sha256": "ad62541b18a4a7652bb883c9aa89940f712d54630c3610c067626acc3fba13c7",
      "sha256_arquivo_local": "43454fceca89bbfdf5c63471caa8d84e9d6a213a6e87807a8df1eaa577fca6d9",
      "licenca": "Sem licença explícita na publicação ANS",
      "total_registros_memoria": 470,
      "total_registros_pg": 470,
      "status": "vigente"
    }
  }
}
```

> **Nota sobre o estado atual:** `tuss_mensagens` representa a **TUSS Tabela 38** no manifesto. Enquanto `tuss_18`, `tuss_22`, `tuss_23`, `tuss_36`, `tuss_38` (`tuss_mensagens`), `tuss_50` (`202609`) e `tiss_comunicacao` (`04.03.00`) já estão `vigente`, **`cid10`** (`57` registros, `piloto_parcial_vigente_v2008`), **`correlacao_rol`** (`34` registros, `defasada`) e a recuradoria CMO das 4 DUTs (`038`, `054`, `065`, `110`) permanecem como pendências operacionais bloqueantes para o portão de produção da `ADR-PEP-0024`.

**Regras do portão (`ADR-PEP-0024`):** o pipeline de release do PEP valida que `(1)` `alinhado_fonte_unica == true`, `(2)` cada base crítica tem `status == "vigente"` (exigindo a carga completa de `cid10` e a atualização de `correlacao_rol` antes de liberar produção), e `(3)` `bases.tuss_22.competencia_ans`, `bases.tuss_mensagens.competencia_ans`, `bases.tiss_comunicacao.versao_tiss` e `bases.correlacao_rol.competencia_ans` coincidem com a competência-alvo esperada pelo release do PEP.

---

## 5. Especificação do Teste de Fonte Única (`data/` ↔ Postgres)

Quando a unificação (P0) for executada, incluir `tests/test_consistencia_fonte_unica.py` executando no CI contra o Postgres de teste carregado:
1. Carregar `bases = carregar(Path("data"))` e consultar `CatalogoTuss` (`tabela = '22'`) e `CatalogoCid` no Postgres.
2. Falhar se `set(bases.tuss.keys()) != set(pg_tuss_22.keys())` ou se houver qualquer divergência em `termo`, `inicio_vigencia`, `fim_vigencia` ou `numero_dut`.
3. Falhar se `set(bases.cid.keys()) != set(pg_cid.keys())`.
4. Falhar se `versao_bases` gravada em `catalogo_tuss` / `catalogo_cid` diferir de `bases.versao`.
5. Falhar se `data/MANIFEST.yaml` tiver `sha256_arquivo` diferente do hash real de cada arquivo em `data/`.

---

## 6. Alerta de Defasagem Agendado e Runbook de Atualização

### 6.1 Job Agendado (Fora do Caminho da Requisição)

- **Script proposto:** `uv run python scripts/verificar_vigencia_ans.py --formato json`
- **Frequência sugerida:** diária (06:00 BRT) via GitHub Actions `schedule` ou Cloud Scheduler → Cloud Run Job.
- **Comportamento estrito:**
  1. Lê `competencia_ans` e `sha256_original` de `data/MANIFEST.yaml`.
  2. Consulta metadados públicos da página oficial do Padrão TISS e do Rol na ANS (`HEAD`/`GET` leve no HTML índice ou CKAN Dados Abertos ANS), sem baixar tabelas inteiras em ciclo normal nem gravar dados em produção.
  3. Compara a competência/versão TISS mais recente publicada pela ANS contra `data/MANIFEST.yaml`:
     - Código de saída `0`: tudo alinhado.
     - Código de saída `2` + alerta estruturado (Slack/Issue/Cloud Monitoring): nova competência publicada pela ANS (ex.: `202611` ou nova RN do Rol). **Nunca atualiza o repositório ou o banco automaticamente.**

### 6.2 Runbook Curto de Atualização de Competência

1. **Baixar pacote oficial ANS** (Representação de Conceitos, Comunicação quando houver nova versão XSD, Arquivos Auxiliares e Rol vigente) para o diretório de insumos.
2. **Conferir integridade e competência:** validar `sha256sum`, versão nos cabeçalhos dos `.xlsx` / `.xsd` e RNs vigentes do Rol.
3. **Executar ingestão unificada:** gerar CSVs canônicos em `data/`, atualizar `data/MANIFEST.yaml` (competência, `publicado_em`, `sha256_original`, `sha256_arquivo`, `licenca`) e sincronizar Postgres (`ingest_catalogo_pg.py` com embeddings se ativo).
4. **Atualizar curadoria e datasets:** revisar impactos em DUTs (`revisado_por` CMO), regras YAML, dataset SFT (`scripts/tunning/02` e `03`) e regenerar `evals` se códigos/candidatos do RAG mudaram.
5. **Rodar bateria completa de verificação:** `uv run alembic upgrade head`, `uv run task check` (incluindo teste de fonte única `data/` ↔ Postgres), `evals/harness.py` e `05_avaliar_modelo_tuned.py`.
6. **Publicar e validar portão:** realizar deploy e confirmar em `GET /v1/bases/versoes` a nova competência antes de liberar o release do PEP (`ADR-PEP-0024`).

---

## 7. Notas de Retorno para o PEP (`../tissia` — Sem Edição Direta)

- **Para `SPK-09` (`ADR-PEP-0024`):** ajustar o verificador do portão de produção no PEP para consumir o endpoint aditivo proposto `GET /v1/bases/versoes` (comparando `competencia_ans`, `sha256` e `alinhado_fonte_unica`), mantendo `versao_bases` (12 chars) apenas como identificador de invalidação de cache. Informar também o mapeamento de licenças (`CC BY 4.0` para Rol, `CC BY-ND 4.0` para TUSS — sem modificar textos oficiais — e ausência de licença explícita na Tabela 38/XSD).
- **Para `SPK-10` (TISS `04.03.00`):** o endpoint `POST /v1/tiss/lotes/ingerir` da Spike já aceita XMLs `04.02.00` e `04.03.00` e rejeita XML sem tag de versão com HTTP `422`. O PEP deve garantir o envio explícito de `<ans:versaoPadrao>04.03.00</ans:versaoPadrao>` (ou `<ans:Padrao>`) e **não** esperar extração/envio de CID estruturado em guias `SP/SADT` ou `Consulta`. Na `RFC-012` do `tissia-ai`, a rota citada para retorno foi corrigida para `POST /v1/tiss/lotes/ingerir`.

