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
  - 0003-gcp-sao-paulo
origem: tissia-ai/docs/adr/0003-gcp-sao-paulo.md
---

# ADR-0003 — GCP `southamerica-east1`

**Status:** Aceita — 29/10/2026 · parcialmente substituída pelo [[ADR-IA-0010-vertex-global-e-gemini-3x|ADR-0010]] (LLM e endpoint `global`)

## Contexto

Dado de saúde é sensível (LGPD Art. 11). Processar e armazenar no Brasil elimina a discussão de transferência internacional para a maior parte do fluxo. O Gemini no Vertex AI oferecia processamento regional em São Paulo para parte dos modelos. *(Superado pelo ADR-0010: os modelos 3.x só respondem no endpoint `global`, que passou a ser permitido.)*

## Decisão

| Componente | Serviço |
| :--- | :--- |
| API | Cloud Run (`southamerica-east1`), mínimo 1 instância em `prod` para evitar cold start |
| Banco | Cloud SQL Postgres 16 (`southamerica-east1`), backups automáticos + PITR |
| Segredos | Secret Manager |
| Imagens | Artifact Registry |
| LLM | Vertex AI — ver [[ADR-IA-0010-vertex-global-e-gemini-3x\|ADR-0010]] (endpoint `global`, Gemini 3.x) |
| Logs/métricas | Cloud Logging + Cloud Monitoring (logs JSON estruturados) |
| CI/CD | GitHub Actions + Workload Identity Federation (sem chave de service account) |

Ambientes: `staging` e `prod`, em projetos GCP separados.

## Consequências

- A infra é criada por um script `infra/bootstrap.sh` (`gcloud`) idempotente e documentado. `# ponytail:` Terraform quando houver mais de 2 ambientes ou mais de uma pessoa mexendo em infra.
- Criptografia em repouso com a chave padrão do Google. CMEK só se um contrato com operadora/hospital exigir.
- Se o PEP do Thiago estiver em outra cloud, a integração continua por HTTPS público com API key (ver ADR-0005); a latência entre clouds em SP é aceitável.
