---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0008-eventos-outbox-bullmq.md
---

# ADR-PEP-0008 — Eventos de domínio com outbox transacional + BullMQ

**Status:** Aceita — 07/11/2026

## Contexto

Os módulos conversam por eventos de domínio ([[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]]): `AtendimentoFechado`, `EvolucaoVersionada`, `GuiaRascunhoCriada`, `GuiaRevisada`, `AnaliseSpikeConcluida`, `FeedbackRegistrado`, `GuiaAssinada`, `LoteGerado`, `LoteEnviado`, `RetornoImportado`, `DesfechoRegistrado`. A Spike e o BI dependem deles; chamadas assíncronas à Spike (desfechos, XML, esquecidas) precisam de retry. Publicar evento direto numa fila depois do `COMMIT` perde evento se o processo cair entre os dois passos. O design de produto deixou "BullMQ/SQS" em aberto.

## Decisão

- **Outbox transacional:** o evento é gravado na tabela `outbox` na mesma transação da mudança de estado. Um relay lê a outbox (`FOR UPDATE SKIP LOCKED`) e publica.
- **BullMQ sobre Redis** como fila (o Redis já é necessário para cache e sessões). Uma fila por consumidor (ex.: `spike.desfechos`, `bi.projecoes`, `notificacoes.whatsapp`).
- **Entrega pelo menos uma vez; consumidores idempotentes** por `evento_id` (tabela `evento_processado`) ou chave natural (`lote_ref` para a Spike).
- Retry com backoff exponencial e fila de mortos (DLQ) com alerta; reprocessamento manual auditado.
- Payload do evento: ids e metadados, sem PII nem texto clínico; o consumidor busca o que precisa.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Publicar direto após commit | Perde evento em falha entre commit e publish |
| Google Pub/Sub / AWS SQS | Gerenciado e escalável, mas amarra à nuvem antes da [[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]] e não traz ganho no volume do piloto |
| Kafka | Operação pesada para o volume previsto |
| Eventos só em memória (EventEmitter) | Não sobrevive a restart; sem retry |

## Consequências

- Redis passa a ser dependência crítica: persistência AOF e monitoramento de fila.
- Ordem só é garantida por agregado (chave de job por `guia_id`/`atendimento_id`).
- Se a carga ou a necessidade de fan-out crescer, trocar o transporte para Pub/Sub mantendo a outbox (nova ADR).
