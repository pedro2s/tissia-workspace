---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - demo
  - metricas
origem: tissia/docs/demo-numeros-do-deck.md
---

# Números do deck da demo — reprodução da avaliação da Spike

> **Advertência:** todos os números abaixo vêm de um **golden set sintético de 60 casos** (`evals/casos/sinteticos.jsonl`), nunca de produção nem de clínica. Não citar como resultado de clínica.


## Atualização (09/11/2026, Spike refinada — commit `e469883` do `tissia-ai`)

Depois da primeira rodada (abaixo), a Spike recebeu ajustes de extração e decisão no Vertex. O comando é o mesmo (`uv run python -m evals.run --conjunto sinteticos --decisor gemini`, Gemini real, 60 casos sintéticos). Duas rodadas independentes:

| Número | 1ª rodada (antes do ajuste) | Rodada A (arquivo do repo da Spike) | Rodada B (reproduzida aqui) | Estável entre A e B? |
| :--- | :---: | :---: | :---: | :--- |
| Recall de critério DUT ausente | 100,0% | 100,0% | 100,0% | **Sim** (100%) |
| Recall@50 de candidatos | 100,0% | 100,0% | 100,0% | **Sim** (100%) |
| Precisão de alerta (meta ≥ 80%) | 63,8% | 88,4% | 88,9% | **Sim** (≈ 88–89%, meta atendida) |
| Top-3 TUSS | 90,3% | 100,0% | 95,2% | **Não** (95–100%) |
| Top-3 CID-10 | 93,3% | 93,3% | 93,3% | Sim (93,3%) |
| Fatos não suportados / negação gerando sugestão | 0 / 0 | 0 / 0 | 0 / 0 | Sim |
| Taxa de `incerto` | 10,3% | 9,5% | 12,7% | Varia (meta ≤ 15%: ok) |
| Brier | 0,0191 | 0,0130 | 0,0191 | Varia |
| Latência p50 / p95 do harness | 8,3 / 16,0 s | 6,3 / 24,2 s | 5,8 / 15,5 s | Não usar (ver abaixo) |

**O que citar:** recall de critério DUT ausente 100%, recall@50 100%, precisão de alerta ≈ 88–89% (acima da meta de 80%), Top-3 CID-10 93,3% e Top-3 TUSS entre 95% e 100%. **Não citar** "100% de TUSS": o modelo não é determinístico e ele não se repetiu; citar a faixa. Sempre "golden set sintético de 60 casos", nunca resultado de clínica.

**O que não citar:** (1) a latência do harness (varia de 15 a 24 s no p95 e não é a latência do PEP; a do PEP é medida com `pnpm --filter @pep/api medir:latencia`: 1ª fase p95 ≈ 2,6–3,1 s e análise completa p95 ≈ 6,7–7,6 s); (2) o custo por guia, que continua sendo uma constante no código e não foi medido; (3) o status "OK" de latência na saída do harness, que é fixo.

**Ressalva de método:** são duas rodadas, não uma distribuição. Antes de pôr uma casa decimal no deck, rodar mais 3 vezes e usar o intervalo.

---

## Primeira rodada (09/11/2026, antes do ajuste da Spike)

- Rodada: 09/11/2026, repo `tissia-ai`, `IA_PROVEDOR=vertex` (padrão), `IA_TIMEOUT_S=10.0`, bases `0342dab23f9b`.
- Comando: `uv run python -m evals.run --conjunto sinteticos --decisor gemini` (gerador padrão do CLI `gemini-3.5-flash`; é só rótulo, o modelo real vem da config da Spike). Resultado gravado fora do repo da Spike.
- Comando de controle (sem IA): `IA_PROVEDOR=local uv run python -m evals.run --decisor gemini --gerador gemini-2.5-flash` (e `--decisor jev`).
- Relatório de referência: `tissia-ai/docs/evals/2026-11-04-decisor.md`.

## Resultado

| Número | Reproduzido com Gemini real (09/11) | Relatório de 04/11 | Controle `IA_PROVEDOR=local` (09/11) | Confirma? |
| :--- | :---: | :---: | :---: | :--- |
| Recall de critério DUT ausente | 100,0% | 98,1% | 98,1% | Diverge (+1,9 p.p.); meta >= 85% atendida |
| Precisão de alerta | **63,8%** | 88,6% | 88,6% | **Diverge (-24,8 p.p.); abaixo da meta de 80%** |
| Taxa de `incerto` | 10,3% | 12,7% | 12,7% | Diverge (-2,4 p.p.) |
| Brier | 0,0191 | 0,0667 | 0,0667 | Diverge (melhor) |
| Top-1 / Top-3 TUSS | 85,5% / 90,3% | 100% / 100% | 100% / 100% | **Diverge (-14,5 / -9,7 p.p.)** |
| Top-1 / Top-3 CID-10 | 83,3% / 93,3% | 96,7% / 96,7% | 96,7% / 96,7% | Diverge (-13,4 / -3,4 p.p.) |
| Recall@50 de candidatos | 100,0% | 99,2% | 99,2% | Diverge (+0,8 p.p.) |
| Fatos não suportados / negação gerando sugestão | 0 / 0 | 0 / 0 | 0 / 0 | Confirma |
| Latência p50 / p95 por guia | 8,3 s / 16,0 s | 1,8 ms / 2,9 ms | 3,8 ms / 4,7 ms | **Diverge (meta <= 5 s não atendida)** |
| Custo de IA por guia | R$ 0,024 (constante no código, não medido) | R$ 0,024 | R$ 0,024 | Não verificável |

Observação: houve registro `ia_indisponivel` durante a rodada; com `IA_TIMEOUT_S=10.0` e p95 de 16 s, parte das chamadas pode ter caído na heurística local, o que mistura os dois modos. Não foi contado quantas.

## Por que o relatório de 04/11 não vale para o Gemini

1. O relatório de 04/11 é anterior à integração real do Gemini no decisor (commit `9a214cc`, 08/11). Em 04/11, `decidir_gemini`, `decidir_jev` e `decidir_openrouter` caíam na **heurística local** (`avaliar_perguntas_local`). Com `IA_PROVEDOR=local` hoje, os números de 04/11 se reproduzem exatamente (e as latências são de milissegundos, incompatíveis com chamada de IA).
2. O rótulo "gerador `gemini-2.5-flash`" no relatório é só argumento de CLI; não foi usado para nada.
3. `custo_estimado_brl` é constante fixa em `evals/run.py` (0,019 para `jev`, 0,024 para os demais). Não é custo medido. O status "OK" de latência na tabela do terminal também é fixo. O custo real por guia ainda não foi medido e o número de chamadas ao Gemini por caso não é contado pelo harness.
4. Existe ainda `evals/resultados/2026-11-08-...-gemini-3.5-flash.json` (rodada real anterior): precisão 61,7%, Top-1 TUSS 87,1%; consistente com esta rodada, não com 04/11.

## Por que os três decisores têm métricas idênticas em 04/11

Não é bug de métrica: o cálculo é correto e distingue os decisores. É que os três backends rodaram **a mesma heurística local** (o `openrouter:*` apenas faz POST se houver `OPENROUTER_API_KEY` e ignora a resposta; `jev` sem `JEV_API_URL`/`JEV_API_KEY` usa o emulador local calibrado). Única diferença: o Brier do `jev` (0,0616 vs 0,0667) vem do flag `calibrado_jev`. O relatório portanto não compara decisores reais. Não foi corrigido (fora do escopo).

## Recomendação para o deck

Citar os números do Gemini real desta rodada (ou rodar de novo), não os de 04/11: precisão de alerta 63,8% e latência p50 de 8,3 s ficam fora das metas do PRD. Se a meta de 80% de precisão for mantida, a Spike precisa de ajuste antes de a precisão entrar no deck como atingida. Custo por guia só pode ser citado depois de medido.
