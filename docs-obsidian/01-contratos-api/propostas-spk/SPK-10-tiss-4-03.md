---
projeto: ecossistema
tipo: spk
status: ativo
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-10-tiss-4-03.md
---

# SPK-10 — Leitura de XML TISS 4.03.00 no `ingerir`

**Status:** Implementada na `tissia-ia` ([[RFC-IA-013-refinamento-vigencia-ans-e-tiss-403|RFC-IA-013]]: `VERSOES_TISS_SUPORTADAS` inclui `4.02.00` e `4.03.00` em `app/tiss_xml.py`, erro explícito quando a versão está ausente e `versao_tiss_suportada_max` em `GET /v1/bases/versoes`); no `PEP`, o envio de lotes/retornos para `POST /v1/tiss/lotes/ingerir` já está integrado em `tissia-ai.client.ts` (pendente apenas checagem automatizada via `GET /v1/bases/versoes` no portão de release se aplicável) · **Necessária para:** retorno e glosas ([[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]]) e lote enviado à Spike, P2

## Problema

A versão vigente do Componente de Comunicação do Padrão TISS é a **04.03.00** (competência set/2026; Conteúdo e Estrutura 202511), e o PEP passa a gerar lotes nela ([[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]]). O parser da Spike (`app/tiss_xml.py`) aceita só `3.05.00`, `4.00.00` e `4.01.00` (`VERSOES_TISS_SUPORTADAS`) e cai em `4.01.00` quando a versão não vem no arquivo. Um lote ou demonstrativo 4.03.00 enviado a `POST /v1/tiss/lotes/ingerir` é recusado, e o histórico de desfechos da Spike (risco por operadora, RFC-008 dela) para de crescer.

A RFC-012 da Spike cita `POST /v1/guias/retorno` como destino do demonstrativo; essa rota não existe. O PEP usa `POST /v1/tiss/lotes/ingerir`.

## Proposta

1. Incluir `4.03.00` (e `4.02.00`, se o custo for o mesmo) em `VERSOES_TISS_SUPORTADAS`, com os XSD oficiais da ANS versionados e a proveniência registrada (URL e SHA-256 do pacote).
2. Sem versão no arquivo: responder erro claro em vez de assumir `4.01.00`.
3. Contrato da resposta inalterado (`versao_tiss` já é devolvida).
4. Lembrete do 4.03.00: SP/SADT e consulta não trazem CID estruturado; o parser não deve esperar CID nessas guias.
5. Corrigir na RFC-012 da Spike a rota citada para o demonstrativo.

## Do lado do PEP

Nada muda no contrato. Enquanto a Spike não ler 4.03.00, a RFC-PEP-007 guarda o XML e marca a ingestão como pendente na fila, sem bloquear a importação nem a conciliação (a Spike nunca bloqueia faturamento).
