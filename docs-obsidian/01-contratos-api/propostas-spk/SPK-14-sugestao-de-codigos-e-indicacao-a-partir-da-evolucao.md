---
projeto: ecossistema
tipo: spk
status: implementada
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
aliases:
  - SPK-14
origem: fatia B da rodada de correções da guia (sugestão proativa durante o atendimento)
---

# SPK-14 — Sugestão de CID, exames (TUSS) e indicação clínica a partir da evolução, sem guia em rascunho

**Status:** Implementada na `tissia-ia` (PR `pedro2s/tissia-ai#9`) · **Destino:** repo `tissia-ai` · **Necessária para:** sugestão proativa durante o atendimento no PEP ([[RFC-PEP-005-spike-adapter|RFC-PEP-005]], [[RFC-PEP-004-prontuario|RFC-PEP-004]]) e para atender também o atendimento **particular**

## Problema

A proposta do produto é auxiliar o médico **durante** o atendimento: a partir da evolução clínica (S/O/A/P), a IA sugere exames (TUSS), CID-10 e a indicação clínica (Campo 23), para que a guia já nasça correta. Hoje o único endpoint que devolve essas três coisas é `POST /v1/guias/analisar`, que foi desenhado para **analisar um rascunho de guia** (regras, DUT, score preditivo). Usá-lo para sugerir a partir da evolução funciona, mas com custos e atritos:

1. **Exige `operadora.registro_ans`** (`OperadoraInput.registro_ans`, 1–20 caracteres, obrigatório). Atendimento **particular** não tem operadora: o PEP não pode sugerir CID/exame/indicação para ele (e não deve inventar um código ANS).
2. **A justificativa (indicação) só é gerada com procedimentos**: `deve_justificar` exige `procedimentos_analisados` não vazio. Antes de o médico pedir qualquer exame não há indicação a sugerir. O PEP contorna com **duas chamadas**: 1ª com `procedimentos: []` e `sugerir_codigos: true` (devolve CID e TUSS); 2ª com os TUSS sugeridos em `procedimentos` e `gerar_justificativa: true` (devolve `justificativa_sugerida`). A 2ª chamada executa a avaliação de DUT dos TUSS sugeridos (decisor/LLM) só para descartar o resultado.
3. **Trabalho inútil por chamada**: regras determinísticas, score preditivo e alertas (sempre `TISS-SADT-INDICACAO` etc., porque a "guia" está vazia) são calculados e descartados.
4. **Efeitos colaterais indesejados**: cada chamada grava uma `Analise` e conta no uso mensal como análise de guia. O PEP usa `guia_ref = sg_<atendimento>` (prefixo próprio) para que desfechos associados pelo par (`clinica_ref`, `guia_ref`) nunca caiam numa análise de sugestão, mas a linha existe e polui indicadores de "análises por guia".
5. **Sem feedback**: a decisão do médico (aceitou, editou, ignorou) sobre uma sugestão proativa não tem como voltar ao motor, porque o PEP não persiste a sugestão (não há `analise_id` estável do lado do PEP).

## Proposta (aditiva em `/v1`)

`POST /v1/guias/sugerir` (nome alternativo: `/v1/consultas/sugerir`), `extra="forbid"`:

```json
{
  "clinica_ref": "un_<uuid>",
  "atendimento_ref": "at_<uuid>",
  "operadora": { "registro_ans": "326305" },
  "paciente": { "ref": "pa_...", "sexo": "F", "idade": 35 },
  "evolucao_clinica": "…",
  "cids": ["M232"],
  "procedimentos": [{ "codigo_tuss": "41101316", "quantidade": 1 }],
  "opcoes": { "sugerir_cids": true, "sugerir_exames": true, "sugerir_indicacao": true }
}
```

- **`operadora` opcional** (e, portanto, também sem `registro_ans`): a sugestão de CID, exames e indicação não depende da operadora. Regras e DUT dependem; nesta rota não rodam.
- **`cids` e `procedimentos` opcionais**: quando informados, servem só para marcar `situacao` (`ja_informado`, `ausente_na_guia`, `diverge_do_informado`) e para evitar repetição; não disparam validação de DUT.
- **Resposta**:

```json
{
  "sugestao_id": "<uuid>",
  "versao_bases": "…",
  "degradado": false,
  "origem_ia": { "extracao": "llm", "decisor": "llm", "justificativa": "llm" },
  "cids": [{ "codigo": "M232", "descricao": "…", "confianca": 0.82, "evidencia": "McMurray positivo", "situacao": "ausente_na_guia" }],
  "procedimentos": [{ "codigo_tuss": "41101316", "termo": "…", "confianca": 0.74, "evidencia": "…", "situacao": "ausente_na_guia" }],
  "indicacao_clinica": { "texto": "…", "procedimentos_cobertos": ["41101316"], "gerada_por_ia": true },
  "aviso_legal": "…"
}
```

- **A indicação é gerada com ou sem procedimentos informados**: sem eles, cobre os exames que a própria sugestão devolve (ordem: extrair → recuperar/escolher TUSS e CID → redigir a indicação para esses TUSS), em **uma** chamada. Mesma validação `ADR-IA-0008` (nunca afirmar critério ausente da evolução) e mesmo limite do Campo 23 (parâmetro opcional `limite_caracteres`, padrão 500, para o texto já nascer dentro do que o PEP aceita).
- **Mesmos invariantes de hoje**: scrubber de PII sobre `evolucao_clinica` (`app/pii.py`), nenhum código fora de `data/` (SHA-256 em `data/MANIFEST.yaml`), falha ou timeout de IA responde `200` com `degradado: true` (nunca `5xx`), `X-Tissia-Tempos-Ms`.
- **Persistência separada**: grava em tabela própria (`sugestoes`), fora de `analises`, e não conta como análise de guia no uso mensal (conta em métrica própria).
- **`POST /v1/guias/sugerir/{sugestao_id}/feedback`** (paralelo a `/v1/analises/{id}/feedback` e a `/v1/consultas/estruturacoes/{id}/feedback`, [[SPK-04-requisitos-do-scribe|SPK-04]]), por item (`cid:<codigo>`, `exame:<tuss>`, `indicacao`) com `aceito | editado | rejeitado` e `texto_final`.
- **Latência-alvo**: p95 ≤ 3 s sem a indicação e ≤ 5 s com ela (sem DUT, regras ou preditivo no caminho); o PEP chama com timeout de 6 s e debounce.

## Do lado do PEP (já implementado com o que existe hoje)

`POST /api/v1/atendimentos/:id/sugestoes-ia` (módulo `prontuario`, `SpikeAdapter.sugerir` em `modulos/tissia-ai/`), a partir da **evolução salva no servidor** (nunca texto vindo do cliente), só com `ref`, sexo e idade do paciente, só no modo `assistido`, sem gravar nada. Etapa 1 = códigos; etapa 2 = indicação para até 3 TUSS sugeridos. Quando esta SPK existir, o `SpikeAdapter.sugerir` troca a rota e as duas chamadas viram uma, sem mudar o contrato do PEP com a UI; atendimento particular passa a receber as mesmas sugestões.

## Decisões a tomar (time da `tissia-ai`)

- Rota nova em `/v1/guias` ou `/v1/consultas` (a sugestão é anterior à guia; a semelhança com `/v1/consultas/estruturar` é só no insumo, texto da consulta).
- Aceitar o `evolucao_clinica` com o mesmo limite de 20.000 caracteres e o corpo com o mesmo teto de 64 KB.
- Se o feedback entra na mesma tabela de `decisoes` ou em tabela própria (o BI do PEP lê `analises` e `decisoes`).

## Notas de implementação

- Implementada em `tissia-ia` (`feat/spk-14-guias-sugerir`, PR `pedro2s/tissia-ai#9`):
  - Endpoints `POST /v1/guias/sugerir` (`200 OK`) e `POST /v1/guias/sugerir/{sugestao_id}/feedback` (`204 No Content`).
  - Suporta `operadora` opcional (atende atendimentos particulares e convênios sem inventar `registro_ans`), scrubber de PII em `evolucao_clinica`, validação de códigos contra `data/` (`ADR-IA-0008`), `metadados_ia` (`ADR-IA-0010`) e persistência separada de `analises`.
  - Suíte `pytest` (`192 passed`) e `ruff check .` / `ruff format --check .` verdes.
