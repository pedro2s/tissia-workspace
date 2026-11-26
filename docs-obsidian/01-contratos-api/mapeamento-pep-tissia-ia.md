---
projeto: ecossistema
tipo: contrato-api
status: ativo
tags:
  - tissia
  - ecossistema
  - contrato-api
  - spike-adapter
  - integracao
aliases:
  - README
  - spike-readme
  - a-spike-no-pep-tissia
origem: tissia/docs/spike/README.md
---

# A Spike no PEP Tissia

A **Spike** é o motor de IA anti-glosa do repositório `tissia-ai` (API `Tissia IA v1`, FastAPI no Cloud Run `southamerica-east1`). No PEP ela é uma integração nativa. Também é vendida à parte para outros PEPs ([[PRD-002-spike-standalone|PRD-002]]).

Fonte de verdade do contrato: [[manual-integracao-tissia-ia-v1|manual de integração da Spike]] e o snapshot `tests/snapshots/openapi_v1.json` do repo `tissia-ai`. Este documento registra **como o PEP usa** esse contrato e **o que falta** nele.

> **Convenção de nomenclatura.** *Spike* foi apenas o nome da PoC inicial e *MedScribe* o nome anterior do produto; o motor oficial é o **Tissia AI** (repositório `tissia-ai`). O contrato oficial com o Tissia AI usa **exclusivamente** chaves de API `tsk_test_` (sandbox, local e staging) e `tsk_live_` (produção) e headers `X-Tissia-*` (`X-Tissia-Cota-Restante`, `X-Tissia-Assinatura`, `X-Tissia-Evento`, `X-Tissia-Entrega`). Os prefixos e headers antigos (os que começavam com `msk` e com `X-Spike`) foram removidos do `tissia-ai` e não valem mais. No código e no banco deste repositório tudo usa o nome novo: módulo `tissia-ai`, variáveis `TISSIA_AI_*`, tabelas `analises_tissia_ai` e `tissia_ai_config_*`, evento `AnaliseTissiaAiConcluida`, origem de revisão `sugestao_tissia_ai` e consumidor `tissia-ai.feedback` (migração `0034`, que migra também as linhas existentes). Seguem com o nome da época só o que não se reescreve ou depende de fora do repositório: as ADRs aceitas (imutáveis), o nome do arquivo de migrações já aplicadas (`0004_analises_spike`), o segredo de CI `SPIKE_DEPLOY_KEY` (o workflow já aceita `TISSIA_AI_DEPLOY_KEY`, que precisa ser criado no GitHub) e os nomes de arquivo da documentação (`docs/spike/`, `*-spike-*`), a renomear junto com os links.

## 1. Status da Spike por fase

| Fase (Spike) | Escopo | Status | Usado pelo PEP em |
| :--- | :--- | :--- | :--- |
| F0 — Fundações | Auth por integrador, scrubber de PII, bases TUSS/CID/DUT, harness | Concluída | P1 |
| F1 — Motor MVP | `analisar`, regras, DUT, justificativa Campo 23, sugestão TUSS/CID, `feedback` | Concluída | P1 |
| F2 — Piloto | Desfechos (individual e lote), regras customizadas, relatório (script) | Concluída | P2 (desfechos); P1 (modo silencioso) |
| F3 — Preditivo | `score_risco`, ingestão XML TISS, auditoria de contratos, guias esquecidas | Concluída | P1 (`score_risco`); P2 (demais) |
| F4 — Scribe | Estruturação da consulta, streaming | Em implementação: pós-consulta em trechos, `documentos`, `analise` e `feedback` entregues; streaming adiado para Fase 2 ([[RFC-IA-009-scribe\|Spike RFC-009]], [[ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai\|ADR-PEP-0025]]) | Marco D / P2 (telas 06–08), via [[SPK-04-requisitos-do-scribe\|SPK-04]] |
| F4+ — Catálogo & RAG | Catálogo unificado TUSS/CID-10 (`SPK-09`), RAG `pgvector` e vigência TISS `04.03.00` (`SPK-10`) | Concluída na `tissia-ia` ([[RFC-IA-011-catalogo-terminologia-e-rag\|RFC-011]], [[RFC-IA-012-treinamento-sft-vertex-e-rag-spsadt\|RFC-012]], [[RFC-IA-013-refinamento-vigencia-ans-e-tiss-403\|RFC-013]]; pendente recuradoria CMO das DUTs) | P1 Onda 4 (`catalogo.service.ts`) e P2 (`ingerir`) |
| Operação — `/ops/v1` | API interna de observabilidade e revogação de chaves (`SPK-06` Fase 1) | Fase 1 concluída na `tissia-ia` (`GET /avaliacoes` em prod pendente; [[RFC-IA-014-api-de-operacao-interna\|RFC-014]], [[ADR-IA-0012-ops-v1-keycloak-jwks\|ADR-0012]]) | Backoffice B2 ([[RFC-PEP-020-backoffice-operacao-da-spike\|RFC-PEP-020]], integração pendente) |
| F5 — Extensão Chrome | Autofill em PEPs de terceiros | Rascunho ([[RFC-IA-010-extensao-chrome\|Spike RFC-010]]) | Não se aplica (canal de aquisição) |

## 2. Endpoints usados pelo PEP

| Endpoint Spike | Evento ou tela no PEP | Chamada | Fase PEP | RFC PEP |
| :--- | :--- | :--- | :--- | :--- |
| `POST /v1/guias/analisar` | Fechar atendimento (tela 05); salvar guia (13/17); conferência do faturista antes do lote (16) | Síncrona, timeout 6 s | P1 | [[RFC-PEP-005-spike-adapter\|005]], [[RFC-PEP-006-guias-e-lotes-tiss\|006]] |
| `POST /v1/guias/sugerir` e `POST /v1/guias/sugerir/{sugestao_id}/feedback` | Sugestão proativa de CID, TUSS e indicação clínica durante o atendimento (inclusive particular, `SPK-14`) | Síncrona (timeout 6 s) + feedback assíncrono | P2 | [[RFC-PEP-004-prontuario\|004]], [[RFC-PEP-005-spike-adapter\|005]], [[RFC-PEP-028-jornada-hibrida-grande-clinica-e-sae-enfermagem\|028]] |
| `POST /v1/analises/{id}/feedback` | Aceitar / Editar / Rejeitar no painel de revisão (12, 13) e na mesa do faturista (16, 17) | Fila (assíncrona, retry) | P1 | 005 |
| `GET /v1/catalogos/tuss`, `/tuss/{codigo}`, `GET /v1/catalogos/cid` | Busca incremental e validação de TUSS/CID no pedido de exame (04/05) e guia SP/SADT (13/17) | Síncrona com cache em memória e fallback local (`catalogo.service.ts`) | P1 (Onda 4) | [[RFC-PEP-004-prontuario\|004]], 005, 006 |
| `POST /v1/consultas/transcrever` e `POST /v1/consultas/estruturar` | Escuta ambiental (06), revisão trecho a trecho (07) e resumo ao paciente (08) | Síncrona, timeout 40 s; sem IA devolve `degradado` | Marco D (antecipada da P2) | [[RFC-PEP-013-voz-ambiental\|013]] |
| `POST /v1/consultas/resumo-paciente` | Sugestão de resumo ao paciente a partir da evolução SOAP (05/08), inclusive particular (`SPK-15`) | Síncrona, timeout 6 s; sem IA devolve `degradado` | P2 | [[RFC-PEP-025-estacao-enfermagem-assinatura-exames-resumo-e-baixa-checkout\|025]], [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia\|026]] |
| `POST /v1/consultas/sugerir-documentos` | Sugestão de rascunhos revisáveis de receita e atestado a partir do SOAP escrito (tela 08 — Documentos, `pedro2s/tissia-ai#14` + `pedro2s/tissia#65`) | Síncrona, timeout 6 s; fallback local determinístico no PEP/Spike | Onda P2 | [[RFC-PEP-005-spike-adapter\|005]], [[RFC-PEP-025-estacao-enfermagem-assinatura-exames-resumo-e-baixa-checkout\|025]] |
| `POST /v1/desfechos` e `POST /v1/analises/{id}/desfecho` | Importação do retorno (19); baixa manual de glosa (20) | Fila | P2 | [[RFC-PEP-007-retorno-e-glosas\|007]] |
| `POST /v1/tiss/lotes/ingerir` | Lote enviado (18) e demonstrativo recebido (19) | Fila, idempotente por `lote_ref` | P2 | 007 |
| `POST /v1/contratos/auditar` | Cadastro/atualização de tabela negociada (21); rotina semanal | Fila | P2 | [[RFC-PEP-008-contratos\|008]] |
| `POST /v1/guias/esquecidas` | Rotina diária agenda × guias; widget "receita esquecida" (22) | Fila (job diário) | P2 | [[RFC-PEP-009-receita-esquecida\|009]] |

> **Nota sobre endpoints ainda não consumidos pelo PEP:** `GET /v1/bases/versoes`, `POST /v1/consultas/estruturacoes/{id}/feedback`, os endpoints de `SPK-01`/`02`/`03`/`05` e a API interna `/ops/v1/*` (`SPK-06` Fase 1) já estão disponíveis na `tissia-ia` e aguardam integração nos fluxos correspondentes do PEP e do Backoffice B2 (`RFC-PEP-020`).

Toda chamada passa pelo **SpikeAdapter** ([[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]]). Nenhum outro módulo do backend fala com a Spike.

## 3. Mapeamento PEP → Spike

### 3.1 Identidade e credenciais

| Conceito PEP | Spike | Regra |
| :--- | :--- | :--- |
| Integrador (o PEP Tissia, por ambiente) | API key `tsk_test_…` / `tsk_live_…` (`Authorization: Bearer`) | Uma chave por ambiente (`staging`, `prod`), no Secret Manager. Rotação documentada em [[SPK-05-empacotamento-standalone\|SPK-05]] |
| Unidade | `clinica_ref` | `clinica_ref` = id estável da **Unidade** (não da Clínica), porque contrato, operadoras e numeração de lote são por unidade. Formato opaco: `un_<uuid>` |
| Paciente | `paciente.ref` | `"pa_" + base32(HMAC-SHA256(paciente.id, sal_da_clinica))[0:24]` ([[RFC-PEP-005-spike-adapter\|RFC-PEP-005]]). Sal por Clínica, guardado cifrado. Vínculo em `VinculoSpike` cifrado no PEP ([[modelo-de-dados-pep\|modelo de dados]]) |
| Guia | `guia_ref` | Id estável da Guia, igual em todas as revisões (`gu_<uuid>`). Nunca o número da guia na operadora |
| Atendimento | `atendimento_ref` (esquecidas) | `at_<uuid>` |
| Lote | `lote_ref` | Número do lote no PEP; no retorno, o protocolo da operadora |
| Requisição | `X-Request-Id` | Propagar o trace id do PEP |

### 3.2 `POST /v1/guias/analisar`

| Campo Spike | Origem no PEP | Observação |
| :--- | :--- | :--- |
| `clinica_ref` | `Unidade.id` | Ver 3.1 |
| `guia_ref` | `Guia.id` | Mesmo valor em toda revisão |
| `tipo_guia` | `Guia.tipo` (`consulta`, `sp_sadt`, `internacao`) | `internacao` só a partir da P3 |
| `operadora.registro_ans` | `Beneficiario.operadora.registro_ans` | Só o registro ANS |
| `especialidade` | `Profissional.especialidade` do atendimento | Texto em minúsculas (`ortopedia`, `cardiologia`), até 64 |
| `paciente.ref` | `VinculoSpike.paciente_ref` | Ver 3.1 |
| `paciente.sexo` | `Paciente.sexo_tiss` (`M`, `F`, `I`) | Sexo do cadastro TISS, não identidade de gênero |
| `paciente.idade` | Idade em anos completos na data do atendimento | Calculada; nunca enviar a data de nascimento |
| `evolucao_clinica` | Texto da `Evolucao` vigente do atendimento (SOAP concatenado) | Até 20.000 caracteres. Truncar priorizando o atendimento atual; depois evoluções anteriores relevantes, mais recentes primeiro |
| `cids` | `GuiaCID.codigo` da revisão | Sem ponto ou com ponto, como a Spike aceita |
| `procedimentos[]` | `GuiaProcedimento` (`codigo_tuss`, `quantidade`) | Quantidade 1–100 |
| `indicacao_clinica` | Campo 23 atual da revisão | Até 5.000 na Spike; o PEP limita a 500 (TISS) |
| `opcoes.gerar_justificativa` | `true` em SP/SADT, `false` em consulta | |
| `opcoes.sugerir_codigos` | `true` | |

Persistência da resposta: `AnaliseTissiaAi` (`analise_id`, `versao_bases`, `risco`, `score_risco`, `degradado`, resposta completa) vinculada à `GuiaRevisao`; filhos `AlertaAnalise`, `CriterioDUT`, `SugestaoCodigo`.

### 3.3 `POST /v1/analises/{id}/feedback`

| Campo Spike | Origem no PEP |
| :--- | :--- |
| `papel` | Perfil de quem decidiu: `medico` ou `faturista` (outros perfis não decidem) |
| `itens[].item_id` | `alertas[].id`; `"<codigo_tuss>:justificativa"` ou `"guia:justificativa"`; `"sug_cid:<codigo>"`; `"sug_tuss:<codigo>"` (no PEP, decisões sobre itens de CID só são aceitas do médico titular — `403` para os demais; ver [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia\|RFC-PEP-026]]) |
| `itens[].acao` | `DecisaoUsuario.acao` (`aceito`, `editado`, `rejeitado`) |
| `itens[].texto_final` | Texto final do Campo 23 quando `editado` (≤ 5.000; o PEP já limita a 500) |

A decisão é gravada primeiro no PEP (`DecisaoUsuario`, com usuário e horário) e depois enviada pela fila. Até 100 itens por chamada.

### 3.4 `POST /v1/desfechos` e `/v1/analises/{id}/desfecho`

| Campo Spike | Origem no PEP |
| :--- | :--- |
| `lote_ref` | Protocolo do retorno / número do lote |
| `itens[].analise_id` | Última `AnaliseTissiaAi` da revisão enviada. Se ausente, usar o par abaixo |
| `itens[].clinica_ref` + `guia_ref` | `Unidade.id` + `Guia.id` |
| `itens[].status` | `Desfecho.status` (`pago`, `glosado_parcial`, `glosado_total`) |
| `itens[].valor_cobrado` / `valor_glosado` | Do demonstrativo |
| `itens[].codigo_glosa_tuss` | Código da Tabela 38 do demonstrativo **[validar numeração]** |
| `itens[].motivo_glosa_texto` | Texto da operadora (a Spike faz scrub antes de gravar) |

Até 1.000 itens por chamada; o PEP pagina.

### 3.5 `POST /v1/tiss/lotes/ingerir`

| Campo Spike | Origem no PEP |
| :--- | :--- |
| `clinica_ref` | `Unidade.id` |
| `xml_conteudo` | XML do `Lote` enviado ou do `RetornoOperadora` recebido, como texto |

Limite de 5.000.000 caracteres e chamada síncrona hoje (ver achado 5). O XML de envio contém nome e carteirinha do beneficiário; a Spike declara que não lê essas tags ([[RFC-IA-008-preditivo-historico-e-contratos|Spike RFC-008]] §2). **Decisão do PEP:** remover as tags de identificação direta antes do envio mesmo assim (minimização no lado do controlador).

### 3.6 `POST /v1/contratos/auditar`

| Campo Spike | Origem no PEP |
| :--- | :--- |
| `clinica_ref` | `Unidade.id` |
| `registro_ans` | `Contrato.operadora.registro_ans` |
| `itens_contrato[]` | `TabelaPreco` vigente do contrato (`codigo_tuss`, `valor_contratado`, `exige_autorizacao_previa` de `RegraAutorizacao`, `observacoes`) |
| `itens_pep[]` | Parametrização atual do faturamento da unidade (valor configurado, exige autorização) |
| `provisionar_regras_autorizacao` | `true` |

Até 2.000 itens por chamada; contratos maiores são paginados por faixa de TUSS.

### 3.7 `POST /v1/guias/esquecidas`

| Campo Spike | Origem no PEP |
| :--- | :--- |
| `clinica_ref` | `Unidade.id` |
| `atendimentos[].atendimento_ref` | `Atendimento.id` |
| `atendimentos[].paciente_ref` | `VinculoSpike.paciente_ref` |
| `atendimentos[].data_atendimento` | Data do atendimento (`YYYY-MM-DD`) |
| `atendimentos[].registro_ans` | Operadora do beneficiário usado no agendamento |
| `atendimentos[].codigo_tuss` | Procedimento agendado/realizado |
| `atendimentos[].guia_ref` | `Guia.id` vinculada, ou `null` |
| `atendimentos[].valor_estimado` | Valor da `TabelaPreco` vigente |

Particulares não entram (não há guia TISS).

## 4. Regras de contrato que o PEP precisa honrar

| Regra | Limite | Onde o PEP garante |
| :--- | :--- | :--- |
| Corpo de `analisar` | ≤ 64 KB (413) | SpikeAdapter mede o JSON serializado e trunca a evolução antes de enviar |
| `evolucao_clinica` | ≤ 20.000 caracteres | Truncamento com prioridade para o atendimento atual |
| Campo 23 TISS | ≤ 500 caracteres | Editor com contador e bloqueio (tela 13, 17); validação também no backend. Nunca truncado: acima de 500 a guia nasce sem indicação e o faturamento é bloqueado no campo 23 (mínimo de 15 caracteres em SP/SADT com procedimento) |
| `indicacao_clinica` na Spike | ≤ 5.000 | Já coberto pelo limite de 500 |
| Campos extras | `extra="forbid"` → 422 | DTOs tipados espelhando o schema; teste de contrato contra o snapshot OpenAPI ([[test-plan-pep\|test plan]]) |
| `item_id` do feedback | Convenção fixa (3.3) | Gerado pelo adapter a partir da resposta, nunca digitado |
| Timeout | 6 s no cliente | Estouro = segue sem alertas, guia marcada "não analisada" e reenfileirada |
| `degradado: true` | Resposta parcial válida | UI mostra "análise parcial (regras TISS/ANS ativas, IA clínica indisponível)" (tela 14) |
| `aviso_legal` | Sempre exibido | Rodapé fixo do painel de revisão |
| Nada vai à operadora sem aceite | Spike ADR-0008 | Justificativa e códigos só entram na guia após ação humana registrada |

## 5. Inconsistências encontradas na Spike

| # | Achado | Evidência | Impacto no PEP | Situação atual / Recomendação |
| :--- | :--- | :--- | :--- | :--- |
| 1 | ~~O release plan diz que `modo` padrão é `sombra`, mas o código usa `visivel`~~ **Resolvido na doc e na `tissia-ia`** (`release-plan-ia §4` e `PUT /v1/integrador/configuracao` via `SPK-01`) | [[release-plan-ia\|Spike release-plan §4]]; `app/api/guias.py:230`; `app/api/clinicas.py` | Clínica nova recebe resposta completa por padrão ([[ADR-PEP-0006-modos-silencioso-assistido\|ADR-PEP-0006]]) | Falta integrar `PUT /v1/integrador/configuracao` e `PUT /v1/clinicas/{ref}` no onboarding do PEP ([[SPK-01-api-de-clinicas-e-flags\|SPK-01]]) |
| 2 | ~~O manual de integração diz que a evolução nunca é armazenada~~ **Resolvido na doc** (`manual-integracao §2.3` explicita opt-in contratual) | [[manual-integracao-tissia-ia-v1\|integracao.md §2.3]]; [[ADR-IA-0007-minimizacao-de-dados\|Spike ADR-0007]] | RIPD/DPA do PEP agora alinhado com a retenção de 180 dias pós-scrubber sob opt-in | `SPK-01` exige `aceite_contratual_ref`; o PEP mantém `false` até haver opt-in assinado |
| 3 | ~~Flags só por script; autoprovisionamento implícito~~ **Resolvido na `tissia-ia`** (`SPK-01`) | `app/api/clinicas.py` (`PUT/GET/DELETE /v1/clinicas`, `PUT /v1/integrador/configuracao`) | O PEP ainda não provisiona unidades via API nem ativa `exigir_clinica_cadastrada` | Integrar `SPK-01` no módulo `tissia-ai` e no onboarding (`RFC-PEP-018`) |
| 4 | ~~Sem endpoint de recurso de glosa~~ **Resolvido na `tissia-ia` e integrado no PEP** (`SPK-02`, PR `pedro2s/tissia#38`) | `app/api/glosas.py` (`POST /v1/glosas/recurso`) ↔ `POST /api/v1/recursos/:id/sugestao` e `Glosas.tsx` | Tela 20 já consome sugestão de recurso por IA com fallback silencioso quando degradada | Concluído |
| 5 | ~~`tiss/lotes/ingerir` apenas síncrono~~ **Resolvido na `tissia-ia`** (`SPK-03`) | `app/api/jobs.py`, `app/api/preditivo.py` (`Prefer: respond-async` → `202` + webhooks) | PEP (`tissia-ai.client.ts`) ainda chama de forma síncrona | Adotar `Prefer: respond-async`, `Idempotency-Key` e webhook de retorno no PEP ([[SPK-03-processamento-assincrono\|SPK-03]]) |
| 6 | ~~Scribe em rascunho, sem endpoint~~ **Resolvido em parte em 08/11/2026** (pós-consulta, `documentos`, `analise` e `feedback` entregues; streaming adiado pela `ADR-PEP-0025`) | [[RFC-IA-009-scribe\|Spike RFC-009]] | Telas 06, 07 e o resumo ao paciente da tela 08 usam `consultas/transcrever` e `consultas/estruturar` | Streaming WebSocket ficará para Fase 2 após benchmark pt-BR ([[SPK-04-requisitos-do-scribe\|SPK-04]]) |
| 7 | A Spike RFC-002 diz que `internacao` é só F2+, mas o schema já aceita | Spike RFC-002 §Request; `app/schemas.py` (`TipoGuia`) | Ambiguidade sobre o que é validado para internação | Documentar na Spike quais regras/DUTs cobrem internação. O PEP só envia `internacao` na P3 |
| 8 | ~~Relatório do piloto só por script~~ **Resolvido na `tissia-ia`** (`SPK-01`) | `app/api/clinicas.py` (`GET /v1/clinicas/{clinica_ref}/indicadores`) | Painel do gestor (tela 24) no PEP calcula localmente e não depende da Spike | PEP já calcula em [[RFC-PEP-012-painel-do-gestor\|RFC-PEP-012]]; endpoint fica disponível para clientes standalone e conferência |

## 6. Propostas para a Spike

| # | Proposta | Necessária para | Situação (`tissia-ia` ↔ `PEP`) |
| :--- | :--- | :--- | :--- |
| [[SPK-01-api-de-clinicas-e-flags\|SPK-01]] | API de clínicas, flags e indicadores | Onboarding de unidade no PEP; venda standalone | **Implementada na `tissia-ia`** (`app/api/clinicas.py`); falta integrar no PEP (`RFC-PEP-018`) |
| [[SPK-02-recurso-de-glosa\|SPK-02]] | Recurso de glosa assistido | Tela 20 (`RFC-PEP-007`) | **Implementada na `tissia-ia` e integrada no PEP** (`POST /api/v1/recursos/:id/sugestao` + `Glosas.tsx`, PR `pedro2s/tissia#38`) |
| [[SPK-03-processamento-assincrono\|SPK-03]] | Jobs assíncronos e webhooks | XML grande, rotinas diárias | **Implementada na `tissia-ia`** (`app/api/jobs.py`, `preditivo.py`, `desfechos.py`); falta integrar no PEP |
| [[SPK-04-requisitos-do-scribe\|SPK-04]] | Requisitos do PEP para o Scribe | Telas 06, 07, 08 (`RFC-PEP-013`) | **Implementada em parte** (pós-consulta, `documentos`, `analise`, `feedback`); consumo de `documentos` + indicação validada sem CID integrado no PEP em `pedro2s/tissia#62`; streaming adiado (`ADR-PEP-0025`) |
| [[SPK-05-empacotamento-standalone\|SPK-05]] | Empacotamento standalone | [[PRD-002-spike-standalone\|PRD-002]] | **Implementada na `tissia-ia`** (`app/api/standalone.py`, cota `X-Tissia-Cota-Restante`, `429`); portal/DPA comercial fora do PEP |
| [[SPK-06-api-de-operacao\|SPK-06]] | API de operação interna (`/ops/v1`) | [[PRD-003-backoffice\|PRD-003]] (Backoffice B2) | **Fase 1 implementada na `tissia-ia`** (`RFC-IA-014`) e **consumida no Backoffice B2** (`RFC-PEP-020`, `spike-operacao.client.ts` + `/bo/v1/ops/*`, PR `pedro2s/tissia#41`) |
| [[SPK-07-latencia-da-analise\|SPK-07]] | Latência da análise com IA (p95 ≤ 5 s) | Release plan §12.8 | **Implementada na `tissia-ia`** (paralelização `asyncio.gather` + `X-Tissia-Tempos-Ms`) e **mitigada no PEP** (2 fases); pendente validar p95 em prod |
| [[SPK-08-fallback-local-sinalizado\|SPK-08]] | Fallback local da IA sinalizado (`degradado`/`origem_ia`) | Transparência da IA ao médico; `taxa_degradado` do piloto | **Implementada na `tissia-ia`** (`origem_ia` em `app/api/guias.py`); falta aceitar `origem_ia` e exibir selo no PEP |
| [[SPK-09-catalogo-tuss\|SPK-09]] | Catálogo TUSS e CID-10 (listagem e busca) | Pedido de exame sem lista fixa no PEP (Onda 4) | **Implementada na `tissia-ia` (`RFC-IA-011`) e consumida no PEP** (`catalogo.service.ts`) |
| [[SPK-10-tiss-4-03\|SPK-10]] | Leitura de XML TISS 4.03.00 no `ingerir` | Retorno e glosas com a versão vigente | **Implementada na `tissia-ia` (`RFC-IA-013`)** (`app/tiss_xml.py` suporta `04.03.00`); envio no PEP já ativo (`tissia-ai.client.ts`) |
| [[SPK-11-contratos-auditar-substituicao\|SPK-11]] | Substituição de tabela e regras em `contratos/auditar` | Ligar `provisionar_regras_autorizacao` com segurança (`RFC-PEP-008` Q3) | **Proposta** (PEP envia `provisionar_regras_autorizacao = false` até implementação na `tissia-ia`) |
| [[SPK-12-desfecho-recurso-manual\|SPK-12]] | Desfecho de recurso manual sem sobrescrever desfecho original | Preservar sinal de glosa no preditivo para recursos manuais (`RFC-PEP-007` Q8) | **Proposta** (aguardando implementação na `tissia-ia` e envio pelo PEP) |
| [[SPK-13-ingestao-mapa-guia-ref\|SPK-13]] | Mapa `numeroGuiaPrestador` ↔ `guia_ref` em `tiss/lotes/ingerir` | Casar histórico ingerido com análises e receita esquecida (`RFC-PEP-007` Q7) | **Proposta** (aguardando implementação na `tissia-ia` e envio pelo PEP) |
| [[SPK-14-sugestao-de-codigos-e-indicacao-a-partir-da-evolucao\|SPK-14]] | `POST /v1/guias/sugerir`: CID, TUSS e indicação a partir da evolução, sem guia, com `operadora` opcional e feedback | Sugestão proativa durante o atendimento (atendimento particular e convênio em 1 chamada) | **Implementada na `tissia-ia` (`pedro2s/tissia-ai#9`, v2 em `#13` com `tipo` e `dut_resumo`) e integrada no PEP (`pedro2s/tissia#38`, governança + feedback + contexto rico em `#61` e UX 1-clique em `#59`)** |
| [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas\|SPK-15]] | `metadados_ia` (CFM 2.454/2026), `POST /v1/consultas/resumo-paciente` via SOAP e vigência/tabelas 18/19/20 em `GET /v1/catalogos/tuss` | Trilha auditável de IA (`RFC-PEP-026`), resumo ao paciente sem Scribe (`RFC-PEP-025`) e vigência TUSS / outras despesas (`RFC-PEP-027`) | **Implementada na `tissia-ia`** (`app/api/{guias,consultas,glosas,catalogos}.py`) e consumida no PEP (`catalogo.service.ts`, `tissia-ai.service.ts`) |

Todas são aditivas em `/v1`. As propostas `SPK-09` e `SPK-10` já evoluíram para `RFC-IA-011` e `RFC-IA-013`, e `SPK-06` Fase 1 evoluiu para `RFC-IA-014` em `tissia-ia`.

### Onda P3 (2026-11-25) — contratos adicionados em `@pep/contracts`

- `sae-assistida.ts`, `ranking-glosas.ts` e `onda-p3-whatsapp.ts` publicam os schemas Zod `.strict()` das rotas internas do PEP (SAE assistida, ranking de glosas e envio de resumo ao paciente). São **contratos PEP ↔ UI**, sem contrapartida na API `v1` do `tissia-ia`: nenhuma rota, payload ou enum desta integração mudou, portanto não há atualização de snapshot `openapi_v1.json` nem nova `SPK-*` pendente.
- A SAE assistida é determinística (template local, sem LLM) e o envio via WhatsApp usa provedor `stub`/`fake` com `WHATSAPP_ENABLED=false` por padrão; nenhum dado clínico é enviado ao motor de IA nesses fluxos.
