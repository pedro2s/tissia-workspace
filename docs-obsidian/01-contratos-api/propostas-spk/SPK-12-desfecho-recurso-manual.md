---
projeto: ecossistema
tipo: spk
status: proposta
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-12-desfecho-recurso-manual.md
---

# SPK-12 — Desfecho de recurso manual sem sobrescrever o desfecho original

**Status:** Proposta · **Destino:** repo tissia-ai · **Necessária para:** [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]] (Q8)

## Objetivo

Permitir registrar na Spike o desfecho de um recurso de glosa criado manualmente no PEP (sem `recurso_id` gerado pela [[SPK-02-recurso-de-glosa|SPK-02]]), preservando o registro original da glosa para o modelo preditivo.

## Motivação

Na Spike, `POST /v1/analises/{id}/desfecho` realiza um upsert por análise (`_upsert_desfecho`). Se o PEP enviar o resultado de um recurso revertido por esse endpoint, o registro da glosa original é sobrescrito por `pago`, apagando o sinal histórico de que a guia foi glosada antes da reversão. A [[SPK-02-recurso-de-glosa|SPK-02]] prevê `POST /v1/glosas/recurso/{id}/desfecho` apenas para recursos com `tissia_ai_recurso_id`, deixando os recursos manuais sem canal próprio.

## Contrato proposto

### `POST /v1/glosas/recurso-manual/desfecho`

```json
{
  "clinica_ref": "un_7f3a…",
  "analise_id": "8f14e45f-ceea-467f-a8c9-3189f4e2109b",
  "guia_ref": "gu_91c2…",
  "recurso_ref": "rec_019a…",
  "resultado": "revertido_total",
  "valor_recuperado": 320.00,
  "codigo_glosa_tuss": "<T38_…>",
  "registrado_em": "2026-11-15T10:00:00Z"
}
```

- `resultado`: `revertido_total` | `revertido_parcial` | `mantido`.
- `recurso_ref`: identificador estável do recurso no PEP para idempotência (`Idempotency-Key` / upsert por `recurso_ref`).
- A Spike grava o desfecho do recurso em tabela própria de reversões vinculada à análise/guia, **sem** alterar o status original de glosa em `desfechos`.

### Response 204

Sem corpo; chamadas repetidas com o mesmo `recurso_ref` são idempotentes.

