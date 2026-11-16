---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0009-hospedagem-gcp-sao-paulo.md
---

# ADR-PEP-0009 — Hospedagem GCP `southamerica-east1` (mesma região da Spike)

**Status:** Aceita — 07/11/2026

## Contexto

Dado de saúde é sensível (LGPD Art. 11); manter processamento e armazenamento no Brasil evita discussão de transferência internacional. A Spike já roda em GCP `southamerica-east1` ([[ADR-IA-0003-gcp-sao-paulo|Spike ADR-0003]]): mesma região reduz latência no `analisar` (orçamento de 6 s) e reaproveita conhecimento operacional. Hospedagem e residência de dados é a decisão em aberto nº 4 do design de produto.

## Decisão

| Componente | Serviço |
| :--- | :--- |
| API e workers do PEP | Cloud Run (`southamerica-east1`), mínimo 1 instância em `prod`. GKE Autopilot só se workers longos ou WebSocket (voz, P2) não couberem no Cloud Run |
| Banco | Cloud SQL PostgreSQL 16, HA regional, backups automáticos + PITR (7 dias) e export diário para bucket com retenção |
| Fila/cache | Memorystore Redis (BullMQ, [[ADR-PEP-0008-eventos-outbox-bullmq\|ADR-PEP-0008]]) |
| Anexos e documentos | Cloud Storage, bucket regional, versionamento; **CMEK opcional** por exigência contratual |
| UI | Cloud Storage + Cloud CDN (estático) |
| Segredos | Secret Manager (inclui API key da Spike e salt de pseudonimização) |
| Identidade | Keycloak em Cloud Run ou GKE ([[ADR-PEP-0010-identidade-keycloak\|ADR-PEP-0010]]) |
| Logs/métricas | Cloud Logging + Monitoring, logs JSON sem PII |

- Ambientes `staging` e `prod` em projetos GCP separados; infra como código (Terraform) desde o P0, por serem mais componentes que a Spike.
- Backup cross-region **dentro do Brasil** não existe hoje no GCP **[validar]**; cópia de backup fora do país só com base legal e criptografia.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| AWS `sa-east-1` | Equivalente técnico; perde a sinergia operacional com a Spike |
| Azure Brazil South | Idem |
| On-premise / colocation | Custo e operação incompatíveis com a equipe |

## Consequências

- Custo base maior que a Spike (Cloud SQL HA, Redis, Keycloak).
- DR: RPO ≤ 5 min (PITR), RTO alvo ≤ 4 h; testar restauração a cada trimestre ([[release-plan-pep|release plan]]).
