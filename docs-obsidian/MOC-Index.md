---
projeto: ecossistema
tipo: moc
status: ativo
tags:
  - tissia
  - ecossistema
  - moc
  - indice
aliases:
  - Index
  - Home
  - MOC
---

# MOC-Index — Ecossistema Tissia (`PEP` + `tissia-ia`)

Mapa de Conteúdo (_Map of Content_) central do Workspace Unificado Tissia — **Single Source of Truth** documental para humanos e agentes de IA.

> **Regra de ouro:** Consulte [[01-contratos-api/manual-integracao-tissia-ia-v1|Manual da API v1]] e [[01-contratos-api/mapeamento-pep-tissia-ia|Mapeamento PEP ↔ tissia-ia]] antes de alterar qualquer contrato HTTP, schema Zod (`@pep/contracts`) ou schema Pydantic (`tissia-ia/app/schemas.py`).
>
> **Trilha de Go-Live (22/11/2026 — 100% P0 + P1 + P2 entregues na `main`):** [[release-plan-pep|Release Plan & Checklist de Go-Live]] · [[test-plan-pep|Test Plan]] · [[auditoria-clinica-regulatoria-ux-2026-11|Auditoria Clínica & Encerramento]] · [[hub-pep-tissia|Hub do PEP]].

## Visão do Workspace

- **Código-Fonte Ativo:**
  - `projetos/PEP` (`~/dev/pedro/tissia`) — Monorepo TypeScript (`@pep/ui`, `@pep/backoffice`, `@pep/api`, `@pep/contracts`)
  - `projetos/tissia-ia` (`~/dev/pedro/tissia-ai`) — Motor de IA anti-glosa (Python 3.12 + FastAPI + Pydantic v2 + PostgreSQL/pgvector + Vertex AI)
- **Referências Somente Leitura (`Read-Only`):**
  - `referencias/tiss` — Biblioteca Python/MCP/CLI e schemas oficiais TISS/ANS
  - `referencias/tissia-ai-tunning` — Tabelas brutas TUSS/ANS e XSDs TISS v4.03.00
  - `referencias/pep-medscribe-api-legado` — API NestJS legada pré-monorepo
- **Orquestração:** Arquivo mestre `AGENTS.md` na raiz do workspace.

---

## 00 — Arquitetura e Fundamentos

- [[arquitetura-macro-ecossistema|Arquitetura Macro do Ecossistema (Diagramas Mermaid e Fluxos PEP ↔ tissia-ia)]]
- [[hub-pep-tissia|Hub Técnico do PEP Tissia]]
- [[hub-tissia-ia|Hub Técnico do tissia-ia]]
- [[modelo-de-dados-pep|Modelo de Dados do PEP (Drizzle + PostgreSQL RLS)]]
- [[conformidade-regulatoria|Conformidade Regulatória (LGPD, CFM, ICP-Brasil, TISS/ANS, SBIS)]]
- [[integracoes-externas|Catálogo de Integrações Externas do PEP]]
- [[glossario-pep|Glossário do PEP]]
- [[glossario-tissia-ia|Glossário do tissia-ia]]
- [[auditoria-mvp-plano-orquestracao|Auditoria MVP e Plano de Orquestração Multi-Agente]]
- [[infraestrutura-oci-provisionamento-deploy|Infraestrutura OCI (sa-saopaulo-1) — Provisionamento e Deploy Always Free]]
- [[refatoracao-oci-always-free-plano|Plano Técnico de Refatoração — Adaptação para OCI Always Free]]
- [[plano-guia-indicacao-unica-e-autopreenchimento|Plano — Indicação clínica única (Campo 23) e autopreenchimento da guia SP/SADT]]
- [[fluxo-guia-consulta-exame-e-estados|Fluxo da guia: consulta + exame, preço do contrato e estados]]

---

## 01 — Contratos de API e Propostas SPK

### Contratos Canônicos (`PEP` ↔ `tissia-ia`)

- [[manual-integracao-tissia-ia-v1|Manual de Integração Tissia IA v1 (Fonte da Verdade HTTP/JSON)]]
- [[mapeamento-pep-tissia-ia|Mapeamento PEP ↔ Tissia IA (SpikeAdapter e Lacunas)]]
- [[indice-contratos-e-schemas|Índice de Contratos e Schemas no Código (@pep/contracts, OpenAPI, Pydantic)]]
- [[validacao-integracao-tissia-ia|Roteiro de Validação da Integração]]
- [[ensaios-postman-api-v1|Ensaios Postman API v1]]

### Propostas de Evolução de Contrato (`SPK-01` .. `SPK-15`)

- [[SPK-01-api-de-clinicas-e-flags|SPK-01 — API de clínicas e flags]]
- [[SPK-02-recurso-de-glosa|SPK-02 — Recurso de glosa assistido]]
- [[SPK-03-processamento-assincrono|SPK-03 — Processamento assíncrono e webhooks]]
- [[SPK-04-requisitos-do-scribe|SPK-04 — Requisitos do Scribe para o PEP]]
- [[SPK-05-empacotamento-standalone|SPK-05 — Empacotamento standalone]]
- [[SPK-06-api-de-operacao|SPK-06 — API de operação interna]]
- [[SPK-07-latencia-da-analise|SPK-07 — Latência da análise com IA]]
- [[SPK-08-fallback-local-sinalizado|SPK-08 — Fallback local sinalizado]]
- [[SPK-09-catalogo-tuss|SPK-09 — Catálogo TUSS e CID-10]]
- [[SPK-10-tiss-4-03|SPK-10 — Suporte a TISS 04.03.00]]
- [[SPK-11-contratos-auditar-substituicao|SPK-11 — Semântica de substituição em contratos/auditar]]
- [[SPK-12-desfecho-recurso-manual|SPK-12 — Desfecho de recurso manual sem sobrescrever o original]]
- [[SPK-13-ingestao-mapa-guia-ref|SPK-13 — Mapa de numeroGuiaPrestador ↔ guia_ref na ingestão TISS]]
- [[SPK-14-sugestao-de-codigos-e-indicacao-a-partir-da-evolucao|SPK-14 — Sugestão de CID, exames e indicação a partir da evolução, sem guia em rascunho]]
- [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas|SPK-15 — Metadados de modelo/prompt, resumo a partir do SOAP e tabelas TUSS 18/19/20]]

---

## 02 — PEP

### PRDs

- [[PRD-001-pep-tissia|PRD-001 — PEP Tissia]]
- [[PRD-002-spike-standalone|PRD-002 — Spike como produto standalone]]
- [[PRD-003-backoffice|PRD-003 — Backoffice interno]]

### RFCs do PEP (`RFC-PEP-000` .. `RFC-PEP-029`)

- [[RFC-PEP-000-fundacoes|RFC-PEP-000 — Fundações e arquitetura de referência]]
- [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001 — Identidade e acesso]]
- [[RFC-PEP-002-paciente-e-beneficiario|RFC-PEP-002 — Paciente e beneficiário]]
- [[RFC-PEP-003-agenda|RFC-PEP-003 — Agenda]]
- [[RFC-PEP-004-prontuario|RFC-PEP-004 — Prontuário]]
- [[RFC-PEP-005-spike-adapter|RFC-PEP-005 — SpikeAdapter (integração com tissia-ia)]]
- [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006 — Guias e lotes TISS]]
- [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007 — Retorno e glosas]]
- [[RFC-PEP-008-contratos|RFC-PEP-008 — Contratos com operadoras]]
- [[RFC-PEP-009-receita-esquecida|RFC-PEP-009 — Receita esquecida]]
- [[RFC-PEP-010-financeiro-e-repasse|RFC-PEP-010 — Financeiro e repasse]]
- [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011 — Assinatura ICP-Brasil]]
- [[RFC-PEP-012-painel-do-gestor|RFC-PEP-012 — Painel do gestor]]
- [[RFC-PEP-013-voz-ambiental|RFC-PEP-013 — Voz ambiental]]
- [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014 — Plataforma, LGPD e auditoria]]
- [[RFC-PEP-015-frontend|RFC-PEP-015 — Frontend]]
- [[RFC-PEP-016-expansao|RFC-PEP-016 — Expansão (P3)]]
- [[RFC-PEP-017-backoffice-fundacoes-e-acesso|RFC-PEP-017 — Backoffice: fundações e acesso]]
- [[RFC-PEP-018-backoffice-clinicas-e-onboarding|RFC-PEP-018 — Backoffice: clínicas e onboarding]]
- [[RFC-PEP-019-backoffice-planos-e-cobranca|RFC-PEP-019 — Backoffice: planos e cobrança]]
- [[RFC-PEP-020-backoffice-operacao-da-spike|RFC-PEP-020 — Backoffice: operação da Spike]]
- [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021 — Dados de referência da ANS]]
- [[RFC-PEP-022-triagem-e-checkout-recepcao|RFC-PEP-022 — Triagem (pré-consulta), Check-out da Recepção e fronteira Médico × Faturista]]
- [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023 — Separação Solicitação Externa × Execução na Unidade e Janela de Retorno]]
- [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024 — Evolução de UX Clínica: Escuta Ambiental, Revisão, Documentos e Catálogos]]
- [[RFC-PEP-025-estacao-enfermagem-assinatura-exames-resumo-e-baixa-checkout|RFC-PEP-025 — Estação Dedicada de Enfermagem, Assinatura de Pedidos de Exame, Resumo IA e Baixa de Check-out]]
- [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia|RFC-PEP-026 — Blindagem Regulatória: Autoria, Sigilo, Trilha Selada, LGPD e Governança de IA]]
- [[RFC-PEP-027-motor-tiss-autorizacao-outras-despesas-e-cbhpm|RFC-PEP-027 — Motor TISS SP/SADT Completo: Autorização, Outras Despesas, Multi-Versão e CBHPM]]
- [[RFC-PEP-028-jornada-hibrida-grande-clinica-e-sae-enfermagem|RFC-PEP-028 — Jornada Híbrida Real (Particular × Convênio), Grande Clínica e SAE/Enfermagem]]
- [[RFC-PEP-029-ergonomia-clinica-resiliencia-e-aceleradores-produtividade|RFC-PEP-029 — Ergonomia Clínica, Resiliência Anti-Perda de Dados e Aceleradores de Produtividade]]

### Operação, Planos e Marca (`PEP`)

- [[auditoria-clinica-regulatoria-ux-2026-11|Auditoria Clínica, Regulatória e UX (Nov/2026) — Diagnóstico e Roadmap de Evolução]]
- [[release-plan-pep|Release Plan do PEP]]
- [[test-plan-pep|Test Plan do PEP]]
- [[demo|Ambiente de Demonstração (Docker Compose + Caddy)]]
- [[demo-numeros-do-deck|Números do Deck da Demonstração]]
- [[demo-dados-ficticios|Demo de dados fictícios (seed demo:reset)]]
- [[marca|Marca Tissia: nome, logotipo, tipografia e cores]]

---

## 03 — tissia-ia

### PRD

- [[PRD-IA-001-tissia-ia|PRD-IA-001 — Tissia IA (v3, faseado)]]

### RFCs do Motor de IA (`RFC-IA-000` .. `RFC-IA-014`)

- [[RFC-IA-000-fundacoes|RFC-IA-000 — Fundações]]
- [[RFC-IA-001-bases-de-conhecimento|RFC-IA-001 — Bases de conhecimento]]
- [[RFC-IA-002-api-analise-de-guia|RFC-IA-002 — API de análise de guia]]
- [[RFC-IA-003-motor-de-regras|RFC-IA-003 — Motor de regras]]
- [[RFC-IA-004-validador-dut-e-justificativa|RFC-IA-004 — Validador DUT e justificativa]]
- [[RFC-IA-005-harness-de-avaliacao|RFC-IA-005 — Harness de avaliação]]
- [[RFC-IA-006-sugestao-tuss-cid|RFC-IA-006 — Sugestão TUSS/CID]]
- [[RFC-IA-007-piloto-e-desfecho|RFC-IA-007 — Piloto e desfecho]]
- [[RFC-IA-008-preditivo-historico-e-contratos|RFC-IA-008 — Preditivo, histórico e contratos]]
- [[RFC-IA-009-scribe|RFC-IA-009 — Scribe]]
- [[RFC-IA-010-extensao-chrome|RFC-IA-010 — Extensão Chrome]]
- [[RFC-IA-011-catalogo-terminologia-e-rag|RFC-IA-011 — Catálogo de terminologia e RAG]]
- [[RFC-IA-012-treinamento-sft-vertex-e-rag-spsadt|RFC-IA-012 — Treinamento SFT Vertex e RAG SP/SADT]]
- [[RFC-IA-013-refinamento-vigencia-ans-e-tiss-403|RFC-IA-013 — Refinamento de vigência ANS e TISS 04.03.00]]
- [[RFC-IA-014-api-de-operacao-interna|RFC-IA-014 — API de operação interna (/ops/v1, Fase 1)]]

### Operação, Evals e Incidentes (`tissia-ia`)

- [[runbook-operacao-ia|Runbook de Operação do tissia-ia]]
- [[release-plan-ia|Release Plan do tissia-ia]]
- [[test-plan-ia|Test Plan do tissia-ia]]
- [[2026-11-04-decisor|Eval 2026-11-04 — Decisor]]
- [[TEMPLATE-revisao-cmo|Template de Revisão CMO]]
- [[TEMPLATE-post-mortem|Template de Post-Mortem de Incidente]]

---

## 04 — ADRs

### ADRs do PEP (`ADR-PEP-0001` .. `ADR-PEP-0037`)

- [[ADR-PEP-0001-repositorios|ADR-PEP-0001 — Repositórios separados]]
- [[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002 — Backend NestJS monólito modular]]
- [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003 — PostgreSQL RLS multi-tenant]]
- [[ADR-PEP-0004-frontend|ADR-PEP-0004 — Frontend React + Vite + Tailwind + shadcn]]
- [[ADR-PEP-0005-spike-adapter|ADR-PEP-0005 — SpikeAdapter e pseudonimização]]
- [[ADR-PEP-0006-modos-silencioso-assistido|ADR-PEP-0006 — Modos silencioso e assistido]]
- [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007 — Prontuário imutável]]
- [[ADR-PEP-0008-eventos-outbox-bullmq|ADR-PEP-0008 — Eventos com outbox transacional e BullMQ]]
- [[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009 — Hospedagem GCP São Paulo]]
- [[ADR-PEP-0010-identidade-keycloak|ADR-PEP-0010 — Identidade com Keycloak]]
- [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011 — Assinatura ICP-Brasil]]
- [[ADR-PEP-0012-motor-tiss|ADR-PEP-0012 — Motor TISS]]
- [[ADR-PEP-0013-asr-na-spike|ADR-PEP-0013 — ASR/voz ambiental na Spike]]
- [[ADR-PEP-0014-convenio-e-particular-no-mvp|ADR-PEP-0014 — Convênio e particular no MVP]]
- [[ADR-PEP-0015-offline-prontuario|ADR-PEP-0015 — Offline no prontuário]]
- [[ADR-PEP-0016-migracao-de-concorrentes|ADR-PEP-0016 — Migração de concorrentes]]
- [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017 — Drizzle ORM com isolamento RLS]]
- [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018 — Monorepo pnpm + Turborepo]]
- [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019 — Drizzle Kit migrações e seed]]
- [[ADR-PEP-0020-backoffice-app-separado|ADR-PEP-0020 — Backoffice como app separado]]
- [[ADR-PEP-0021-backoffice-backend|ADR-PEP-0021 — Backend do backoffice]]
- [[ADR-PEP-0022-backoffice-sem-dado-clinico|ADR-PEP-0022 — Backoffice sem dado clínico]]
- [[ADR-PEP-0023-spike-no-backoffice|ADR-PEP-0023 — Spike no backoffice]]
- [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024 — Dados de referência da ANS]]
- [[ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai|ADR-PEP-0025 — Decisões de integração para Scribe e /ops/v1]]
- [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026 — Seed de demonstração modular e referências opcionais]]
- [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027 — Indicação clínica única por atendimento (Campo 23)]]
- [[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028 — Resolvedor de dados da guia no servidor e cadastros de origem]]
- [[ADR-PEP-0029-consulta-tiss-e-preco-de-contrato-na-guia|ADR-PEP-0029 — Consulta sempre cobrada (SP/SADT única ou separada por contrato) e preço só pelo contrato]]
- [[ADR-PEP-0030-separacao-pedido-externo-execucao-local-e-retorno|ADR-PEP-0030 — Separação entre solicitação externa e execução na unidade, e janela de retorno]]
- [[ADR-PEP-0031-separacao-papeis-checkout-recepcao-e-triagem|ADR-PEP-0031 — Separação operacional entre Recepção (check-out), Enfermagem (triagem), Médico e Faturista]]
- [[ADR-PEP-0032-autoria-clinica-sigilo-e-trilha-selada-obrigatoria|ADR-PEP-0032 — Autoria clínica estrita, sigilo por papel e trilha selada obrigatória]]
- [[ADR-PEP-0033-governanca-ia-cfm-2454-e-seguranca-prescricao|ADR-PEP-0033 — Governança de IA (CFM 2.454/2026) e segurança determinística na prescrição]]
- [[ADR-PEP-0034-motor-tiss-autorizacao-outras-despesas-e-cbhpm|ADR-PEP-0034 — Motor TISS SP/SADT: autorização prévia, despesas acessórias, multi-versão e CBHPM]]
- [[ADR-PEP-0035-conta-hibrida-por-item-e-processo-enfermagem-sae|ADR-PEP-0035 — Conta híbrida por item e segregação formal do Processo de Enfermagem (SAE/COFEN)]]
- [[ADR-PEP-0036-defaults-fail-closed-autenticacao-e-assinatura|ADR-PEP-0036 — Defaults fail-closed de autenticação e assinatura]]
- [[ADR-PEP-0037-nota-por-etapa-sae-e-insumos-pendentes-de-codificacao|ADR-PEP-0037 — Nota por etapa SAE e insumos pendentes de codificação]]

### ADRs do Motor de IA (`ADR-IA-0001` .. `ADR-IA-0012`)

- [[ADR-IA-0001-anti-glosa-antes-do-scribe|ADR-IA-0001 — Anti-glosa antes do Scribe]]
- [[ADR-IA-0002-python-fastapi|ADR-IA-0002 — Python + FastAPI]]
- [[ADR-IA-0003-gcp-sao-paulo|ADR-IA-0003 — GCP southamerica-east1]]
- [[ADR-IA-0004-estrategia-de-modelos|ADR-IA-0004 — Estratégia de modelos]]
- [[ADR-IA-0005-integracao-rest-sincrona|ADR-IA-0005 — Integração REST síncrona]]
- [[ADR-IA-0006-bases-versionadas-sem-banco-vetorial|ADR-IA-0006 — Bases versionadas sem banco vetorial na F1]]
- [[ADR-IA-0007-minimizacao-de-dados|ADR-IA-0007 — Minimização de dados e retenção (LGPD)]]
- [[ADR-IA-0008-human-in-the-loop|ADR-IA-0008 — Human-in-the-loop]]
- [[ADR-IA-0009-regras-declarativas|ADR-IA-0009 — Regras determinísticas em YAML]]
- [[ADR-IA-0010-vertex-global-e-gemini-3x|ADR-IA-0010 — Vertex AI global e Gemini 3.x]]
- [[ADR-IA-0011-catalogo-unificado-e-rag-pgvector|ADR-IA-0011 — Catálogo unificado e RAG pgvector]]
- [[ADR-IA-0012-ops-v1-keycloak-jwks|ADR-IA-0012 — API de operação interna /ops/v1 com JWT do Keycloak (JWKS)]]
