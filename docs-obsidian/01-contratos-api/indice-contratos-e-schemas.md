---
projeto: ecossistema
tipo: contrato-api
status: ativo
tags:
  - tissia
  - ecossistema
  - contrato-api
  - openapi
  - zod
  - pydantic
aliases:
  - indice-contratos-schemas
---

# Índice de Contratos, Schemas e Código-Fonte

Este documento conecta a documentação de contratos em `01-contratos-api/` aos schemas executáveis e snapshots OpenAPI mantidos nos dois repositórios ativos (`projetos/PEP` e `projetos/tissia-ia`).

## 1. Documentos Canônicos de Contrato no Vault

| Documento | Papel | Fonte / Projeto |
| :--- | :--- | :--- |
| [[manual-integracao-tissia-ia-v1\|Manual de Integração Tissia IA v1]] | Contrato oficial HTTP/JSON da API `v1` (`extra="forbid"`, códigos de status, headers `X-Tissia-*`, chaves `tsk_*`) | `tissia-ia` |
| [[mapeamento-pep-tissia-ia\|Mapeamento PEP ↔ Tissia IA]] | Como o `SpikeAdapter` do PEP traduz entidades de domínio para os endpoints da API `v1` e lista inconsistências | `ecossistema` |
| [[validacao-integracao-tissia-ia\|Validação de Integração]] | Roteiro de validação ponta a ponta da integração | `tissia-ia` |
| [[ensaios-postman-api-v1\|Ensaios Postman API v1]] | Cenários de chamada e verificação manual dos endpoints `v1` | `tissia-ia` |
| [[MOC-Index#01 — Contratos de API e Propostas SPK\|Propostas SPK (01..13)]] | Lacunas de contrato levantadas pelo PEP que evoluem para `RFC-IA-*` | `ecossistema` |

## 2. Onde Moram os Contratos no Código

### 2.1 Lado Servidor (`projetos/tissia-ia`)
- **Schemas Pydantic v2 (`extra="forbid"`):** `projetos/tissia-ia/app/schemas.py`
- **Rotas FastAPI (`v1` e operação interna `/ops/v1`):**
  - `projetos/tissia-ia/app/api/guias.py` (`POST /v1/guias/analisar`)
  - `projetos/tissia-ia/app/api/analises.py` (`POST /v1/analises/{id}/feedback`)
  - `projetos/tissia-ia/app/api/consultas.py` (`POST /v1/consultas/transcrever`, `estruturar`, `estruturacoes/{id}/feedback`)
  - `projetos/tissia-ia/app/api/desfechos.py` (`POST /v1/desfechos`, `/v1/analises/{id}/desfecho`)
  - `projetos/tissia-ia/app/api/preditivo.py` (`POST /v1/tiss/lotes/ingerir`, `/contratos/auditar`, `/guias/esquecidas`)
  - `projetos/tissia-ia/app/api/catalogos.py` (`GET /v1/catalogos/tuss`, `/tuss/{codigo}`, `/cid`, `/cid/{codigo}`)
  - `projetos/tissia-ia/app/api/bases.py` (`GET /v1/bases/versoes`)
  - `projetos/tissia-ia/app/api/clinicas.py` (`PUT/GET/DELETE /v1/clinicas*`, `/v1/clinicas/{ref}/indicadores`, `PUT /v1/integrador/configuracao`)
  - `projetos/tissia-ia/app/api/glosas.py` (`POST /v1/glosas/recurso`, `/recurso/{id}/desfecho`, `/recurso/{id}/feedback`)
  - `projetos/tissia-ia/app/api/jobs.py` (`GET /v1/jobs/{job_id}`, `PUT /v1/integrador/webhook`, `POST /v1/integrador/webhook/teste`)
  - `projetos/tissia-ia/app/api/standalone.py` (`GET /v1/uso`, `GET/POST/DELETE /v1/integrador/chaves`)
  - **Operação interna (`/ops/v1`, JWT Keycloak JWKS — contrato separado em `/ops/openapi.json`):**
    - `projetos/tissia-ia/app/api/ops.py` (`GET /ops/v1/versoes`)
    - `projetos/tissia-ia/app/api/ops_observabilidade.py` (`GET /ops/v1/saude`, `/analises`, `/analises/{id}`, `/integradores`, `/integradores/{id}/uso`, `POST /ops/v1/integradores/{id}/chaves/{chave_id}/revogacao`)
    - `projetos/tissia-ia/app/api/ops_estado.py` (`GET /ops/v1/roteamento`, `/prompts`, `/avaliacoes`, `/bases`, `/regras`, `/feedback/agregado`)
- **Snapshot OpenAPI:** `projetos/tissia-ia/tests/snapshots/openapi_v1.json`

### 2.2 Lado Cliente / PEP (`projetos/PEP`)
- **Contratos Isomórficos Zod (`@pep/contracts`):** `projetos/PEP/packages/contracts/src/`
  - Destaques relacionados à IA e faturamento: `analise.ts`, `escuta.ts`, `atendimento.ts`, `contratos.ts`, `faturamento.ts`, `validacao-guia.ts`, `identidade.ts`, `lotes.ts`, `catalogo.ts`, `operadoras.ts`, `comum.ts`, `rotas.ts`
  - **Schemas da indicação única e autopreenchimento da guia** ([[plano-guia-indicacao-unica-e-autopreenchimento|plano]], [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]], [[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028]]):
    - `atendimento.ts` (na fundação `feat/guia-fundacao`): `DefinirIndicacaoPedidosSchema` (`DefinirIndicacaoPedidosInput`), `AtendimentoSchema.indicacaoPedidos` e `CriarPedidoExameSchema.indicacaoClinica` opcional.
    - `validacao-guia.ts` (na fundação `feat/guia-fundacao`): `OrigemDadoGuiaSchema` (`OrigemDadoGuia`) e `SugestaoDadosGuiaSchema` (`SugestaoDadosGuia`).
    - `contratos.ts` (na fundação `feat/guia-fundacao`): `codigoContratado` em `CriarContratoSchema`, `AtualizarContratoSchema`, `ContratoResumoSchema` e `ContratoDetalheSchema`.
    - `identidade.ts` (*em implementação na fatia G5 — a confirmar na integração*): schema de dados profissionais (`conselho`, `numeroConselho`, `ufConselho`, `cbo`).
- **Novas rotas da API interna do PEP (`@pep/api`, *em implementação — a confirmar na integração*):**
  - `PUT /api/v1/atendimentos/{id}/indicacao-pedidos` (`DefinirIndicacaoPedidosSchema`, fatia G1)
  - `GET /api/v1/guias/{id}/sugestao-dados` (`SugestaoDadosGuiaSchema`, fatia G3)
  - `GET` / `PUT` `/api/v1/me/dados-profissionais` (dados profissionais e `cbo`, fatia G5)
- **Módulo Anticorrupção (`SpikeAdapter` em `@pep/api`):** `projetos/PEP/apps/api/src/modulos/tissia-ai/`
  - `tissia-ai.client.ts`, `tissia-ai.service.ts`, `tissia-ai.controller.ts`, `catalogo.service.ts`, `schema.ts`
  - **Nota:** O contrato HTTP/JSON com a `tissia-ia` (`POST /v1/guias/analisar`) **não muda** com a indicação única e o autopreenchimento: o campo `indicacao_clinica` continua sendo enviado no mesmo formato (`string | null`), alterando apenas a origem interna no `PEP`.
- **Snapshot OpenAPI do PEP:** `projetos/PEP/apps/api/openapi/openapi.json` (regenerar com `pnpm --filter @pep/api openapi` na integração)

## 3. Protocolo Obrigatório Antes de Alterar Contratos

1. **Ler primeiro:** [[manual-integracao-tissia-ia-v1]] + [[mapeamento-pep-tissia-ia]] + `openapi_v1.json`.
2. **Alinhar ambos os lados:** Toda alteração de campo em `tissia-ia/app/schemas.py` deve respeitar `extra="forbid"` e manter retrocompatibilidade aditiva em `/v1`, atualizando `openapi_v1.json` e verificando o impacto em `tissia-ai.client.ts` e `@pep/contracts`.
3. **Atualizar a documentação:** Registrar imediatamente na seção de histórico/endpoints do [[manual-integracao-tissia-ia-v1]], no [[mapeamento-pep-tissia-ia]] e na RFC correspondente (`RFC-IA-*` / `RFC-PEP-*`).

