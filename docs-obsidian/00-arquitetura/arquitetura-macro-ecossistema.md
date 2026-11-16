---
projeto: ecossistema
tipo: arquitetura
status: ativo
tags:
  - tissia
  - ecossistema
  - arquitetura
  - mermaid
  - sistemas-distribuidos
aliases:
  - visao-geral-ecossistema
  - arquitetura-macro
---

# Arquitetura Macro do Ecossistema Tissia

Visão consolidada da arquitetura distribuída entre o **PEP Tissia** (`projetos/PEP`, monorepo TypeScript) e o motor de IA anti-glosa **Tissia IA** (`projetos/tissia-ia`, backend FastAPI/Python).

## 1. Topologia de Serviços e Limites de Domínio

```mermaid
flowchart LR
  subgraph PEP["PEP Tissia (projetos/PEP - Monorepo pnpm/Turborepo)"]
    UI["@pep/ui<br/>React 18 + Vite 5 + TanStack Query"]
    BKO["@pep/backoffice<br/>React + Vite (Operação Interna)"]
    CTR["@pep/contracts<br/>Zod .strict() Isomórfico"]
    API["@pep/api<br/>NestJS 12 Monólito Modular"]
    WRK["@pep/api Worker<br/>BullMQ + Outbox Relay"]
    PG_PEP[("PostgreSQL 16 + RLS<br/>(App RW sem BYPASSRLS)")]
    RED[("Redis 7<br/>Filas BullMQ")]
    KCL["Keycloak 26<br/>OIDC / MFA / FIDO2"]
  end

  subgraph IA["Motor de IA (projetos/tissia-ia - FastAPI / Python 3.12)"]
    FAPI["API Tissia IA v1<br/>Pydantic v2 extra=forbid"]
    PII["Scrubber PII<br/>app/pii.py (Em Memória)"]
    REG["Motor Regras + XML TISS<br/>YAML + Bases Oficiais ANS"]
    LLM["Decisor + LLM/RAG<br/>Vertex AI (Gemini 3.x global)"]
    PG_IA[("PostgreSQL 16<br/>pgvector + pg_trgm")]
  end

  subgraph REF["Referências Read-Only (referencias/)"]
    TISS_LIB["tiss (Python/MCP/XSD)"]
    TUN["tissia-ai-tunning (Tabelas ANS)"]
  end

  UI <-->|"REST /api/v1 + Zod"| API
  BKO <-->|"REST + FIDO2"| API
  CTR -.-> UI
  CTR -.-> API
  API --> KCL
  API <-->|"Db.comTenant + SET LOCAL"| PG_PEP
  API -->|"Transação + Outbox"| PG_PEP
  WRK <--> RED
  WRK <--> PG_PEP

  API -->|"SpikeAdapter (HTTPS/REST, timeout 6s)"| FAPI
  WRK -->|"Feedback / Desfechos / Lotes"| FAPI
  FAPI --> PII --> LLM
  FAPI --> REG --> PG_IA
  LLM --> PG_IA
  TISS_LIB -.->|"Consulta de Padrões"| REG
  TUN -.->|"Ingestão Controlada"| PG_IA
```

## 2. Fluxos Críticos Entre `PEP` e `tissia-ia`

### 2.1 Fluxo Síncrono de Análise Anti-Glosa (`POST /v1/guias/analisar`)

```mermaid
sequenceDiagram
  autonumber
  actor Med as Médico / Faturista (@pep/ui)
  participant Api as @pep/api (SpikeAdapter)
  participant Db as Postgres PEP (RLS)
  participant AI as tissia-ia (FastAPI)
  participant Vtx as Vertex AI (Gemini 3.x)

  Med->>Api: Fecha atendimento ou valida guia SP/SADT
  Api->>Db: Lê evolução, guia e vínculo pseudonimizado (vinculos_tissia_ai)
  Note over Api: Minimização LGPD: envia apenas clinica_ref (un_), paciente.ref (pa_ HMAC), sexo, idade
  Api->>AI: POST /v1/guias/analisar (Bearer tsk_*, timeout 6s)
  AI->>AI: Valida Pydantic (extra="forbid") + Scrubber PII em memória
  AI->>AI: Avalia regras determinísticas ANS/Operadora + score_risco
  AI->>Vtx: Avalia critérios DUT, codificação TUSS/CID e Campo 23 (budget 4s)
  alt Sucesso na IA
    Vtx-->>AI: Resposta estruturada
    AI-->>Api: 200 OK (degradado: false, alertas, dut, justificativa, sugestoes)
  else Timeout/Instabilidade no Provedor IA
    AI-->>Api: 200 OK (degradado: true, regras determinísticas + score_risco)
  end
  Api->>Db: Persiste analises_tissia_ai + evento AnaliseTissiaAiConcluida (Outbox)
  Api-->>Med: Exibe alertas e sugestões (Human-in-the-Loop obrigatório)
```

### 2.2 Fluxo Assíncrono Pós-Decisão e Faturamento (`Outbox` + `BullMQ`)

```mermaid
sequenceDiagram
  autonumber
  actor User as Médico / Faturista
  participant Api as @pep/api
  participant Out as Outbox / BullMQ Worker
  participant AI as tissia-ia (FastAPI)

  User->>Api: Aceita / Edita / Rejeita sugestão (decisoes_usuario)
  Api->>Out: Grava decisão + evento FeedbackRegistrado na mesma transação
  Out->>AI: POST /v1/analises/{id}/feedback (204 No Content, retry com backoff)

  Note over User,AI: Retorno da Operadora (Lote TISS / Demonstrativo)
  User->>Api: Importa retorno XML da operadora
  Api->>Out: Grava desfechos + remove identificação direta antes de enviar
  Out->>AI: POST /v1/desfechos e/ou POST /v1/tiss/lotes/ingerir
```

## 3. Pontos de Entrada da Documentação

- **Índice Mestre:** [[MOC-Index]]
- **Hub do PEP (`projetos/PEP`):** [[hub-pep-tissia]] · [[modelo-de-dados-pep]] · [[integracoes-externas]]
- **Hub do Motor de IA (`projetos/tissia-ia`):** [[hub-tissia-ia]] · [[runbook-operacao-ia]]
- **Contratos entre Serviços:** [[manual-integracao-tissia-ia-v1]] · [[mapeamento-pep-tissia-ia]] · [[indice-contratos-e-schemas]]
- **Governança e Segurança:** [[conformidade-regulatoria]] · [[ADR-PEP-0005-spike-adapter]] · [[ADR-IA-0007-minimizacao-de-dados]]

