---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0006-modos-silencioso-assistido.md
---

# ADR-PEP-0006 — Modo silencioso/assistido controlado pelo PEP; Spike sempre `visivel` para o PEP

**Status:** Aceita — 09/11/2026 (proposta em 07/11/2026; já implementada: o modo vive em `unidades.modo_ia` e a escrita exige `admin:configurar`; a auditoria da troca de modo entra com a [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]]. Desvios nas notas da [[RFC-PEP-005-spike-adapter|RFC-PEP-005]])

## Contexto

O design de produto prevê dois modos por clínica (tela 15): **silencioso** (mede a linha de base sem exibir nada) e **assistido** (exibe e coleta feedback). O argumento de venda é o relatório "glosas que teriam sido evitadas".

A Spike já tem uma flag parecida, `modo` = `sombra | visivel`, em `clinicas.flags` (jsonb), alterada só por script (`scripts/configurar_clinica.py`). Em `sombra` ela grava a resposta completa no banco dela, mas devolve `alertas: []`, `sugestoes_codigos: null` e `justificativa_sugerida: null`. A clínica é autoprovisionada no primeiro `analisar`; no código, sem flag o default é `visivel`, enquanto o release plan da Spike diz que o default é `sombra` (inconsistência registrada em [[mapeamento-pep-tissia-ia|Spike no PEP]]).

Se o PEP usar o `sombra` da Spike, ele não recebe o conteúdo da análise e não consegue montar o relatório nem cruzar com desfechos sem depender de um relatório da Spike.

## Decisão

- **O PEP é o dono do modo.** Flag `modo_spike` por Unidade no PEP (`silencioso | assistido`), editável na tela 15 por perfil gestor, com auditoria.
- **A clínica na Spike fica sempre em `visivel`.** O PEP recebe e grava a análise completa nos dois modos; em `silencioso` a UI não mostra alertas, sugestões nem justificativa, e não há feedback.
- O relatório "glosas que teriam sido evitadas" é calculado no PEP: análises gravadas em modo silencioso × desfechos importados (Tabela 38) ([[RFC-PEP-012-painel-do-gestor|RFC-PEP-012]]).
- Cada análise grava o modo em vigor no momento (`analise_spike.modo`), para separar linha de base e período assistido.
- **Até a [[SPK-01-api-de-clinicas-e-flags|SPK-01]]:** provisionar cada Unidade na Spike por script com `modo=visivel` **explícito** (não confiar no default). Depois, o PEP fixa a flag via API no onboarding da Unidade.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Usar `sombra` da Spike no modo silencioso | O PEP não recebe o conteúdo; relatório dependeria da Spike e de cruzamento por `guia_ref` |
| Não chamar a Spike no modo silencioso | Sem linha de base, não há prova de valor |
| Modo controlado pelos dois lados | Duas fontes de verdade divergem; o default da Spike já é ambíguo |

## Consequências

- O PEP armazena análises que o usuário não viu; isso precisa constar no RIPD e no termo da clínica ([[conformidade-regulatoria|conformidade]]).
- O modo `sombra` da Spike continua útil para integradores de terceiros sem esse controle ([[PRD-002-spike-standalone|PRD-002]]).
- Teste: em `silencioso`, nenhuma resposta da API do PEP para a UI contém alertas/sugestões ([[test-plan-pep|test plan]]).

**Para aceitar:** Pedro confirmar que a Spike aceita `visivel` fixo para o PEP nativo; time da Spike corrigir a ambiguidade do default (código × release plan); jurídico confirmar que guardar a análise não exibida está coberto pelo termo de adesão.
