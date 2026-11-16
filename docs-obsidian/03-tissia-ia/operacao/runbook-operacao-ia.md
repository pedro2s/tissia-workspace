---
projeto: tissia-ia
tipo: operacao
status: ativo
tags:
  - tissia
  - tissia-ia
  - operacao
  - runbook
  - sre
aliases:
  - runbook-operacao
origem: tissia-ai/docs/runbook-operacao.md
---

# Runbook de Operação, Rollout F2 e Rollback — Tissia IA

Guia prático de operação em produção/homologação baseado no [[release-plan-ia|Release Plan]].

## 1. Rollout do Piloto (F2)

### 1.1 Provisionar clínica em `modo=sombra`
```bash
uv run python scripts/configurar_clinica.py \
  --integrador "<nome_integrador>" \
  --clinica-ref "<clinica_ref>" \
  --nome "<Nome Amigável>" \
  --modo sombra \
  --guardar-entrada false \
  --baseline-glosa 0.14
```

### 1.2 Listar clínicas e flags de um integrador
```bash
uv run python scripts/configurar_clinica.py \
  --integrador "<nome_integrador>" \
  --listar
```

### 1.3 Promover clínica para `modo=visivel` (após Gate de sombra)
```bash
uv run python scripts/configurar_clinica.py \
  --integrador "<nome_integrador>" \
  --clinica-ref "<clinica_ref>" \
  --modo visivel
```

## 2. Playbooks de Rollback (< 5 min)

### 2.1 Bug de código em `prod` (Cloud Run)
```bash
gcloud run revisions list --service=tissia-ai-api --region=southamerica-east1
gcloud run services update-traffic tissia-ai-api \
  --region=southamerica-east1 \
  --to-revisions=<REVISAO_ANTERIOR>=100
```

### 2.2 Modelo de IA degradado (latência, custo ou qualidade)
Desligar `gerar_justificativa` e/ou `sugerir_codigos` globalmente sem redeploy de código:
```bash
gcloud run services update tissia-ai-api \
  --region=southamerica-east1 \
  --update-env-vars="IA_GERAR_JUSTIFICATIVA_HABILITADO=false,IA_SUGERIR_CODIGOS_HABILITADO=false"
```
Para reabilitar após normalização:
```bash
gcloud run services update tissia-ai-api \
  --region=southamerica-east1 \
  --update-env-vars="IA_GERAR_JUSTIFICATIVA_HABILITADO=true,IA_SUGERIR_CODIGOS_HABILITADO=true"
```

### 2.3 Alerta clinicamente errado reportado em uma clínica
1. Retornar imediatamente a clínica para `modo=sombra`:
   ```bash
   uv run python scripts/configurar_clinica.py \
     --integrador "<nome_integrador>" \
     --clinica-ref "<clinica_ref>" \
     --modo sombra
   ```
2. Registrar post-mortem usando [[TEMPLATE-post-mortem|TEMPLATE-post-mortem.md]].
3. Adicionar o caso anonimizado como regressão no golden set (`evals/casos/`) ou `tests/` antes de reativar `visivel`.

## 3. Relatório Semanal e Final do Piloto (RFC-007)
```bash
uv run python scripts/relatorio_piloto.py --help
```

## 4. Vigência ANS (TISS e Rol)
- O workflow `.github/workflows/vigencia-ans.yml` executa diariamente às `06:00 BRT` (`09:00 UTC`).
- Execução manual:
  ```bash
  uv run python scripts/verificar_vigencia_ans.py --formato json
  ```
- Código de saída `2` indica nova competência publicada na ANS; revisar `data/MANIFEST.yaml` e executar ingestão controlada antes de gerar nova tag.
