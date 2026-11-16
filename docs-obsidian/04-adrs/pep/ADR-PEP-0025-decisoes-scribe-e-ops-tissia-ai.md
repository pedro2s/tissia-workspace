---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai.md
---

# ADR-PEP-0025 — Decisões de integração para Scribe (streaming) e `/ops/v1` na `tissia-ai`

**Status:** Aceita — 15/11/2026 (alinhada com [[ADR-IA-0012-ops-v1-keycloak-jwks|ADR-IA-0012]])

## Contexto

Duas decisões de fronteira entre o PEP Tissia (incluindo o backoffice) e a `tissia-ai` estavam em aberto e bloqueavam o alinhamento de escopo do piloto e da operação interna:

1. **Streaming do Scribe ([[SPK-04-requisitos-do-scribe|SPK-04]], `tissia-ai` [[RFC-IA-009-scribe|RFC-009]] etapa 2):** iniciar o piloto já com WebSocket contínuo ou manter o envio em trechos de ~12 s via HTTP ([[ADR-PEP-0013-asr-na-spike|ADR-PEP-0013]]), dado que ainda não há conjunto de áudio real em pt-BR para benchmark de STT (Deepgram × Whisper GCP SP × Google Chirp), o parecer jurídico sobre voz como dado biométrico está pendente e WebSocket longo impacta a escolha Cloud Run × GKE.
2. **API de operação interna `/ops/v1` ([[SPK-06-api-de-operacao|SPK-06]]):** modelo de autenticação (Keycloak da equipe vs. IdP próprio), existência de `PUT /ops/v1/roteamento` e promoção de prompt a canário antes da existência de um portão formal de aprovação.

## Decisão (proposta)

### 1. Scribe: trechos de ~12 s no piloto; streaming na Fase 2
- O PEP Tissia **mantém o modelo em trechos de ~12 s por HTTP** (`POST /v1/consultas/transcrever` + `POST /v1/consultas/estruturar`) durante o piloto/MVP.
- **Zero retenção de áudio** permanece obrigatório em qualquer etapa (processamento em memória e descarte imediato, exigindo `consentimento_ref`).
- A etapa 2 (WebSocket `/v1/consultas/stream`) só é destravada após:
  1. coleta de métricas de revisão e vocabulário médico real no piloto;
  2. benchmark comparativo dos candidatos de STT com áudio real pt-BR e garantia de residência de dados no Brasil;
  3. parecer jurídico sobre enquadramento biométrico de voz;
  4. avaliação de infraestrutura (Cloud Run vs. GKE) para conexões WebSocket longas.

### 2. `/ops/v1`: adoção em duas fases com Keycloak JWKS
- **Autenticação:** `/ops/v1` adota o **Keycloak da equipe** (`tissia-equipe-<amb>`) validado via JWKS, com audiência `tissia-ai-operacao` e claim `papeis`. API keys de integrador recebem `401`.
- **Fase 1 (Observabilidade + Revogação de emergência):** inclui tabela `ops_auditoria`, rotas `GET` somente leitura (`/saude`, `/analises`, `/analises/{id}` sem texto livre, `/integradores`, `/integradores/{id}/uso`, `/versoes`, `/roteamento`, `/prompts`, `/avaliacoes`, `/bases`, `/regras`, `/feedback/agregado`) e `POST /ops/v1/integradores/{id}/chaves/{chave}/revogacao`.
- **Roteamento e Canário na Fase 2:** `PUT /ops/v1/roteamento` **não entra** na Fase 1 — a troca de provedor/fallback continua por configuração de deploy revisada no Git. `POST /ops/v1/prompts/{familia}/canario` também fica para a Fase 2, condicionado à implementação do portão formal de aprovação de prompts e dupla aprovação.

## Consequências

- O piloto do PEP e o backoffice B2 (leitura operacional da `tissia-ai`) avançam sem depender de migração para GKE, benchmark prévio de STT ou novo IdP.
- O repositório `tissia-ai` recebe um escopo enxuto e seguro para o `/ops/v1` Fase 1, mantendo zero exposição de texto clínico, justificativa gerada ou `paciente_ref`.

