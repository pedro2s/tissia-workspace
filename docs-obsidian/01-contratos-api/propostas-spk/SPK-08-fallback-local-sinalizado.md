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
origem: tissia/docs/spike/SPK-08-fallback-local-sinalizado.md
---

# SPK-08 — Fallback local da IA deve ser sinalizado (`degradado` / `origem`)

**Status:** Implementada na `tissia-ia` (`origem_ia` e `degradado` em `app/schemas.py`, `app/api/guias.py`, `app/ia/gemini.py` + testes em `tests/test_spk08_fallback_sinalizado.py`); falta integrar no `PEP` (aceitar `origem_ia` no schema Zod / `SpikeAdapter`, exibir selo granular na UI e atualizar `redundancia.spec.ts`) · **Necessária para:** transparência ao médico e ao faturista sobre o que foi gerado por LLM e o que veio de heurística; relatório do piloto (`taxa_degradado`) confiável

## Problema

Com o Gemini fora do ar (ou com erro de modelo/permissão), `POST /v1/guias/analisar` responde **200, `degradado: false`** e uma análise de aparência completa: risco, aviso de CID divergente, justificativa do Campo 23 e sugestão de CID. O conteúdo vem do fallback local, mas nada na resposta diz isso.

## Evidência (10/11/2026, E2E `redundancia.spec.ts` com `ESTADO_IA=gemini_quebrado`)

Spike no ar com `IA_PROVEDOR=vertex` e `GEMINI_MODEL=GEMINI_MODEL_QUALIDADE=modelo-inexistente`:

- Log da Spike: três `ia_indisponivel` (`ClientError`) por atendimento.
- Tabela `analises`: `decisor = gemini`, `modelo_geracao = modelo-inexistente@…`, `degradado = f`.
- UI do PEP: painel "Ativa" (sem selo "Parcial"), justificativa de modelo pronto ("Quadro clínico: … Solicita-se … (CID M232).") e CID sugerido S835, sob o rótulo "Sugestões geradas por IA de apoio".

Causa no código: `app/ia/gemini.py` captura `vertex.IaIndisponivel` e devolve o resultado local sem propagar (decisor, linha ~304; extração clínica, ~368; rascunho de justificativa, ~578). `degradado = True` em `app/api/guias.py` só ocorre quando a exceção **sobe** (linhas ~274, ~291, ~338), o que o fallback impede.

## Por que importa

- O PEP só mostra "Análise parcial" quando `degradado` é verdadeiro ou a Spike não responde. Aqui o médico lê texto de heurística como se fosse do LLM; a revisão humana continua obrigatória, mas o rótulo "IA de apoio" induz a confiar mais do que deveria.
- `relatorio_piloto.py` calcula `taxa_degradado` a partir da coluna `degradado`. Uma queda total do Gemini apareceria como 0 %.
- `modelo_geracao` grava o modelo **configurado**, não o que respondeu.

## Proposta (aditiva em `/v1`)

1. Cada etapa que cair no fallback local marca a resposta: `degradado: true` e um campo novo `origem_ia` por parte (`decisor`, `extracao`, `justificativa`) com `llm` | `local`.
2. `modelo_geracao` passa a registrar `local` (ou `modelo+local`) quando o fallback respondeu.
3. Manter o fallback: ele é o que deixa o atendimento seguir. Só a sinalização muda.

## Do lado do PEP (a fazer quando a Spike expuser o campo)

- Tratar `degradado: true` já existente como hoje ("Análise parcial"); mostrar na justificativa e nas sugestões o selo "gerado por regra local, sem IA".
- Não há mudança de contrato no PEP além de aceitar `origem_ia` (opcional, `.strict()` no schema de resposta exige adicioná-lo).
- Cobertura: `apps/e2e/specs-demo/redundancia.spec.ts` (`ESTADO_IA=gemini_quebrado`) hoje aceita a análise completa; passará a esperar o selo.

## Decisões a tomar

- `degradado` global ou só por parte? Recomendação: manter `degradado` como "qualquer parte caiu" (compatível com a UI e o relatório) e acrescentar `origem_ia` para o detalhe.
