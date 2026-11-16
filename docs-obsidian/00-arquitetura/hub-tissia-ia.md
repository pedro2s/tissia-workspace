---
projeto: tissia-ia
tipo: arquitetura
status: ativo
tags:
  - tissia
  - tissia-ia
  - arquitetura
  - hub
  - indice
aliases:
  - README
  - tissia-ia-docs-readme
origem: tissia-ai/docs/README.md
---

# Tissia IA — Documentação Técnica

Fonte de verdade para implementação por fases. Cada fase é executada RFC por RFC; nenhuma RFC é iniciada sem as ADRs dela aceitas.

Insumos originais (não editar): `projetos/tissia-ia/.docs/` — PRD v2.0, one-pagers dos sócios e transcrição da reunião com Thiago Polydoro (21/10/2026).

## Índice

| Tipo | Documento |
| :--- | :--- |
| Produto | [[PRD-IA-001-tissia-ia\|PRD-001 — Tissia IA (v3, faseado)]] |
| Contrato | [[manual-integracao-tissia-ia-v1\|Manual de Integração da API v1]] |
| Glossário | [[glossario-tissia-ia\|Glossário de domínio (TISS, TUSS, DUT, glosa…)]] |
| Operação | [[runbook-operacao-ia\|Runbook de Operação]] |
| Testes | [[test-plan-ia\|Test Plan]] |
| Release | [[release-plan-ia\|Release Plan]] |
| Decisões | [[MOC-Index#04 — ADRs\|ADRs]] |
| Design | [[MOC-Index#03 — tissia-ia\|RFCs]] |

## Mapa de fases

| Fase | Objetivo | RFCs | Status |
| :--- | :--- | :--- | :--- |
| **F0 — Fundações** | Repo, infra GCP, auth, minimização de PII, bases TUSS/CID/DUT versionadas, harness de avaliação de modelos | [[RFC-IA-000-fundacoes\|RFC-000]], [[RFC-IA-001-bases-de-conhecimento\|RFC-001]], [[RFC-IA-005-harness-de-avaliacao\|RFC-005]] | Concluída |
| **F1 — Motor Anti-Glosa MVP** | API síncrona por guia: regras determinísticas + validador DUT + justificativa SP/SADT + sugestão TUSS/CID | [[RFC-IA-002-api-analise-de-guia\|RFC-002]], [[RFC-IA-003-motor-de-regras\|RFC-003]], [[RFC-IA-004-validador-dut-e-justificativa\|RFC-004]], [[RFC-IA-006-sugestao-tuss-cid\|RFC-006]] | Concluída |
| **F2 — Piloto fechado** | Clínicas-piloto da base do Thiago, desfecho de glosa, relatório de piloto | [[RFC-IA-007-piloto-e-desfecho\|RFC-007]] | Concluída |
| **F3 — Preditivo** | Risco de glosa a partir de histórico de lotes TISS e contratos com operadoras | [[RFC-IA-008-preditivo-historico-e-contratos\|RFC-008]] | Concluída |
| **F4 — Scribe** | Evolução clínica estruturada a partir da consulta (pós-consulta → streaming) | [[RFC-IA-009-scribe\|RFC-009]] | Em implementação (pós-consulta completa: documentos, análise encadeada, feedback; streaming pendente) |
| **F4+ — Catálogo & RAG** | Catálogo unificado TUSS/CID-10 para o PEP (`SPK-09`) e RAG híbrido (`pgvector`) para modelo na Vertex AI | [[RFC-IA-011-catalogo-terminologia-e-rag\|RFC-011]], [[RFC-IA-012-treinamento-sft-vertex-e-rag-spsadt\|RFC-012]], [[RFC-IA-013-refinamento-vigencia-ans-e-tiss-403\|RFC-013]] | RFC-011 e RFC-012 concluídas; RFC-013 concluída (falta recuradoria CMO das DUTs) |
| **Operação — `/ops/v1`** | API interna para o backoffice: observabilidade e revogação de chave (SPK-06, Fase 1) | [[RFC-IA-014-api-de-operacao-interna\|RFC-014]] | Concluída (Fase 1 entregue; `GET /avaliacoes` em prod pendente) |
| **F5 — Extensão standalone** | Extensão Chrome "Universal Autofill" para PEPs de terceiros | [[RFC-IA-010-extensao-chrome\|RFC-010]] | Rascunho |

RFCs em **Rascunho** só têm escopo, dependências e perguntas abertas. Detalhar antes de iniciar a fase, com os aprendizados da fase anterior.

## ADRs

| # | Decisão | Status |
| :--- | :--- | :--- |
| [[ADR-IA-0001-anti-glosa-antes-do-scribe\|0001]] | Anti-glosa antes do Scribe | Aceita |
| [[ADR-IA-0002-python-fastapi\|0002]] | Python + FastAPI | Aceita |
| [[ADR-IA-0003-gcp-sao-paulo\|0003]] | GCP `southamerica-east1` | Aceita |
| [[ADR-IA-0004-estrategia-de-modelos\|0004]] | Jev para decisões + Gemini para texto; OpenRouter só em eval | Aceita |
| [[ADR-IA-0005-integracao-rest-sincrona\|0005]] | Integração REST síncrona por guia; UI no PEP | Aceita |
| [[ADR-IA-0006-bases-versionadas-sem-banco-vetorial\|0006]] | Bases de conhecimento como arquivos versionados, sem banco vetorial na F1 | Substituída parcialmente por 0011 |
| [[ADR-IA-0007-minimizacao-de-dados\|0007]] | Minimização de dados e retenção (LGPD) | Aceita |
| [[ADR-IA-0008-human-in-the-loop\|0008]] | Human-in-the-loop: o sistema só sugere | Aceita |
| [[ADR-IA-0009-regras-declarativas\|0009]] | Regras determinísticas em YAML com tipos fechados | Aceita |
| [[ADR-IA-0010-vertex-global-e-gemini-3x\|0010]] | Vertex AI no endpoint `global` e modelos Gemini 3.x | Aceita |
| [[ADR-IA-0011-catalogo-unificado-e-rag-pgvector\|0011]] | Catálogo unificado no Postgres (`pg_trgm` + `pgvector`), ingestão em camadas e RAG para modelo treinado | Aceita |
| [[ADR-IA-0012-ops-v1-keycloak-jwks\|0012]] | `/ops/v1` interno com JWT do Keycloak da equipe (JWKS) | Aceita |

## Convenções

- Idioma: português nos docs e no domínio (nomes de campos da API, regras); inglês só para termos técnicos consagrados.
- ADR: imutável depois de aceita; mudança = nova ADR que a substitui.
- RFC: atualizar a seção "Status" ao iniciar/concluir; desvios de implementação viram nota no fim da RFC.
- Toda afirmação regulatória ou código TUSS/CID citado como exemplo está marcado **[validar]** até ser conferido contra a fonte oficial.
