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
  - 0005-integracao-rest-sincrona
origem: tissia-ai/docs/adr/0005-integracao-rest-sincrona.md
---

# ADR-0005 — Integração REST síncrona por guia; UI fica no PEP

**Status:** Aceita — 29/10/2026

## Contexto

A parceria com o Thiago ainda depende dos sócios dele, e não temos acesso ao código do PEP. A faturista monta a guia dentro do PEP, e é ali que o alerta precisa aparecer. Na reunião também se falou de webhook e de socket. Esses dois só fazem sentido com streaming (F4).

## Decisão

- **`POST /v1/guias/analisar`, síncrono:** o PEP chama ao salvar ou antes de enviar a guia e exibe a resposta. Detalhes na [[RFC-IA-002-api-analise-de-guia|RFC-002]].
- Autenticação: API key por integrador (header `Authorization: Bearer <chave>`), com `clinica_id` no corpo. Chave guardada como hash no banco.
- Sem UI própria na F1. Para o piloto e testes manuais, o Swagger UI do FastAPI (`/docs`, protegido) e consultas SQL sobre as análises bastam.
- Versionamento no path (`/v1`). Mudança incompatível = `/v2` convivendo por no mínimo 90 dias.

## Consequências

- O PEP precisa lidar com timeout e indisponibilidade: se não houver resposta em 6 s, segue sem alertas. Isso vai no guia de integração.
- Webhook/socket só na F4 (Scribe streaming) e, se necessário, para análises de lote na F3 (aí via callback).
- Plano B sem o Thiago: qualquer PEP ou planilha pode chamar a mesma API. O contrato não tem nada específico dele.
