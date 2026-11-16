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
  - RFC-003-motor-de-regras
origem: tissia-ai/docs/rfc/RFC-003-motor-de-regras.md
---

# RFC-003 — Motor de regras determinísticas

**Fase:** F1 · **Status:** Concluída · **ADRs:** 0009

## Objetivo

Pegar os erros administrativos sem IA, rápido e de forma explicável: código inválido, campo faltando, CID incompatível, acessório esquecido, autorização prévia.

## Formato

`data/regras/<fonte>.yaml`, uma lista de regras. Schema comum:

```yaml
- id: TISS-SADT-INDICACAO          # único, estável; aparece no alerta
  tipo: campo_obrigatorio
  fonte: ANS_TISS                  # ANS_TISS | ANS_DUT | OPERADORA:<registro_ans> | BOA_PRATICA
  severidade: bloqueante
  vigencia: { inicio: 2026-01-01, fim: null }
  mensagem: "Guia SP/SADT sem indicação clínica."
  params: { tipo_guia: sp_sadt, campo: indicacao_clinica }
```

## Tipos (conjunto fechado)

| Tipo | `params` | Avaliação |
| :--- | :--- | :--- |
| `campo_obrigatorio` | `tipo_guia`, `campo` | Campo vazio/nulo na guia daquele tipo. Exceção: `indicacao_clinica` vazia com justificativa sugerida → o alerta referencia a sugestão |
| `cid_incompativel_sexo` | — (usa `restricao_sexo` da base CID-10) | CID restrito a um sexo, paciente de outro sexo |
| `incompativel_idade_sexo` | `tuss` ou `cid`, `sexo`, `idade_min`, `idade_max` | Procedimento TUSS ou CID incompatível com o sexo ou fora da faixa etária (`idade_min`..`idade_max`) do paciente |
| `tuss_invalido` | — (usa a base TUSS) | Código inexistente ou fora da vigência na data da análise |
| `acessorio_ausente` | `principal: [tuss…]`, `acessorio: tuss`, `condicao: {pergunta: "..."}` (opcional) | Algum `principal` está na guia, `acessorio` não está e (se houver `condicao`) o decisor responde a pergunta com p ≥ 0,7 sobre a evolução |
| `procedimento_excludente` | `codigos: [tuss, tuss…]` | Dois ou mais códigos TUSS mutuamente excludentes/concomitantes proibidos lançados na mesma guia |
| `quantidade_invalida` | `tuss: [...]`, `max: int` | Procedimento lançado na guia com `quantidade > max` permitido |
| `exige_autorizacao_previa` | `tuss: [...]`, `operadora: registro_ans` | Procedimento na lista para aquela operadora → alerta `aviso` lembrando a autorização. `# ponytail:` não sabemos se a autorização já foi obtida. Adicionar o campo `senha_autorizacao` ao request quando o PEP puder enviar |

Exemplo do caso citado pelo Thiago (os códigos são placeholders e devem ser preenchidos a partir da tabela 22, com revisão do consultor de faturamento):

```yaml
- id: ACESS-US-DOPPLER
  tipo: acessorio_ausente
  fonte: BOA_PRATICA
  severidade: aviso
  mensagem: "Ultrassom com Doppler descrito, mas o código do Doppler não foi lançado. Possível receita não cobrada."
  params:
    principal: ["<TUSS_US_…>"]
    acessorio: "<TUSS_DOPPLER_…>"
    condicao: { pergunta: "O exame descrito incluiu estudo Doppler?" }
```

`acessorio_ausente` com `condicao` é a única regra que chama o decisor. Ela entra no mesmo lote de perguntas da DUT (RFC-004), sem custo de chamada extra.

## Implementação

```python
# app/regras.py
AVALIADORES: dict[str, Callable[[Regra, Guia, Bases], Alerta | None]] = {
    "campo_obrigatorio": _campo_obrigatorio,
    ...
}

def avaliar(guia: Guia, bases: Bases, regras: list[Regra]) -> tuple[list[Alerta], list[PerguntaPendente]]:
    """Alertas determinísticos + perguntas para o decisor (acessorio_ausente com condicao)."""
```

- Regras com `fonte: OPERADORA:<x>` só rodam quando `operadora.registro_ans == x`.
- Regras fora da vigência são ignoradas.
- Carga e validação no startup, junto com as bases. Uma regra inválida derruba o startup (o CI pega antes).

## Fontes de regras na F1

| Arquivo | Dono | Conteúdo inicial |
| :--- | :--- | :--- |
| `data/regras/ans_tiss.yaml` | Dev | Campos obrigatórios de Consulta e SP/SADT na versão TISS vigente **[validar contra o componente organizacional]** |
| `data/regras/boas_praticas.yaml` | Consultor de faturamento + CMO | Acessórios das especialidades-piloto |
| `data/regras/operadora_<registro>.yaml` | Thiago/consultor | Exigências específicas, se exportáveis do sistema dele (Q4) |

## Critérios de aceite

- [x] Um teste por tipo, com caso positivo, caso negativo e caso fora da vigência.
- [x] Teste de carga: 500 regras × guia com 10 procedimentos em < 20 ms.
- [x] Todo alerta carrega `id` da regra, `fonte` e `severidade`.
