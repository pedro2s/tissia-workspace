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
  - 0008-human-in-the-loop
origem: tissia-ai/docs/adr/0008-human-in-the-loop.md
---

# ADR-0008 — Human-in-the-loop: o sistema só sugere

**Status:** Aceita — 29/10/2026

## Contexto

Justificativa clínica redigida por IA que afirme um achado inexistente pode configurar informação falsa à operadora. As normas do CFM sobre prontuário (Res. CFM nº 1.638/2002) e telemedicina (Res. CFM nº 2.314/2022) atribuem ao médico a responsabilidade pelo conteúdo registrado. **[validar]** se há resolução CFM específica sobre IA vigente em 2026.

## Decisão

- A API devolve **sugestões**. Ela não envia guia à operadora, não grava no PEP e não assina nada.
- Toda justificativa vem com `gerada_por_ia: true` e com um `aviso_legal` que o integrador deve exibir.
- A justificativa só usa fatos presentes na evolução. Quando falta critério DUT, a resposta é um **alerta pedindo confirmação**, nunca um texto que afirme o critério.
- O feedback (`aceito`/`editado`/`rejeitado`) registra o papel de quem decidiu (`faturista` ou `medico`).

## Consequências

- O guia de integração exige que o PEP mostre a justificativa como editável e só a use depois de uma ação humana.
- Métrica de segurança no eval: taxa de "fato não suportado" nas justificativas = 0 no golden set ([[test-plan-ia|test plan]]).
