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
  - 0001-anti-glosa-antes-do-scribe
origem: tissia-ai/docs/adr/0001-anti-glosa-antes-do-scribe.md
---

# ADR-0001 — Anti-glosa antes do Scribe

**Status:** Aceita — 29/10/2026

## Contexto

O PRD v2.0 começava pelo Scribe (transcrição em streaming + SOAP). Na reunião de 21/10/2026, o Thiago Polydoro informou que:

- o PEP dele já grava, transcreve, diariza e resume a consulta (pós-consulta);
- quem lança o faturamento é a secretária/faturista;
- a dor mais carente e monetizável é a glosa (e o procedimento acessório esquecido);
- ele valida produtos com clientes-cobaia antes de vender.

## Decisão

A Fase 1 entrega o **motor anti-glosa** como API: regras determinísticas, validador DUT, justificativa SP/SADT e sugestão TUSS/CID. O Scribe vai para a F4 e, quando chegar, consome primeiro a transcrição que o PEP já produz.

## Consequências

- A entrada da F1 é a guia + o texto da evolução, não o áudio. Não há STT, diarização nem captura na F1.
- O valor é mensurável em R$ (taxa de glosa), o que facilita o piloto e a conversa com os sócios do Thiago.
- O payload do PRD v2 §4 continua como referência para a F4, não para a F1.
- Risco: se o PEP não enviar a evolução junto da guia (pergunta Q1 do PRD), o validador DUT e a sugestão de códigos perdem a entrada. Isso precisa ser confirmado antes de fechar a RFC-002.
