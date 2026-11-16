---
projeto: tissia-ia
tipo: rfc
status: concluida
tags:
  - tissia
  - tissia-ia
  - rfc
  - especificacao
aliases:
  - RFC-004-validador-dut-e-justificativa
origem: tissia-ai/docs/rfc/RFC-004-validador-dut-e-justificativa.md
---

# RFC-004 — Validador DUT, justificativa SP/SADT e interface do decisor

**Fase:** F1 · **Status:** Concluída · **ADRs:** 0004, 0007, 0008

## Objetivo

1. Para cada procedimento da guia com DUT curada, dizer quais critérios a evolução documenta e qual o veredito de conformidade.
2. Redigir a indicação clínica usando só fatos da evolução e checar que nada foi inventado.

## Interface do decisor (`app/ia/decisor.py`)

```python
@dataclass
class Pergunta:
    id: str
    tipo: Literal["noul", "choice", "score"]
    texto: str
    opcoes: list[str] | None = None  # choice: até 255


@dataclass
class Resposta:
    id: str
    distribuicao: dict[str, float]  # noul: {"sim": p, "nao": 1-p}; choice: {opcao: p}
    confianca: float


async def decidir(
    state: str | dict, perguntas: list[Pergunta], *, backend: str
) -> list[Resposta]: ...
```

- **Backend `jev`:** um POST com `state` + todas as perguntas. O formato exato segue a doc do early access. O exemplo abaixo é ilustrativo:
  ```json
  { "state": { "evolucao": "…", "procedimento": "RM de joelho" },
    "questions": [ { "id": "c1", "type": "noul", "text": "A evolução documenta teste de Lachman positivo?" } ] }
  ```
- **Backend `gemini`:** uma chamada com saída estruturada `{id: "sim"|"nao"|"incerto", evidencia: str}` por pergunta. Mapeamento: sim → 0,9; não → 0,1; incerto → 0,5. Choice → opção escolhida 0,8, demais divididas igualmente.
- **Evidência:** o Jev não devolve trecho. Para critérios `presente`, a evidência é obtida assim: no backend `gemini`, vem na resposta; no backend `jev`, o campo vai `null` na F1. `# ponytail:` sem evidência no Jev. Se a faturista sentir falta (feedback do piloto), fazer uma chamada Gemini só para os critérios presentes.
- O `state` sempre é o texto **já passado pelo scrubber** (ADR-0007).

## Validador DUT (`app/dut.py`)

```
para cada procedimento p da guia com bases.dut_por_tuss[p]:
    para cada critério c da DUT:
        se c.condicao_estruturada existir:
            avaliar deterministicamente (idade, sexo, cids) → presente (1.0) ou ausente (0.0)
        se c.pergunta existir e não falhou na condicao_estruturada:
            perguntas += [noul(c.pergunta)]
perguntas += perguntas pendentes do motor de regras (acessorio_ausente com condicao)
respostas = decidir(state={evolucao, indicacao_clinica}, perguntas)     # 1 chamada para toda a guia
status(c) = presente se p≥0.7, ausente se p≤0.3, senão incerto
grupo ok  = combinação (qualquer/todos) dos critérios, com lógica de três valores: presente=V, ausente=F, incerto=?
dut       = combinação dos grupos → conforme | nao_conforme | incerto
```

Alertas gerados:

- `nao_conforme` → `dut_criterio_ausente`, `bloqueante`. A mensagem lista os critérios que faltam em linguagem de ação: "Confirme se consta no atendimento: …".
- `incerto` → `dut_criterio_incerto`, `aviso`.
- Em ambos: `fonte = ANS_DUT` + referência da DUT.

## Justificativa SP/SADT

Gerada só se `opcoes.gerar_justificativa`, o tipo de guia tem indicação clínica e há procedimento na guia. É feita **uma única chamada ao Gemini para toda a guia** (consolidando os procedimentos solicitados para o Campo 23 único da guia SP/SADT).

1. **Gemini (Vertex SP)** recebe a evolução (já com scrubber), os procedimentos da guia (termos TUSS), os CIDs e os critérios DUT com status `presente` e suas evidências. Prompt (resumo):
   - "Escreva a indicação clínica para auditor médico de operadora, em português técnico, em até N caracteres."
   - "Use **exclusivamente** fatos presentes na evolução. Não mencione critério que não esteja documentado. Não invente exames, tempos, escalas nem resultados."
   - Saída estruturada `{texto, fatos_usados: [trecho literal da evolução]}`.
2. **Checagem automática em dois níveis (fast-path determinístico + decisor na zona cinza):**
   - cada `fatos_usados[i]` precisa aparecer na evolução (comparação normalizada sem acento/caixa via `rapidfuzz`);
   - **Fast-path:** se todos os `fatos_usados` tiverem similaridade ≥ 0,90 na evolução e o texto da justificativa tiver cobertura lexical compatível com a evolução + termos TUSS/CID da guia, a justificativa é aprovada sem chamada extra de rede ao decisor;
   - **Zona cinza (0,75 ≤ similaridade < 0,90):** pergunta ao decisor: `noul` "O texto da justificativa afirma algum fato clínico que não está na evolução?" com `state = {evolucao, justificativa}`;
   - Se a similaridade for < 0,75 ou o decisor reprovar → descarta e tenta uma vez (se houver orçamento de timeout restante). Se falhar de novo → `justificativa_sugerida = null` + alerta `info`.
3. N = limite do campo de indicação clínica no schema TISS vigente **[validar; referência ~500 caracteres]**. Truncamento é proibido: se passar do limite, pedir para reescrever menor (dentro do mesmo retry).

Prompts versionados em `app/prompts/*.md`. O hash do prompt vai em `analises.modelo_geracao` (ex.: `gemini-x-flash@<hash>`).

## Critérios de aceite

- [x] Uma chamada ao decisor por guia na etapa de avaliação de critérios (DUT + acessórios), verificada com stub contando chamadas.
- [x] A lógica de três valores tem testes de tabela para `qualquer`/`todos` com V/F/?.
- [x] Golden set ([[RFC-IA-005-harness-de-avaliacao|RFC-005]]): recall de critério ausente ≥ 85%, precisão de alerta DUT ≥ 80%, **0 justificativas com fato não suportado** (avaliação humana do CMO numa amostra de 50).
- [x] Justificativa nunca excede N caracteres.
- [x] Timeout do Gemini → resposta com `justificativa_sugerida: null` e `degradado: true` dentro do orçamento.
