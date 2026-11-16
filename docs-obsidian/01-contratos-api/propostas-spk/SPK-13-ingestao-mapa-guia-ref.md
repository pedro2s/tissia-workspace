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
aliases:
  - SPK-13
origem: tissia/docs/spike/SPK-11-ingestao-mapa-guia-ref.md
---

# SPK-13 — Mapa de `numeroGuiaPrestador` ↔ `guia_ref` na ingestão de lotes TISS

> **Nota de contexto (renumeração):** Proposta originalmente aberta como `SPK-11` na [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]] e renumerada para `SPK-13` para resolver a colisão de numeração com [[SPK-11-contratos-auditar-substituicao|SPK-11 (auditoria de contratos)]].

**Status:** Proposta · **Destino:** repo tissia-ai · **Necessária para:** [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]] (Q7), [[RFC-PEP-009-receita-esquecida|RFC-PEP-009]]

## Objetivo

Permitir que o PEP envie, junto com o XML sanitizado em `POST /v1/tiss/lotes/ingerir`, o mapa explícito entre `numeroGuiaPrestador` (do XML TISS) e o `guia_ref` estável da Spike (`gu_<uuid>`), para que o histórico de guias ingerido case corretamente com as análises (`analises_tissia_ai`) e com as consultas de receita esquecida.

## Motivação

Hoje `POST /v1/tiss/lotes/ingerir` recebe apenas `{ clinica_ref, xml_conteudo }` e tenta vincular desfechos pelo `numeroGuiaPrestador` extraído do XML. No PEP, o identificador opaco enviado à Spike nas análises é `guia_ref = "gu_" + guia_id`, que não coincide com `numero_prestador`. Como resultado, a ingestão do XML não se associa às análises prévias nem aos desfechos enviados via `POST /v1/desfechos`, além de não haver deduplicação caso um lote seja reenviado após timeout ([[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]], `tissia_ai_ingestao = incerto`).

## Contrato proposto

### Extensão em `POST /v1/tiss/lotes/ingerir`

```json
{
  "clinica_ref": "un_7f3a…",
  "xml_conteudo": "<ans:mensagemTISS>…</ans:mensagemTISS>",
  "lote_ref": "un_7f3a:retorno:<sha256>",
  "mapa_guias": [
    { "numero_guia_prestador": "G-004102", "guia_ref": "gu_91c2…" }
  ],
  "vincular_desfechos_automaticamente": false
}
```

- `mapa_guias` (opcional): lista de pares `{ numero_guia_prestador, guia_ref }`. Quando presente, a Spike associa os registros de `historico_guias_tiss` ao `guia_ref` informado em vez de usar `numeroGuiaPrestador` como `guia_ref`.
- `vincular_desfechos_automaticamente` (opcional, padrão `true` para compatibilidade): quando `false`, a ingestão alimenta apenas o histórico e deixa a atualização de desfechos a cargo exclusivo de `POST /v1/desfechos`.
- `lote_ref` (opcional): chave de idempotência para evitar duplicação em `historico_guias_tiss` quando o PEP reenviar após timeout (complementa a [[SPK-03-processamento-assincrono|SPK-03]]).

## Regras de privacidade

- `guia_ref` e `numero_guia_prestador` são identificadores operacionais do prestador; nenhuma tag ou dado de identificação do beneficiário é enviado ou aceito.
