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
  - RFC-007-piloto-e-desfecho
origem: tissia-ai/docs/rfc/RFC-007-piloto-e-desfecho.md
---

# RFC-007 — Piloto fechado, desfecho de glosa e relatório de impacto

**Fase:** F2 · **Status:** Concluída · **ADRs:** 0005, 0007, 0008, 0009

## Objetivo

Operar o piloto fechado nas clínicas-piloto da base do integrador (primeiro em `modo=sombra`, depois em `modo=visivel`), fechar o ciclo de aprendizado com o retorno real de faturamento das operadoras (**desfecho de glosa** individual e em lote), medir a **autocorreção na origem** via `guia_ref` e gerar o **relatório executivo do piloto** (Markdown e CSV) comprovando redução de glosa e precisão clínica.

---

## 1. Modelo de Dados (Migração Aditiva `0002_piloto_f2`)

Seguindo a diretriz de migrações estritamente aditivas na F1/F2 ([[release-plan-ia|Release Plan §6]]), são adicionadas duas tabelas:

```sql
CREATE TABLE desfechos (
  id                  uuid PRIMARY KEY,
  analise_id          uuid NOT NULL UNIQUE REFERENCES analises(id),
  clinica_id          uuid NOT NULL REFERENCES clinicas(id),
  guia_ref            text,
  lote_ref            text,
  status              text NOT NULL,         -- pago | glosado_parcial | glosado_total
  valor_cobrado       numeric(12, 2),
  valor_glosado       numeric(12, 2),
  codigo_glosa_tuss   text,                  -- código da tabela 38 TUSS (ex.: 1706, 1429)
  motivo_glosa_texto  text,                  -- sempre passado por scrub() antes de gravar
  created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_desfechos_clinica_created ON desfechos (clinica_id, created_at);

CREATE TABLE regras_customizadas (
  id               uuid PRIMARY KEY,
  clinica_id       uuid REFERENCES clinicas(id), -- NULL = vale para todas as clínicas do integrador
  integrador_id    uuid NOT NULL REFERENCES integradores(id),
  codigo_regra     text NOT NULL,
  tipo             text NOT NULL,            -- conjunto fechado de 8 tipos (ADR-0009)
  fonte            text NOT NULL,            -- OPERADORA:<ans> | BOA_PRATICA | ANS_TISS
  severidade       text NOT NULL,            -- bloqueante | aviso | info
  inicio_vigencia  date NOT NULL,
  fim_vigencia     date,
  mensagem         text NOT NULL,
  params           jsonb NOT NULL DEFAULT '{}',
  ativo            boolean NOT NULL DEFAULT true,
  created_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_regras_customizadas_integrador_clinica ON regras_customizadas (integrador_id, clinica_id);
```

---

## 2. Endpoints de Desfecho de Glosa

### 2.1 Individual: `POST /v1/analises/{analise_id}/desfecho`

Permite registrar (ou atualizar idempotentemente) o resultado do faturamento de uma análise específica após o retorno da operadora.

```json
{
  "status": "glosado_parcial",
  "valor_cobrado": 850.00,
  "valor_glosado": 320.00,
  "codigo_glosa_tuss": "1706",
  "motivo_glosa_texto": "Ausência de descrição de falha do tratamento conservador prévio.",
  "lote_ref": "ret-unimed-2026-11"
}
```

- **Respostas:**
  - `200 OK`: `{ "desfecho_id": "...", "analise_id": "...", "status": "glosado_parcial" }`
  - `404 Not Found`: se `analise_id` não existir ou pertencer a outro integrador.
- `motivo_glosa_texto` passa obrigatoriamente por `app/pii.py:scrub()` antes de ser persistido.

### 2.2 Em lote: `POST /v1/desfechos`

Como o retorno de faturamento das operadoras (demonstrativo de análise de conta / XML TISS de retorno) chega por lote referenciando o número da guia no PEP (`guia_ref`), este endpoint aceita até 1.000 desfechos por chamada, casando cada item por `analise_id` **ou** pelo par `(clinica_ref, guia_ref)`:

```json
{
  "lote_ref": "ret-operadora-2026-11-01",
  "itens": [
    {
      "clinica_ref": "clinica-123",
      "guia_ref": "guia-pep-9876",
      "status": "pago",
      "valor_cobrado": 650.00,
      "valor_glosado": 0.00
    },
    {
      "analise_id": "0b4c1d2e-3f4a-5b6c-7d8e-9f0a1b2c3d4e",
      "status": "glosado_total",
      "valor_cobrado": 480.00,
      "valor_glosado": 480.00,
      "codigo_glosa_tuss": "1429"
    }
  ]
}
```

- Quando o item usa `(clinica_ref, guia_ref)`, o sistema vincula o desfecho à **última análise registrada** (`ORDER BY created_at DESC, id DESC`) para aquela guia naquela clínica do integrador.
- **Resposta `200 OK`:**
  ```json
  {
    "processados": 2,
    "nao_encontrados": []
  }
  ```
  Itens cujo `analise_id` ou `(clinica_ref, guia_ref)` não forem localizados para o integrador não abortam o lote: são listados em `nao_encontrados`.

---

## 3. Evoluções no Motor para o Piloto (`POST /v1/guias/analisar`)

1. **Suporte a `tipo_guia: "internacao"` (Solicitação de Internação):**
   - Além de `"consulta"` e `"sp_sadt"`, o endpoint passa a aceitar `"internacao"`, aplicando validação DUT, sugestão de códigos e geração de justificativa clínica consolidada.
2. **Regras customizadas por clínica/operadora no Postgres (`regras_customizadas`):**
   - Durante a análise da guia, as regras ativas do integrador/clínica em `regras_customizadas` são validadas pelo mesmo schema declarativo ([[ADR-IA-0009-regras-declarativas|ADR-0009]]) e avaliadas em conjunto com as regras estáticas de `data/regras/*.yaml`.

---

## 4. Métricas do Piloto e Relatório (`scripts/relatorio_piloto.py`)

O módulo `app/relatorio_piloto.py` e o script CLI `scripts/relatorio_piloto.py` calculam e exportam (em Markdown ou CSV):

1. **Autocorreção na origem (via `guia_ref`):**
   - Proporção de guias (`guia_ref`) em `modo=visivel` cuja **primeira análise** continha ao menos um alerta `bloqueante` e cuja **última análise** (após edição pela pessoa faturista/médica antes do envio) teve os alertas bloqueantes eliminados.
2. **Comparativo de Glosa (Baseline × Modo Sombra × Modo Visível):**
   - Taxa de glosa histórica da clínica (`clinicas.flags.baseline_taxa_glosa`, quando informada), taxa de glosa observada nas guias em `sombra` e taxa de glosa nas guias em `visivel`, além do valor financeiro cobrado vs. glosado.
3. **Valor Preditivo no Modo Sombra:**
   - Entre as guias analisadas em `modo=sombra` nas quais o motor teria emitido alerta `bloqueante`, qual percentual efetivamente sofreu glosa (`glosado_parcial` ou `glosado_total`) no retorno da operadora.
4. **Comparação Causal no Mesmo Período (Alerta Aceito × Rejeitado):**
   - Taxa de glosa em guias onde os alertas foram aceitos/corrigidos no feedback versus guias onde alertas bloqueantes foram rejeitados/ignorados.
5. **Métricas de Qualidade e Operação:**
   - Precisão dos alertas por `tipo` (`aceito + editado / total`), taxa de aceitação de justificativa (`aceito + editado / total`), latência `p50`/`p95` e taxa de `degradado`.

---

## 5. Gestão de Feature Flags de Clínica (`scripts/configurar_clinica.py`)

Script CLI para provisionar clínicas-piloto e atualizar `clinicas.flags` (`modo=sombra|visivel`, `gerar_justificativa`, `sugerir_codigos`, `decisor`, `guardar_entrada_para_avaliacao`, `baseline_taxa_glosa`).

---

## Critérios de aceite

- [x] Migração `0002_piloto_f2` cria `desfechos` e `regras_customizadas` de forma estritamente aditiva.
- [x] `POST /v1/analises/{id}/desfecho` grava/atualiza desfecho com scrubber no texto livre e retorna `404` para análise de outro integrador.
- [x] `POST /v1/desfechos` processa lotes casando tanto por `analise_id` quanto pela última análise de `(clinica_ref, guia_ref)`, reportando itens `nao_encontrados` sem abortar o lote.
- [x] Cálculo de **autocorreção na origem** identifica guias (`guia_ref`) que tinham alerta bloqueante na 1ª revisão e saíram sem alerta bloqueante na revisão final.
- [x] Regras salvas em `regras_customizadas` no Postgres são avaliadas apenas para a clínica/integrador correspondente.
- [x] `scripts/relatorio_piloto.py` gera relatório completo em Markdown e CSV com todas as métricas do piloto.
