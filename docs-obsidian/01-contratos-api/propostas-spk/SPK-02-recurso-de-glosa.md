---
projeto: ecossistema
tipo: spk
status: ativo
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-02-recurso-de-glosa.md
---

# SPK-02 — Recurso de glosa assistido

**Status:** Implementada na `tissia-ia` (`app/api/glosas.py` + testes em `tests/test_spk02_recurso_glosa.py`: `POST /v1/glosas/recurso`, `/recurso/{id}/desfecho` e `/recurso/{id}/feedback`); falta integrar no `PEP` (tela 20, `tissia-ai.client.ts` e `retorno.controller.ts` em [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]]) · **Necessária para:** tela 20 (Central de glosas e recurso), [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]], P2

## Objetivo

Gerar um texto de recurso de glosa ancorado nas evidências da evolução, no motivo da glosa (Tabela 38) e no histórico de recursos aceitos pela mesma operadora, sem afirmar fato clínico ausente.

## Motivação

O contrato `v1` cobre a prevenção (`analisar`) e o registro do desfecho (`desfechos`), mas não a contestação. O design (tela 20) promete "recurso assistido por IA com evidências", e o loop de desfechos da Spike já sabe quais argumentos passam em cada operadora. Hoje esse conhecimento não volta para o usuário.

## Contrato proposto

### `POST /v1/glosas/recurso`

```json
{
  "clinica_ref": "un_7f3a…",
  "analise_id": "8f14e45f-ceea-467f-a8c9-3189f4e2109b",
  "guia_ref": "gu_91c2…",
  "operadora": { "registro_ans": "005711" },
  "itens_glosados": [
    { "codigo_tuss": "<TUSS_…>", "valor_glosado": 320.00, "codigo_glosa_tuss": "<T38_…>", "motivo_glosa_texto": "Ausência de detalhamento de tratamento conservador prévio." }
  ],
  "evolucao_clinica": "…",
  "documentos_anexos": ["laudo_radiografia", "termo_fisioterapia"],
  "limite_caracteres": 2000
}
```

- `analise_id` ou o par (`clinica_ref`, `guia_ref`) identifica a guia; a Spike usa a última análise anterior ao envio.
- `evolucao_clinica` é obrigatória: a evolução não é guardada na Spike (salvo opt-in) e o recurso precisa citá-la.
- `documentos_anexos`: rótulos dos anexos que o PEP vai juntar ao recurso (sem conteúdo). Servem para o texto referenciá-los.
- Limites: corpo ≤ 64 KB; evolução ≤ 20.000; até 50 itens.

### Response 200

```json
{
  "recurso_id": "c1d2…",
  "versao_bases": "2026.04.1",
  "degradado": false,
  "itens": [
    {
      "codigo_tuss": "<TUSS_…>",
      "argumento": {
        "texto": "Paciente realizou fisioterapia por 8 semanas sem melhora, conforme evolução de 02/09…",
        "gerada_por_ia": true
      },
      "evidencias": [
        { "trecho": "Fez fisioterapia por 8 semanas sem melhora significativa.", "origem": "evolucao" }
      ],
      "criterios_nao_documentados": [
        { "texto": "Tempo mínimo de tratamento conservador", "orientacao": "Não consta na evolução. Recurso provavelmente não reverte sem complemento documental." }
      ],
      "chance_reversao": 0.58,
      "base_historica": { "recursos_similares": 14, "aceitos": 8 }
    }
  ],
  "aviso_legal": "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."
}
```

- `chance_reversao` só aparece com pelo menos 10 recursos similares na base (mesma operadora, TUSS e código de glosa); senão `null`. Similares = desfechos de recursos registrados (abaixo).
- Falha de IA → `200` com `degradado: true`, `argumento: null` e as evidências encontradas por busca lexical.

### Desfecho do recurso

`POST /v1/glosas/recurso/{recurso_id}/desfecho` com `{"status": "revertido_total" | "revertido_parcial" | "mantido", "valor_recuperado": 320.00}`. Alimenta `base_historica`.

### Feedback

Reusa `POST /v1/analises/{id}/feedback`? Não: o recurso tem id próprio. Proposta: `POST /v1/glosas/recurso/{recurso_id}/feedback` com `item_id = "<codigo_tuss>:recurso"` e a mesma semântica de `acao`/`texto_final`.

## Impacto no modelo de dados da Spike

Migração aditiva: `recursos` (id, integrador, clínica, analise_id, guia_ref, registro_ans, resposta jsonb, versões, latência, criado_em), `recursos_desfechos`, `recursos_feedbacks`. Índice por `(integrador_id, registro_ans, codigo_tuss, codigo_glosa_tuss)`.

## LGPD e segurança

- Evolução passa pelo scrubber antes da IA e não é gravada (mesma regra da ADR-0007 da Spike, incluindo o opt-in).
- `motivo_glosa_texto` passa pelo scrubber antes de gravar.
- Spike ADR-0008: o argumento só usa fatos presentes na evolução ou nos rótulos de anexos. Critério ausente vira `criterios_nao_documentados`, nunca texto afirmativo. Mesma checagem de `fatos_usados` da justificativa (Spike RFC-004).

## Compatibilidade

Rotas novas em `/v1`; nada muda nas existentes.

## Critérios de aceite

- [ ] Argumento sem fato não suportado: 0 no golden set de recursos (casos sintéticos + revisão do CMO).
- [ ] Critério ausente nunca aparece no texto do argumento (teste de propriedade com stubs).
- [ ] `chance_reversao = null` abaixo de 10 similares.
- [ ] Degradação: IA fora → 200 com `degradado: true` e evidências lexicais.
- [ ] Isolamento por integrador (404 para `analise_id` de outro integrador).
- [ ] Revisão de um consultor de faturamento em 20 recursos reais antes de liberar em `prod`.

## Perguntas abertas

1. Os prazos de recurso por operadora (para alertar vencimento) ficam no PEP (contrato) ou na Spike? Proposta: no PEP.
2. O recurso via XML TISS (`recursoGlosa` **[validar nome da mensagem]**) é gerado pelo PEP; a Spike só entrega texto. Confirmar.
3. Precisamos do texto do laudo anexado (OCR) para evidências? Fora do escopo desta proposta.
