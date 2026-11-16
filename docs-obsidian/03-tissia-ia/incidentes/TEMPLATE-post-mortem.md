---
projeto: tissia-ia
tipo: incidente
status: template
tags:
  - tissia
  - tissia-ia
  - incidente
  - post-mortem
  - operacao
origem: tissia-ai/docs/incidentes/TEMPLATE-post-mortem.md
---

# Post-Mortem de Incidente — AAAA-MM-DD — `<Título Curto>`

> Preencher no mesmo dia em `docs/incidentes/AAAA-MM-DD-<slug>.md` sempre que ocorrer incidente com dado ou alerta clinicamente errado (`docs/release-plan.md §8`).

## 1. Resumo

- **Data/Hora de detecção:** `AAAA-MM-DD HH:MM BRT`
- **Data/Hora de mitigação:** `AAAA-MM-DD HH:MM BRT`
- **Severidade:** `Crítica | Alta | Média`
- **Clínica(s) / Integrador afetado(s):** `<clinica_ref / integrador>`
- **Impacto:** `<descrever impacto clínico, operacional ou de disponibilidade>`

## 2. Contenção Imediata (< 5 min)

- [ ] Clínica retornada para `modo=sombra` (se alerta clínico incorreto):
  ```bash
  uv run python scripts/configurar_clinica.py --integrador <nome> --clinica-ref <ref> --modo sombra
  ```
- [ ] Kill switch global ativado ou revisão anterior do Cloud Run restaurada (se falha sistêmica)
- [ ] Thiago e clínica afetada comunicados no mesmo dia

## 3. Causa Raiz

`<Explicação objetiva do motivo técnico, regra/DUT ou prompt que gerou o comportamento inesperado>`

## 4. Ações Corretivas e Regressão

- [ ] Caso anonimizado adicionado ao golden set (`evals/casos/`) ou suíte `pytest` antes de reativar `modo=visivel`
- [ ] Correção aplicada em regra / DUT / prompt / código (`PR #...`)
- [ ] Parecer do CMO confirmando resolução (`quando aplicável`)
