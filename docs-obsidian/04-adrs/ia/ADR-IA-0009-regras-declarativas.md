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
  - 0009-regras-declarativas
origem: tissia-ai/docs/adr/0009-regras-declarativas.md
---

# ADR-0009 — Regras determinísticas em YAML com tipos fechados

**Status:** Aceita — 29/10/2026

## Contexto

As regras de faturamento (acessórios, campos obrigatórios, compatibilidades, exigências de operadora) serão escritas por quem conhece faturamento (o consultor indicado pelo Thiago e o CMO), não só por devs. Motores de regra genéricos (Drools, DSLs, json-logic) são flexíveis demais para o volume da F1.

## Decisão

- Regras em `data/regras/*.yaml`, com um conjunto **fechado** de tipos, cada um implementado por uma função Python pequena ([[RFC-IA-003-motor-de-regras|RFC-003]]):
  - `campo_obrigatorio`
  - `cid_incompativel_sexo`
  - `incompativel_idade_sexo`
  - `tuss_invalido`
  - `acessorio_ausente`
  - `procedimento_excludente`
  - `quantidade_invalida`
  - `exige_autorizacao_previa`
- Cada regra tem `id`, `fonte` (`ANS_TISS`, `ANS_DUT`, `OPERADORA:<registro_ans>`, `BOA_PRATICA`), `severidade`, `mensagem` e `vigencia`.
- Um tipo novo exige PR com código + teste. Uma regra nova de um tipo existente é só YAML.

## Consequências

- O YAML é revisável em PR e validado por schema Pydantic no startup (regra inválida impede o deploy).
- `# ponytail:` regras no repo. Quando clínicas/operadoras tiverem regras próprias editáveis (F2+), elas vão para o Postgres com o mesmo schema.
