---
projeto: tissia-ia
tipo: operacao
status: template
tags:
  - tissia
  - tissia-ia
  - operacao
  - evals
  - llm
origem: tissia-ai/docs/evals/TEMPLATE-revisao-cmo.md
---

# Revisão Humana Pré-Piloto (Gate F1 -> F2) — AAAA-MM-DD

> Preencher e salvar uma cópia como `docs/evals/AAAA-MM-DD-revisao-cmo.md` antes de promover o piloto visível (F2), conforme `docs/test-plan.md §5` e `docs/release-plan.md §3`.

## 1. Metadados da Revisão

- **Data:** `AAAA-MM-DD`
- **Ambiente avaliado:** `staging`
- **Versão/Commit:** `<git sha>`
- **Competência ANS (`GET /v1/bases/versoes`):** `202609` (TISS `04.03.00`)
- **CMO responsável:** `<Nome / CRM>`
- **Faturista responsável:** `<Nome>`

## 2. Checklist de Pré-Requisitos Legais e Operacionais (Gate F1 -> F2)

- [ ] Termos de uso e política de privacidade (B2B + LGPD) revisados e assinados
- [ ] Cláusula contratual de apoio à decisão / responsabilidade médica aprovada pelo jurídico
- [ ] Canal de suporte clínico/faturamento definido para clínicas piloto
- [ ] Testes de carga em `staging` executados (`locust` 5 req/s por 5 min, p95 <= 6s, 0% 5xx)
- [ ] Uptime check (`/healthz`) e alertas do Cloud Monitoring aplicados (`infra/monitoring.sh`)

## 3. Amostra CMO (50 casos)

Critérios obrigatórios:
- **Fato inventado em justificativa:** `0 / 50` (0% tolerância)
- **Código TUSS ou CID inexistente sugerido:** `0 / 50` (0% tolerância)
- **Aprovação clínica das justificativas e classificações DUT:** `>= 90%` (`>= 45 / 50`)

| # | Caso / Especialidade | Decisão DUT adequada? | Fato inventado? (S/N) | Código inexistente? (S/N) | Justificativa aprovada? (S/N) | Observações |
|---|----------------------|-----------------------|----------------------|---------------------------|-------------------------------|-------------|
| 1 |                      |                       |                      |                           |                               |             |

- **Total aprovado pelo CMO:** `___ / 50` (`___%`)
- **Fatos inventados:** `___ / 50`
- **Códigos inexistentes:** `___ / 50`

## 4. Amostra Faturista (20 casos)

Critério: verificação de utilidade prática dos alertas TISS/TUSS, coerência de porte/rol e ausência de ruído excessivo.

| # | Caso / Guia | Alertas úteis? (S/N) | Falsos positivos críticos? | Observações |
|---|-------------|----------------------|----------------------------|-------------|
| 1 |             |                      |                            |             |

- **Avaliação geral do faturista:** `<Aprovado / Ajustes necessários>`

## 5. Parecer Final do Gate F1 -> F2

- **Decisão:** `[ ] Aprovado para F2 (piloto visível)` / `[ ] Reprovado (listar bloqueios abaixo)`
- **Observações finais:**
