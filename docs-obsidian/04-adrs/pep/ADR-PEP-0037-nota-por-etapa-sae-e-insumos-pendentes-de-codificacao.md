---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - enfermagem
  - sae
  - cofen
  - faturamento
aliases:
  - ADR-PEP-0037
  - Nota por etapa SAE e insumos pendentes de codificação
---

# ADR-PEP-0037 — Nota pós-checagem por etapa SAE e insumos pendentes de codificação/precificação

**Status:** Aceita — 26/11/2026
**Relacionadas:** [[ADR-PEP-0035-conta-hibrida-por-item-e-processo-enfermagem-sae|ADR-PEP-0035]], [[ADR-PEP-0033-governanca-ia-cfm-2454-e-seguranca-prescricao|ADR-PEP-0033]], [[RFC-PEP-028-jornada-hibrida-grande-clinica-e-sae-enfermagem|RFC-PEP-028]], [[RFC-PEP-025-estacao-enfermagem-assinatura-exames-resumo-e-baixa-checkout|RFC-PEP-025]]

## Contexto

A jornada de enfermagem de 26/11/2026 achou dois problemas que a [[ADR-PEP-0035-conta-hibrida-por-item-e-processo-enfermagem-sae|ADR-PEP-0035]] (aceita, imutável) não cobria: (1) toda nota sugerida após a checagem era gerada como `evolucao`, ato **privativo do enfermeiro** (Lei 7.498/1986, COFEN 736/2024), embora a checagem seja executada em geral pelo **técnico**; o técnico ficava sem caminho de registro e o enfermeiro supervisor era bloqueado por `executadoPor !== usuarioId`; (2) a baixa de insumo sem TUSS ou sem preço gravava o texto literal `<TUSS_...>` e valor 0 em despesa e conta, o que quebraria o XML TISS e mascararia receita esquecida.

## Decisão

1. A nota sugerida da checagem tem **etapa**: o técnico gera e assina a **Anotação de Enfermagem** (`implementacao`) da ordem que executou; a `evolucao` é estritamente do enfermeiro, que pode evoluir a partir de ordem executada por técnico. Existe uma nota por `(ordem, etapa)`.
2. A baixa de insumo **nunca grava `<TUSS_...>` nem assume valor 0**. Sem TUSS ou sem preço (buscado em `tabela_particular` quando há TUSS), fica **pendente** (`codificacao`, `precificacao` ou `codificacao_precificacao`), **fora** de despesa, conta, guia e recebível, sem travar a checagem; a enfermagem confirma só item, quantidade e lote.
3. O **faturista** (`guias:escrever`) resolve a pendência informando TUSS e valor; a resolução é transacional, idempotente e auditada, e a enfermagem não vê preço.

## Consequências

- A migração `0082` não é puramente aditiva: remove `ux_nota_sugerida_ordem` (trocado pelo composto por etapa) e relaxa `despesa_id` para anulável. Só afrouxa regras, e o código anterior não depende do índice removido.
- O TUSS de insumo é validado só pelo **formato** enquanto as tabelas 19 e 20 não forem carregadas; é uma exposição a glosa por código inexistente.
- Ficam pendentes: limpar linhas históricas com `<TUSS_...>` ou valor 0, e a regra do enfermeiro que não assina a anotação de um técnico (`403` de propósito).
