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
origem: tissia/docs/spike/SPK-07-latencia-da-analise.md
---

# SPK-07 — Latência da análise com IA (p95 ≤ 5 s)

**Status:** Implementada na `tissia-ia` (paralelização de DUT e Sugestão via `asyncio.gather` e exposição do header `X-Tissia-Tempos-Ms` em `app/api/guias.py`) e mitigada no `PEP` (análise em duas fases + `pnpm --filter @pep/api medir:latencia`); falta no `PEP` ler/registrar `X-Tissia-Tempos-Ms` e validar o p95 em produção com IA real (decidindo se a resposta progressiva ainda será necessária) · **Necessária para:** gate de saída do Marco D ([[release-plan-pep|release plan §12.8]]): "análise com IA em ≤ 5 s p95", e para a experiência do prontuário (meta de revisão da guia em ~15 s, tela 13)

## Objetivo

Trazer o p95 de `POST /v1/guias/analisar` para ≤ 5 s com IA real, sem perder a justificativa do Campo 23 nem as sugestões de código.

## Medição (09/11/2026, `pnpm --filter @pep/api medir:latencia`)

Mesmo adapter do PEP, Spike local com `IA_PROVEDOR=vertex`, `GEMINI_MODEL=gemini-3.5-flash-lite`, `IA_TIMEOUT_S=10.0`, casos sintéticos de Ortopedia (RM + artroscopia) e Cardiologia (MAPA), `guia_ref` novo a cada rodada, sequencial.

| Opções enviadas | n | p50 | p95 | máx |
| :--- | :--- | :--- | :--- | :--- |
| completas (`gerar_justificativa` e `sugerir_codigos` ligados) | 12 | 6,5 s | **9,2 s** | 9,2 s |
| básicas (regras TISS/ANS + checklist DUT, sem justificativa nem sugestões) | 8 | 2,0 s | **2,6 s** | 2,6 s |

Conclusões: o gate **não** é atendido com as opções completas; as regras e o checklist da DUT cabem folgados no orçamento; a justificativa e as sugestões de código somam cerca de 4 s. Uma amostra de 10,0 s encosta no `IA_TIMEOUT_S` e vira análise `degradada`.

## Proposta (aditiva em `/v1`)

1. **Paralelizar as chamadas ao modelo** dentro da análise: DUT, justificativa e sugestão de códigos hoje parecem rodar em sequência (a soma dos tempos acima é compatível com isso). Em paralelo o tempo tende ao da mais lenta.
2. **Resposta progressiva** (alternativa ou complemento): devolver `risco`, `alertas` e `procedimentos[].dut` em ≤ 3 s com `justificativa_sugerida` e `sugestoes_codigos` marcados como `pendentes`, e liberar o restante em `GET /v1/guias/analises/{id}` (ou SSE), com o MESMO `analise_id` para o feedback ([[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]]). O PEP já mostra o painel por partes (aba Anti-glosa primeiro, Sugestões depois).
3. Expor `latencia_ms` por etapa no `aviso`/cabeçalho de resposta, para medir p95 por etapa no ambiente de demo.

## Do lado do PEP (já feito)

- **Análise em duas fases** (09/11/2026): o PEP chama `analisar` primeiro só com regras + DUT (`gerar_justificativa` e `sugerir_codigos` desligados; p95 2,6 s) e, em segundo plano, com tudo ligado (p95 7,6 s), atualizando a mesma análise. O gate de 5 s passa a valer para a 1ª fase. O custo: duas chamadas ao modelo por análise e a DUT é calculada duas vezes; a proposta (1) acima (paralelizar dentro da Spike) ou (2) (resposta progressiva com o mesmo `analise_id`) eliminaria essa duplicação.

- Medição reprodutível: `pnpm --filter @pep/api medir:latencia [rodadas] [basica]` (sai com código 1 se p95 > 5 s).
- Circuit breaker: Spike lenta ou fora do ar vira "análise parcial" sem travar o atendimento ([[RFC-PEP-005-spike-adapter|RFC-PEP-005]]).

## Decisões a tomar

- Paralelizar (1) basta, ou a resposta progressiva (2) é necessária? Recomendação: começar por (1) e medir de novo; se o p95 ficar > 5 s, adotar (2).
