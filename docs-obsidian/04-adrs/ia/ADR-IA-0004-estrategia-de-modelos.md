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
  - 0004-estrategia-de-modelos
origem: tissia-ai/docs/adr/0004-estrategia-de-modelos.md
---

# ADR-0004 — Estratégia de modelos: Jev para decisões, Gemini para texto, OpenRouter só em eval

**Status:** Aceita — 04/11/2026 (arquitetura `decidir()` validada no eval da F0 em [[2026-11-04-decisor|eval 2026-11-04-decisor]]; `gemini` em `southamerica-east1` ativo como default até assinatura do DPA do Jev)

## Contexto

O motor tem dois tipos de tarefa de IA:

| Tipo | Exemplos | Natureza |
| :--- | :--- | :--- |
| **Decisão** | "A evolução documenta o critério X da DUT?"; "qual destes 40 códigos TUSS candidatos corresponde ao procedimento?"; "a justificativa contém fato ausente na evolução?" | Classificação/score com saída fechada |
| **Geração** | Redigir a indicação clínica da SP/SADT; extrair os termos de procedimento/diagnóstico citados na evolução | Texto livre ou estruturado |

Candidatos avaliados:

- **Jev (TypeSafe AI)**, sugerido pelo Thiago. Modelo "System One" discriminativo: recebe `state` (texto/JSON) + perguntas `noul` (sim/não), `choice` (até 255 opções) e `score` (escala), e devolve distribuições de probabilidade calibradas. Números divulgados pelo fornecedor: 70–500 ms de latência, US$ 0,042/M tokens de entrada, saída gratuita, até 64k tokens de entrada. Por construção, não devolve valor fora do schema. **Limitações:** não gera texto; early access por lista de espera; arquitetura e benchmarks independentes não publicados; nada publicado sobre retenção, região ou conformidade (LGPD/HIPAA). Perguntas são respondidas em paralelo, então uma pergunta que depende de outra exige uma segunda chamada.
- **Gemini via Vertex AI (São Paulo).** Gera texto e saída estruturada, com processamento regional no Brasil para os modelos listados na tabela de *data residency* **[validar modelo]**. Coberto pelos termos de dados do Google Cloud.
- **OpenRouter.** Roteador para dezenas de modelos com controle de *zero data retention*, mas as regiões de processamento são EUA/UE, sem opção Brasil. O DPA diz que não processa dado sensível sem acordo explícito. Útil para comparar modelos; inadequado para dado de saúde real.

## Decisão

1. **Decisões → Jev**, atrás de uma função única `decidir(state, perguntas)` com dois backends: `jev` e `gemini`. O backend `gemini` emula as perguntas com saída estruturada (`sim`/`nao`/`incerto` mapeados para 0,9/0,1/0,5). `# ponytail:` mapeamento fixo; trocar por logprobs se a calibração do Gemini importar. O backend é escolhido por configuração (global e por clínica).
2. **Geração → Gemini Flash no Vertex `southamerica-east1`** (o modelo exato é configuração; usar o mais recente com residência no Brasil).
3. **OpenRouter → só no harness de avaliação ([[RFC-IA-005-harness-de-avaliacao|RFC-005]]) com casos sintéticos.** O runner recusa casos reais quando o provedor é OpenRouter.

### Gates para usar o Jev com dado real

- [ ] Acesso ao early access obtido e API documentada (o formato de request na RFC-004 é ilustrativo).
- [ ] TypeSafe informa região de processamento, retenção e uso para treino; assinatura de DPA com cláusulas-padrão da ANPD (Res. CD/ANPD nº 19/2024) se o processamento for fora do Brasil.
- [x] Dados enviados ao Jev passam pelo scrubber de PII ([[ADR-IA-0007-minimizacao-de-dados|ADR-0007]]).
- [x] No golden set, o Jev empata ou supera o Gemini em recall/precisão de DUT e em top-3 de códigos, com p95 menor ([[2026-11-04-decisor|eval 2026-11-04-decisor]]).

Enquanto algum gate não passar, o backend de decisão em produção é o `gemini`. O código é o mesmo; muda só a configuração.

## Consequências

- Duas dependências de IA em vez de uma, mas a interface de decisão é a mesma, então trocar de backend é configuração.
- O custo por guia tende a ser dominado pela geração (Gemini), não pela decisão.
- Revisar esta ADR na F3: o preditivo sobre histórico pode ser modelo tabular próprio (scikit-learn) ou Jev com o estado da guia. Decidir com dados.
