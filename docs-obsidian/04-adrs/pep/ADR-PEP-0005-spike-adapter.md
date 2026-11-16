---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0005-spike-adapter.md
---

# ADR-PEP-0005 — Camada anticorrupção (SpikeAdapter) e pseudonimização no PEP

**Status:** Aceita — 07/11/2026

## Contexto

A Spike tem contrato próprio, estrito (`extra="forbid"` → 422 em qualquer campo a mais), sem PII por construção ([[ADR-IA-0007-minimizacao-de-dados|Spike ADR-0007]]) e é vendida a outros PEPs, então não vai se moldar ao modelo do PEP. O PEP tem o paciente identificado, a guia com dados completos e o fluxo clínico. Se cada módulo chamar a Spike direto, o vínculo paciente ↔ ref, os limites do contrato e a degradação se espalham pelo código.

## Decisão

Um módulo NestJS `spike` é o **único** ponto de contato com a Spike:

- **Pseudonimização:** `paciente.ref = HMAC-SHA256(paciente_id, sal_da_clinica)`; tabela de vínculo `vinculos_spike` criptografada no PEP. `clinica_ref` = id estável da **Unidade**; `guia_ref` = id estável da guia, igual em todas as revisões. Só sexo e idade (anos) saem; nunca nome, CPF, CNS, carteirinha, contato.
- **DTOs tipados** gerados do OpenAPI `v1` da Spike; o adapter monta o payload só com campos do contrato. Teste de contrato contra o snapshot.
- **Limites do contrato aplicados antes do envio:** corpo ≤ 64 KB; `evolucao_clinica` ≤ 20.000 caracteres, truncada priorizando o atendimento atual; `indicacao_clinica` ≤ 5.000; Campo 23 ≤ 500 no editor.
- **Timeout de 6 s**; `degradado: true` ou timeout → UI mostra "análise parcial" e o fluxo segue com validações locais (nunca bloqueia o atendimento).
- **Persistência:** `analise_id`, `versao_bases`, `score_risco`, `risco`, `degradado`, resposta completa e decisões do usuário, vinculadas à revisão da guia.
- **Assíncrono** (fila, retry com backoff, idempotência por `lote_ref`): desfechos, ingestão XML, auditoria de contratos, guias esquecidas.
- Credencial: uma API key de integrador por ambiente, no cofre de segredos.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Chamadas diretas de cada módulo | Duplica pseudonimização e tratamento de erro; risco de PII vazar num módulo esquecido |
| Spike aceitar o modelo do PEP | Quebra o contrato usado por outros integradores e a minimização |
| Proxy genérico (gateway) | Não resolve mapeamento de domínio nem pseudonimização |

## Consequências

- Teste de propriedade: nenhum payload que sai do adapter contém PII do paciente ([[test-plan-pep|test plan]]).
- Troca de versão da Spike (`/v2`) afeta só o módulo `spike`.
- Detalhes na [[RFC-PEP-005-spike-adapter|RFC-PEP-005]] e em [[mapeamento-pep-tissia-ia|Spike no PEP]].
