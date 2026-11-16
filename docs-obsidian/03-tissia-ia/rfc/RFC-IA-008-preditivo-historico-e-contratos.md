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
  - RFC-008-preditivo-historico-e-contratos
origem: tissia-ai/docs/rfc/RFC-008-preditivo-historico-e-contratos.md
---

# RFC-008 — Motor preditivo sobre histórico TISS, auditoria de contratos e guias esquecidas

**Fase:** F3 · **Status:** Concluída · **ADRs:** 0004, 0005, 0007, 0009

## Objetivo

Evoluir o Tissia IA de um validador puramente baseado em regras/DUT para um **motor preditivo de risco de glosa** alimentado pelo histórico real de faturamento (lotes XML TISS de envio e demonstrativos de retorno da operadora), auditar divergências entre **contratos de operadoras** e a parametrização do PEP, e identificar **guias esquecidas** (atendimentos realizados na agenda sem guia TISS faturada).

---

## 1. Modelo de Dados (Migração Aditiva `0003_preditivo_f3`)

```sql
CREATE TABLE historico_guias_tiss (
  id                  uuid PRIMARY KEY,
  integrador_id       uuid NOT NULL REFERENCES integradores(id) ON DELETE CASCADE,
  clinica_id          uuid NOT NULL REFERENCES clinicas(id) ON DELETE RESTRICT,
  lote_ref            text NOT NULL,
  versao_tiss         text NOT NULL,         -- ex.: 3.05.00, 4.00.00, 4.01.00
  guia_ref            text NOT NULL,
  tipo_guia           text NOT NULL,         -- consulta | sp_sadt | internacao
  registro_ans        text NOT NULL,
  codigo_tuss         text NOT NULL,
  cid                 text,
  prestador_ref       text,
  data_atendimento    date,
  valor_cobrado       numeric(12, 2),
  valor_glosado       numeric(12, 2),
  status_desfecho     text NOT NULL,         -- enviado | pago | glosado_parcial | glosado_total
  codigo_glosa_tuss   text,
  criado_em           timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_hist_tiss_op_proc ON historico_guias_tiss (integrador_id, registro_ans, codigo_tuss);
CREATE INDEX ix_hist_tiss_clinica_data ON historico_guias_tiss (clinica_id, data_atendimento);

CREATE TABLE contratos_operadora (
  id                         uuid PRIMARY KEY,
  integrador_id              uuid NOT NULL REFERENCES integradores(id) ON DELETE CASCADE,
  clinica_id                 uuid NOT NULL REFERENCES clinicas(id) ON DELETE CASCADE,
  registro_ans               text NOT NULL,
  codigo_tuss                text NOT NULL,
  valor_contratado           numeric(12, 2),
  exige_autorizacao_previa   boolean NOT NULL DEFAULT false,
  observacoes                text,
  criado_em                  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_contratos_clinica_op_tuss ON contratos_operadora (clinica_id, registro_ans, codigo_tuss);
```

---

## 2. Parser de Lotes XML TISS (`app/tiss_xml.py`)

O módulo `app/tiss_xml.py` faz o parse seguro (bloqueando entidades externas / DTD) de arquivos XML do padrão ANS TISS nas versões `3.05.00`, `4.00.00` e `4.01.00` (com ou sem prefixo de namespace `ans:`):

1. **Lotes de Envio (`loteGuiasWS` / `guiasTISS`):**
   - Extrai `versaoPadrao`, `registroANS`, `numeroLote` e as guias (`guiaConsulta`, `guiaSP-SADT`, `guiaSolicitacaoInternacao`).
   - Para cada procedimento executado/solicitado, extrai `codigoProcedimento`, `valorTotal`, `dataExecucao` e `codigoDiagnostico`.
   - **Minimização LGPD (ADR-0007):** tags de identificação direta (`nomeBeneficiario`, `numeroCarteira`, `numeroCNS`, `cpf`) nunca são lidas nem armazenadas.
2. **Demonstrativos de Retorno / Análise de Conta (`demonstrativoAnaliseConta` / `relacaoGuias`):**
   - Extrai `numeroGuiaPrestador` (`guia_ref`), `codigoProcedimento`, `valorInformado`, `valorGlosa` e `codigoGlosa` (Tabela 38 TUSS), classificando o item em `pago`, `glosado_parcial` ou `glosado_total`.

### Endpoint: `POST /v1/tiss/lotes/ingerir`
Recebe o XML TISS (`xml_conteudo`, `clinica_ref`), persiste os registros em `historico_guias_tiss` e, quando se tratar de demonstrativo de retorno com glosa/pagamento, atualiza automaticamente os `desfechos` das análises correspondentes por `(clinica_ref, guia_ref)`.

---

## 3. Score Preditivo de Risco de Glosa (`app/preditivo.py`)

### 3.1 Modelo Empírico Bayesiano (Beta-Binomial Smoothing)
Para evitar overfitting em clínicas com baixo volume em determinados códigos TUSS e garantir explicabilidade imediata sem dependência de GPU:

$$P(\text{glosa} \mid \text{op}, \text{tuss}) = \frac{\alpha_0 \cdot \hat{p}_{\text{global}} + \alpha_1 \cdot \hat{p}_{\text{op}} + \alpha_2 \cdot \hat{p}_{\text{tuss}} + n_{\text{op,tuss}} \cdot \hat{p}_{\text{op,tuss}}}{\alpha_0 + \alpha_1 + \alpha_2 + n_{\text{op,tuss}}}$$

- O score final da guia (`score_risco` $\in [0, 1]$, exposto em `AnalisarGuiaResponse.score_risco`) combina a maior probabilidade histórica entre os procedimentos da guia com o sinal dos alertas ativos na revisão atual:
  - Se houver alerta `bloqueante`: $\text{score\_risco} = \max(0.85, p_{\text{hist}})$ → `risco = "alto"`.
  - Se não houver alerta `bloqueante`, mas $p_{\text{hist}} \ge 0.65$: `risco = "alto"` (promovido pelo modelo preditivo!).
  - Se $0.30 \le \text{score\_risco} < 0.65$ ou houver alerta `aviso`: `risco = "medio"`.
  - Caso contrário: `risco = "baixo"`.
- Quando um par `(registro_ans, codigo_tuss)` possui histórico com $n \ge 3$ guias e taxa de glosa empírica $\ge 40\%$, o motor adiciona um alerta preventivo (`id="preditivo_glosa_elevada:<tuss>"`, severidade `aviso`, fonte `OPERADORA`) informando a taxa histórica e o principal código de motivo de glosa observado naquela operadora.

### 3.2 Backtest Temporal (`executar_backtest_temporal`)
Conforme o [[test-plan-ia|Test Plan §7]], o modelo é validado separando o histórico no tempo:
- **Treino:** guias com `data_atendimento <= data_corte` (meses antigos).
- **Teste:** guias com `data_atendimento > data_corte` (meses recentes).
- Métricas calculadas: **AUC-ROC**, **Brier Score** e **taxa de glosa no top-decil de risco vs. base**.

---

## 4. Auditoria de Contratos com Operadoras (`POST /v1/contratos/auditar`)

Compara a tabela contratual negociada com a operadora (extraída do contrato/aditivo da clínica) contra a tabela atualmente configurada no faturamento do PEP:

- **Divergências detectadas:**
  1. `valor_abaixo_do_contrato`: PEP está cobrando valor menor que o `valor_contratado` (receita deixada na mesa).
  2. `valor_acima_do_contrato`: PEP está cobrando valor maior que o `valor_contratado` (glosa financeira certa por divergência de tabela — código TUSS `1702`).
  3. `autorizacao_previa_nao_configurada`: contrato exige senha/autorização prévia para o TUSS, mas o PEP está configurado com `exige_autorizacao_previa = false` (E cria automaticamente a `RegraCustomizada` correspondente para a clínica!).
  4. `procedimento_ausente_no_pep`: procedimento coberto no contrato não está habilitado na tabela do convênio no PEP.

---

## 5. Detecção de Guias Esquecidas (`POST /v1/guias/esquecidas`)

Recebe o resumo dos atendimentos realizados na agenda do PEP em um período (`atendimento_ref`, `paciente_ref`, `data_atendimento`, `registro_ans`, `codigo_tuss`, `valor_estimado`) e cruza com as guias analisadas (`analises`) e os lotes enviados (`historico_guias_tiss`) da clínica:
- Retorna a lista de atendimentos sem guia correspondente (`guias_esquecidas`) e a **receita total em risco** (`valor_total_em_risco`), permitindo à faturista emitir as guias antes do fechamento do lote mensal.

---

## Critérios de aceite

- [x] Migração `0003_preditivo_f3` cria `historico_guias_tiss` e `contratos_operadora` de forma estritamente aditiva.
- [x] Parser XML TISS (`app/tiss_xml.py`) processa lotes de envio e demonstrativos de retorno nas versões `3.05.00`, `4.00.00` e `4.01.00`, ignorando PII do beneficiário.
- [x] Backtest temporal (`executar_backtest_temporal`) treina em meses antigos (`<= data_corte`) e atinge AUC-ROC $\ge 0.75$ nos meses recentes (`> data_corte`).
- [x] `POST /v1/guias/analisar` incorpora `score_risco` preditivo e emite alerta quando a operadora possui histórico elevado de glosa para o procedimento.
- [x] `POST /v1/contratos/auditar` detecta divergências de preço e autorização prévia e provisiona regras contratuais da clínica.
- [x] `POST /v1/guias/esquecidas` identifica atendimentos da agenda sem guia TISS correspondente e calcula a receita recuperável.
