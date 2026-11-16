---
projeto: PEP
tipo: adr
status: substituida-parcialmente
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0001-repositorios.md
---

# ADR-PEP-0001 — UI, backend e Spike em repositórios separados; docs no repo da UI

**Status:** Substituída parcialmente por [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]] em 07/11/2026 (UI e backend do PEP unificados no monorepo `pep-medscribe`; Spike `tissia-ai` e hub `docs/` permanecem como decidido aqui)

## Contexto

São três peças com ciclos de vida diferentes:

- a **UI do PEP** (`pep-medscribe`), hoje React + Vite + TS com as telas 01–26 do design, ainda sem backend;
- o **backend do PEP** (NestJS), que ainda não existe;
- a **Spike** (`tissia-ai`), um serviço Python já em produção, com contrato `v1` próprio, que também é vendido para outros PEPs ([[PRD-002-spike-standalone|PRD-002]]).

A Spike não pode depender do PEP: outros integradores consomem a mesma API ([[ADR-IA-0005-integracao-rest-sincrona|Spike ADR-0005]]). A documentação precisa de um único lugar para quem vai implementar o PEP por fases.

## Decisão

| Repositório | Conteúdo | Deploy |
| :--- | :--- | :--- |
| `pep-medscribe` | UI (SPA/PWA) e **este hub de docs** (`docs/`) | Estático (CDN) |
| `pep-medscribe-api` (a criar na [[RFC-PEP-000-fundacoes\|RFC-PEP-000]]) | Backend NestJS, migrações, workers | Contêiner |
| `tissia-ai` | Spike: docs, código e contrato `v1` | Inalterado (Cloud Run SP) |

- O contrato UI ↔ backend é um OpenAPI gerado pelo backend; a UI gera o cliente tipado a partir dele (snapshot versionado em `pep-medscribe-api`).
- O contrato backend ↔ Spike é o OpenAPI `v1` da Spike (`tests/snapshots/openapi_v1.json`), consumido só pelo módulo `spike` ([[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]]).
- Lacunas da Spike ficam como propostas `SPK-NN` em `docs/spike/` e só viram RFC no repo `tissia-ai` quando aceitas.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Monorepo (UI + backend + Spike) | Mistura Python e Node num pipeline só e acopla o release da Spike ao do PEP; a Spike é vendida à parte |
| Monorepo só do PEP (UI + backend) | Viável; rejeitado porque o backend foi pedido como projeto à parte e a UI já existe sozinha. Reavaliar se a sincronia de contrato virar atrito |
| Docs em cada repo | Três lugares para ler antes de cada RFC; decisão do usuário foi hub único |

## Consequências

- Mudança de contrato UI ↔ backend exige PR nos dois repos; o teste de contrato (snapshot do OpenAPI) pega a divergência ([[test-plan-pep|test plan]]).
- A documentação do backend mora fora do repo do backend: o `README` de `pep-medscribe-api` aponta para este hub.
- Links para a Spike (`../../../tissia-ai/...`) só funcionam com os repos lado a lado em `~/dev/pedro/`.
