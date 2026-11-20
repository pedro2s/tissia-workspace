---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-005-spike-adapter.md
---

# RFC-PEP-005 — SpikeAdapter: integração nativa com a Spike anti-glosa

**Fase:** P1 · **Status:** Em implementação · **ADRs:** ADR-PEP-0005, ADR-PEP-0006, ADR-PEP-0008, ADR-PEP-0009 · Spike ADR-0005, ADR-0007, ADR-0008
**Telas:** 12 (Painel anti-glosa), 13 (Revisão da guia SP/SADT), 14 (Análise parcial e estados), 15 (Modo silencioso × assistido); painel lateral da 05 e coluna de risco da 16
**Requisitos:** RF-SPK-01 a RF-SPK-14
**Depende de:** RFC-PEP-004 (evolução, `AtendimentoFechado`), RFC-PEP-006 (guia e `GuiaRevisao`), Spike `v1` em `staging` ([[manual-integracao-tissia-ia-v1|manual de integração]])

## Objetivo

Um módulo isolado (`spike`) concentra toda a comunicação com a Spike: pseudonimiza, monta o payload exato do contrato `v1`, chama com timeout, degrada sem bloquear, persiste a análise e as decisões do usuário e devolve à UI um modelo próprio do PEP. Nenhum outro módulo conhece o formato da Spike.

## Escopo

**Dentro (P1)**

- [RF-SPK-01] Análise de guia (`POST /v1/guias/analisar`) nos gatilhos: fechar atendimento, salvar guia, conferência do faturista e checagem prévia do lote.
- [RF-SPK-02] Pseudonimização (`paciente.ref`, `clinica_ref`, `guia_ref`) e tabela `vinculos_tissia_ai`.
- [RF-SPK-03] DTOs estritos e teste de contrato contra o snapshot do OpenAPI da Spike.
- [RF-SPK-04] Truncamento da evolução dentro de 20.000 caracteres e do corpo de 64 KB.
- [RF-SPK-05] Timeout de 6 s, circuit breaker e modo degradado (tela 14); nova tentativa automática.
- [RF-SPK-06] Persistência: `analises_tissia_ai`, `alertas_analise`, `criterios_dut`, `sugestoes_codigo`, `decisoes_usuario`, ligadas à `guia_revisoes`.
- [RF-SPK-07] Painel de revisão (telas 12/13): risco, alertas por severidade, checklist DUT com evidência, Campo 23 editável com contador de 500, sugestões TUSS/CID em 1 clique, aviso legal fixo. Meta de revisão ≤ 15 s.
- [RF-SPK-08] Aplicar sugestão na guia (cria nova `GuiaRevisao`) e registrar a decisão.
- [RF-SPK-09] Feedback (`POST /v1/analises/{id}/feedback`) assíncrono, idempotente, com retry.
- [RF-SPK-10] Modos por unidade e exceção por profissional: `desligado`, `silencioso`, `assistido` (ADR-PEP-0006); opções do modo assistido (tela 15).
- [RF-SPK-11] Relatório "glosas que teriam sido evitadas" (tela 15) — o cálculo depende dos desfechos da P2; na P1 entrega a contagem de guias analisadas e alertas por motivo.
- [RF-SPK-12] Revalidação pelo faturista com o mesmo `guia_ref` (individual e em massa na tela 16).
- [RF-SPK-13] Métricas: latência, taxa de degradação, taxa de aceite por tipo de item, tempo de revisão.
- [RF-SPK-14] Chave do integrador por ambiente no Secret Manager; desligar a Spike por clínica sem deploy.

**Fora (P2, outras RFCs)**

- `POST /v1/desfechos`, `/v1/analises/{id}/desfecho` e `POST /v1/tiss/lotes/ingerir` → RFC-PEP-007.
- `POST /v1/contratos/auditar` → RFC-PEP-008.
- `POST /v1/guias/esquecidas` → RFC-PEP-009.
- Recurso de glosa assistido → SPK-02 + RFC-PEP-007. Estruturação de consulta por voz → SPK-04 + RFC-PEP-013.

## Design

### Estrutura do módulo

```
modules/spike/
  spike.client.ts          # HTTP (undici), timeout, breaker, headers, mapeamento de erros
  spike.dto.ts             # classes espelho do schema v1 (request e response), strict
  pseudonimizacao.ts       # HMAC, vinculos_tissia_ai
  payload.builder.ts       # GuiaRevisao + evolução → AnalisarGuiaRequest (inclui truncamento)
  analise.service.ts       # orquestra gatilho → chamada → persistência → evento
  feedback.processor.ts    # job BullMQ `spike-feedback`
  modo.service.ts          # resolve modo efetivo (unidade, profissional)
  spike.controller.ts      # rotas internas /api/v1/... usadas pela UI
```

Outros módulos só falam com `analise.service` (por evento ou chamada interna) e leem as tabelas de análise via API do módulo. `# ponytail:` sem interface/porta genérica para "provedor anti-glosa": há um só; trocar de fornecedor = reescrever este módulo.

### Pseudonimização (ADR-PEP-0005)

| Campo Spike | Origem no PEP | Regra |
| :--- | :--- | :--- |
| `clinica_ref` | `unidades.id` | `"un_" + unidades.id` (UUID da unidade não é dado pessoal). Estável; a Spike autoprovisiona a clínica no primeiro uso |
| `paciente.ref` | `pacientes.id` | `"pa_" + base32(HMAC-SHA256(paciente_id, sal_da_clinica))[0:24]` |
| `guia_ref` | `guias.id` | `"gu_" + guias.id` (UUID já é opaco). Igual em todas as revisões da guia |
| — | `vinculos_tissia_ai` | `(tenant_id, tipo, ref, clinica_id, entidade_id_cifrado)` cifrado; é o único jeito de ir do ref ao paciente |

Sal por **Clínica (`clinica_id` = CNPJ controlador)** fica no KMS, compartilhado entre todas as unidades daquela clínica (para que o paciente atendido em duas unidades do mesmo CNPJ tenha um único `paciente.ref`), mas distinto entre clínicas diferentes do mesmo tenant; rotação = novo sal + nova linha em `vinculos_tissia_ai` (refs antigos continuam resolvendo).

### Mapeamento do payload (`POST /v1/guias/analisar`)

| Campo Spike | Fonte no PEP | Observação |
| :--- | :--- | :--- |
| `clinica_ref` | unidade da guia | ver acima |
| `guia_ref` | guia | ver acima |
| `tipo_guia` | `guias.tipo` | `consulta` \| `sp_sadt` (`internacao` só na P3) |
| `operadora.registro_ans` | `operadoras.registro_ans` do beneficiário | particular não chama a Spike |
| `especialidade` | especialidade do profissional executante/solicitante | texto minúsculo, ≤ 64 |
| `paciente.ref` | ver acima | |
| `paciente.sexo` | `pacientes.sexo_tiss` | `M`/`F`/`I` |
| `paciente.idade` | `nascimento` × data do atendimento | anos completos, 0–130 |
| `evolucao_clinica` | `evolucao_versoes.texto_plano` da versão vinculada à revisão | truncada (abaixo) |
| `cids` | `guia_cids` da revisão | normalizado sem ponto (`M232`) **[validar formato aceito]** |
| `procedimentos[]` | `guia_procedimentos` da revisão | só `codigo_tuss` e `quantidade` (1–100); acima de 100 vira alerta local e não vai |
| `indicacao_clinica` | Campo 23 atual da revisão | ≤ 5.000 (o PEP já limita a 500) |
| `opcoes.gerar_justificativa` | `tipo = sp_sadt` e Campo 23 vazio ou pedido de nova sugestão | consulta → `false` |
| `opcoes.sugerir_codigos` | `true`, salvo flag da unidade | |

Nada além disso sai do PEP: nome, CPF, CNS, carteirinha, telefone, nome do profissional, CRM e valores ficam no PEP. O DTO de request é uma classe com `forbidNonWhitelisted` + teste de propriedade (ver Critérios) que garante que nenhum campo extra é serializado.

### Truncamento (RF-SPK-04)

Ordem de prioridade até 20.000 caracteres e ≤ 60 KB no corpo final (margem para o JSON):

1. Evolução do atendimento atual (SOAP completo).
2. Indicação clínica dos pedidos do atendimento.
3. Histórico recente do mesmo problema: últimas evoluções do paciente com o mesmo CID ou especialidade, da mais nova para a mais antiga, cada uma com cabeçalho `[Atendimento de DD/MM/AAAA]`.
4. Corte por parágrafo inteiro; se o item 1 sozinho passar do limite, mantém o início de S/O e o A/P inteiros e marca `truncada=true` na análise.

`# ponytail:` heurística de prioridade fixa; trocar por seleção por relevância se o eval mostrar critério DUT perdido por truncamento.

### Gatilhos e debounce

| Gatilho | Quem | Comportamento |
| :--- | :--- | :--- |
| `GuiaRascunhoCriada` (após `AtendimentoFechado`) | evento | Analisa a revisão inicial. Resultado aparece no painel do prontuário |
| Salvar guia (`GuiaRevisada`) | evento | Debounce de 3 s por `guia_id`: só a última revisão é analisada |
| "Revalidar" (`R`) na tela 16/17 | faturista | Chamada imediata, mesmo `guia_ref` |
| Checagem prévia do lote | job | Reanalisa guias com análise `parcial`, com revisão sem análise ou com `versao_bases` diferente da corrente |

Uma análise por `(guia_revisao_id, gatilho)`; repetir o mesmo gatilho na mesma revisão devolve a análise gravada (idempotência local).

### Chamada, timeout e erros

- `undici` com timeout total de 6 s, `Authorization: Bearer tsk_test_…` (sandbox, local e staging) ou `tsk_live_…` (produção), `X-Request-Id` = id da análise local (correlaciona logs dos dois lados).
- Circuit breaker por ambiente: abre com 50% de falhas/timeouts em 20 chamadas na janela de 60 s; meio-aberto após 30 s. Breaker aberto = análise local imediata como `parcial`.
- Concorrência: fila BullMQ `spike-analise` com limite de 10 chamadas simultâneas por unidade e 50 globais; a chamada do usuário espera até 6 s e, passando disso, a UI mostra "Analisando…" e recebe o resultado por SSE quando o job terminar.

| Resposta | Tratamento | Estado da análise |
| :--- | :--- | :--- |
| `200`, `degradado: false` | Persiste tudo | `completa` |
| `200`, `degradado: true` | Persiste o que veio; agenda nova tentativa em 30 s (máx. 3) | `parcial` |
| Timeout / `5xx` / rede | Regras locais (RFC-PEP-006) seguem valendo; agenda nova tentativa (30 s, 2 min, 10 min) | `falhou` → UI "análise parcial" |
| `401` / `403` | Alerta de operação (chave inválida/revogada); breaker aberto para o ambiente | `falhou` |
| `404` (feedback) | Loga e descarta o job | — |
| `413` | Retrunca com limite 50% menor e tenta uma vez; depois `falhou` | `falhou` |
| `422` | **Bug de contrato**: alerta de erro com o `detail` (sem payload), não retenta | `falhou` |

Nenhum desses casos bloqueia o atendimento nem o envio ao faturista (princípio "nunca bloquear"). O único bloqueio possível é por configuração do modo assistido ("impedir envio ao faturista com alerta bloqueante"), e só com análise `completa` ou com regra local bloqueante.

### Modelo de dados

```sql
CREATE TABLE vinculos_tissia_ai (
  tenant_id uuid NOT NULL, tipo text NOT NULL,     -- paciente | unidade
  ref text NOT NULL, clinica_id uuid, entidade_id_cifrado bytea NOT NULL, sal_versao int NOT NULL,
  PRIMARY KEY (tenant_id, tipo, ref)
);

CREATE TABLE analises_tissia_ai (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  guia_id uuid NOT NULL, guia_revisao_id uuid NOT NULL, -- sem FK cross-módulo (ADR-PEP-0017)
  gatilho text NOT NULL,                 -- fechamento | salvar | revalidar | pre_lote
  modo text NOT NULL,                    -- silencioso | assistido (desligado não grava)
  estado text NOT NULL,                  -- pendente | completa | parcial | falhou
  analise_id_tissia_ai uuid,                 -- analise_id devolvido
  versao_bases text, risco text, score_risco numeric(4,3), degradado boolean,
  evolucao_truncada boolean NOT NULL DEFAULT false,
  latencia_ms int, tentativas smallint NOT NULL DEFAULT 0,
  resposta jsonb,                        -- resposta integral (sem PII: a Spike não recebe PII)
  criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (guia_revisao_id, gatilho)
);

CREATE TABLE alertas_analise (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, analise_id uuid NOT NULL REFERENCES analises_tissia_ai(id),
  item_id text NOT NULL,                 -- alertas[].id da Spike
  tipo text NOT NULL, severidade text NOT NULL, procedimento_tuss text,
  mensagem text NOT NULL, fonte_tipo text, fonte_ref text, confianca numeric(4,3)
);
CREATE TABLE criterios_dut (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, analise_id uuid NOT NULL REFERENCES analises_tissia_ai(id), codigo_tuss text NOT NULL,
  dut_numero int, conformidade text, criterio_id text NOT NULL, texto text NOT NULL,
  status text NOT NULL, probabilidade numeric(4,3), evidencia text
);
CREATE TABLE sugestoes_codigo (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, analise_id uuid NOT NULL REFERENCES analises_tissia_ai(id),
  item_id text NOT NULL,                 -- sug_cid:<cod> | sug_tuss:<cod> | <tuss>:justificativa | guia:justificativa
  tipo text NOT NULL,                    -- cid | tuss | justificativa
  codigo text, texto text, situacao text, confianca numeric(4,3), evidencia text,
  alternativas jsonb NOT NULL DEFAULT '[]'
);
CREATE TABLE decisoes_usuario (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, analise_id uuid NOT NULL REFERENCES analises_tissia_ai(id),
  item_id text NOT NULL, acao text NOT NULL,   -- aceito | editado | rejeitado
  texto_final text, motivo_rejeicao text,      -- motivo exigido se a unidade configurar
  papel text NOT NULL,                         -- medico | faturista
  usuario_id uuid NOT NULL, decidido_em timestamptz NOT NULL DEFAULT now(),
  guia_revisao_resultante uuid,                -- revisão criada ao aplicar
  feedback_enviado_em timestamptz,
  UNIQUE (analise_id, item_id)
);

CREATE TABLE tissia_ai_config_unidade (
  unidade_id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  modo text NOT NULL DEFAULT 'desligado',      -- desligado | silencioso | assistido (unidade nova começa desligada: nada vai à Spike antes do DPA e do aceite da clínica; release-plan §4)
  bloquear_envio_com_bloqueante boolean NOT NULL DEFAULT false,
  exigir_motivo_ao_ignorar boolean NOT NULL DEFAULT false,
  mostrar_info_ao_medico boolean NOT NULL DEFAULT true,
  sugerir_codigos boolean NOT NULL DEFAULT true,
  inicio_linha_base date
);
CREATE TABLE tissia_ai_config_profissional (
  tenant_id uuid NOT NULL, profissional_id uuid NOT NULL, unidade_id uuid NOT NULL, modo text NOT NULL,
  PRIMARY KEY (tenant_id, profissional_id, unidade_id)
);
```

Retenção de `analises_tissia_ai.resposta`: enquanto existir a guia (prova de revisão anexada à trilha da assinatura).

### Modos (ADR-PEP-0006)

| Modo | Chama a Spike | Mostra na UI | Feedback | Uso |
| :--- | :--- | :--- | :--- | :--- |
| `desligado` | Não | Só regras locais | — | Clínica sem contrato da Spike, particular |
| `silencioso` | Sim (todos os gatilhos) | Não. Selo neutro "análise registrada" só para gestor | Não | Linha de base |
| `assistido` | Sim | Sim | Sim | Operação |

- O modo efetivo é `tissia_ai_config_profissional` se existir, senão o da unidade.
- A clínica na Spike fica sempre com `modo=visivel` (a Spike devolve tudo; quem esconde é o PEP), conforme ADR-PEP-0006. Configurar isso exige hoje o script `configurar_clinica.py` da Spike; o default de código já é `visivel`, mas o release plan da Spike fala em default `sombra`: confirmar e pedir a API de flags (SPK-01).

### API interna (para a UI)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/guias/{id}/analise` | Última análise da guia já no modelo do PEP (ou `estado: pendente`); `?revisao=` para histórico |
| `GET` | `/api/v1/guias/{id}/analise/stream` | SSE autenticado via `Authorization: Bearer` (`@microsoft/fetch-event-source` na UI; nunca token em query string): `pendente → completa/parcial/falhou` |
| `POST` | `/api/v1/guias/{id}/analise` | Revalidar `{ "gatilho": "revalidar" }` |
| `POST` | `/api/v1/analises/{id}/decisoes` | `{ "itens": [{ "item_id": "sug_cid:<CID_…>", "acao": "aceito", "aplicar": true }] }` → cria nova `GuiaRevisao` quando `aplicar` |
| `POST` | `/api/v1/guias/revalidar` | Em massa (tela 16): `{ "guia_ids": [...] }` → job, progresso por SSE |
| `GET/PUT` | `/api/v1/unidades/{id}/spike-config` | Tela 15 (perfil gestor) |
| `GET` | `/api/v1/unidades/{id}/spike-relatorio?de=&ate=` | Tela 15: guias analisadas, alertas por motivo; com desfechos (P2), glosas evitáveis por motivo e operadora |

Modelo devolvido à UI (resumo):

```json
{
  "estado": "completa", "modo": "assistido", "risco": "medio", "score_risco": 0.62,
  "versao_bases": "2026.11", "analisada_em": "2026-11-06T09:44:02-03:00", "latencia_ms": 2100,
  "alertas": [{ "item_id": "TISS-SADT-INDICACAO", "severidade": "bloqueante", "mensagem": "…", "fonte": "TISS", "acao_sugerida": "revisar_justificativa", "decisao": null }],
  "dut": [{ "codigo_tuss": "<TUSS_…>", "numero": null, "criterios": [{ "texto": "…", "status": "presente", "evidencia": "…", "ancora": "E1" }] }],
  "justificativa": { "item_id": "guia:justificativa", "texto": "…", "caracteres": 412, "decisao": null },
  "sugestoes": [{ "item_id": "sug_cid:<CID_…>", "tipo": "cid", "situacao": "diverge_do_informado", "evidencia": "…" }],
  "aviso_legal": "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."
}
```

`ancora` (E1, E2…) é calculada pelo PEP localizando a `evidencia` no texto da evolução (busca literal normalizada); sem match, o trecho aparece sem âncora.

### Aplicação de sugestões e feedback

1. Usuário decide um item (aceitar/editar/rejeitar). Aceitar CID/TUSS/justificativa com `aplicar: true` cria nova `GuiaRevisao` com a mudança (RFC-PEP-006) e grava `decisoes_usuario` na mesma transação, com o evento `FeedbackRegistrado` na outbox.
2. Worker `spike-feedback` agrega as decisões pendentes da análise (até 100 itens) e chama `POST /v1/analises/{analise_id_tissia_ai}/feedback` com `papel` do usuário. Sucesso (`204`) marca `feedback_enviado_em`. Retry exponencial (1 min → 6 h, 8 tentativas); `404` descarta.
3. Idempotência: a decisão é única por `(analise_id, item_id)`; reenviar o mesmo item é seguro do lado da Spike (sobrescreve). Edição do Campo 23 vai como `editado` com `texto_final`.
4. Nova revisão salva dispara nova análise (gatilho `salvar`), com o mesmo `guia_ref`: a Spike liga as revisões e mede autocorreção.

No modo `silencioso` não há decisões, logo não há feedback.

### Eventos

- Consome: `GuiaRascunhoCriada`, `GuiaRevisada`, `LoteEmPreparacao` (RFC-PEP-006).
- Publica: `AnaliseTissiaAiConcluida { guia_id, guia_revisao_id, estado, risco, score, bloqueantes }` (consumido por `faturamento` para ordenar a mesa e por `bi`), `FeedbackRegistrado`.

## UI

- **Componente único `PainelAntiGlosa`** com os estados da tela 14 (carregando, nada para analisar, erro, parcial, sucesso), usado nas telas 05 (lateral compacto), 12 (completo), 13 (revisão guiada) e 16 (coluna de risco + badge "Parcial").
- Tela 12: score com régua 0–1 e marcas 0,35/0,70 **[validar faixas com a Spike]**, "com as N correções sugeridas: X (estimado)" só se a Spike devolver estimativa (hoje não devolve → esconder), alertas agrupados por severidade, tabela DUT com evidência literal e âncora na evolução, "não citado na evolução" quando houver anexo relevante.
- Tela 13: até 3 decisões numeradas, atalhos `A`/`E`/`I` (aceitar/editar/ignorar), cronômetro de revisão (métrica RF-SPK-13), "Checagem local (sem IA)" sempre visível, "Aprovar e enviar ao faturista", "Nada é enviado à operadora a partir desta tela".
- Campo 23: `textarea` com contador `n / 500`, bloqueio de digitação e de colagem acima de 500, origem da justificativa ("montada a partir de E1, E2, E3").
- Aviso legal fixo no rodapé do painel em todos os estados que exibem IA.
- Tela 14: banner "IA clínica indisponível… Nova tentativa automática em 30 s", score "—", só alertas de regras; o faturista vê a marca "análise parcial".
- Tela 15: modo da unidade, opções do assistido, exceções por profissional, relatório com "Como calculamos".
- Acessibilidade: severidade com texto e ícone, não só cor; foco gerenciado ao aplicar sugestão.

## Segurança e LGPD

- O que sai para a Spike é exatamente a tabela de mapeamento. Teste de propriedade: para pacientes gerados com PII aleatória, o JSON serializado não contém nome, CPF, CNS, carteirinha, telefone nem e-mail do paciente.
- Texto da evolução pode conter PII digitada pelo médico; a Spike aplica scrubber (Spike ADR-0007). O PEP não tenta limpar o texto além de remover o cabeçalho do paciente.
- `vinculos_tissia_ai` cifrada, acesso só pelo módulo `tissia-ai`; RLS por tenant.
- Logs do módulo: `analise_id`, `guia_id`, gatilho, estado, latência, status HTTP. Nunca payload, evolução ou justificativa.
- Chave `tsk_test_…` (staging) ou `tsk_live_…` (produção) por ambiente no Secret Manager; rotação sem deploy (recarrega a cada 5 min).
- Base legal e papéis (clínica controladora, PEP operador, Spike suboperadora) em [[conformidade-regulatoria|conformidade.md]].

## Observabilidade

| Métrica | Alerta |
| :--- | :--- |
| `tissia_ai_latencia_ms` p50/p95 por gatilho | p95 > 5 s por 15 min |
| `tissia_ai_estado_total{estado}` | `parcial`+`falhou` > 10% em 15 min |
| `tissia_ai_breaker_aberto` | qualquer abertura em `prod` |
| `tissia_ai_422_total` | > 0 (bug de contrato) |
| `tissia_ai_aceite_ratio{tipo}` (alerta, justificativa, cid, tuss) | painel semanal; queda > 10 p.p. |
| `tissia_ai_tempo_revisao_s` p50 | meta ≤ 15 s |
| `tissia_ai_feedback_pendente` | > 1.000 ou mais velho que 24 h |

## Critérios de aceite

- [ ] Teste de contrato: DTOs gerados/validados contra `tests/snapshots/openapi_v1.json` da Spike (copiado com versão); mudança no snapshot quebra o CI do backend.
- [ ] Teste de propriedade de PII (acima) verde.
- [ ] Matriz de erros com a Spike simulada (200 ok, 200 degradado, timeout 7 s, 401, 413, 422, 500, conexão recusada): o fechamento do atendimento e o envio ao faturista nunca falham; estado da análise correto em cada caso.
- [ ] Evolução de 60.000 caracteres → corpo ≤ 64 KB, `evolucao_truncada = true`, atendimento atual preservado.
- [ ] Debounce: 5 salvamentos em 2 s → 1 chamada.
- [ ] Modo `silencioso`: chamadas acontecem e são gravadas; nenhuma rota da UI de médico/faturista devolve alertas; nenhum feedback enviado.
- [ ] Aceitar sugestão de CID cria nova revisão, grava decisão e envia feedback exatamente uma vez (com reentrega do job).
- [ ] Contador do Campo 23 bloqueia a 500 (E2E).
- [ ] E2E com Spike `staging`: guia SP/SADT de exemplo → painel completo em ≤ 6 s; revisão completa em ≤ 15 s por um usuário de teste (medido no cronômetro).
- [ ] Métricas da tabela expostas e com alertas configurados em `staging`.

## Perguntas abertas

1. A Spike aceita CID com ou sem ponto? O exemplo do manual usa `I209` e `M51.1` **[validar]**.
2. Faixas de risco (0,35 / 0,70) e "risco estimado após correções" (tela 12): a Spike pode devolver isso? Se não, remover do design.
3. Analisar guia de consulta traz valor suficiente para justificar a chamada (custo por guia)? Medir no silencioso.
4. Spike `degradado` nos diz **qual** parte falhou? Hoje não: a UI mostra genérico (pedir na SPK-01/03 se necessário).
5. O feedback aceita `motivo_rejeicao`? Hoje não; fica só no PEP.

## Notas de implementação

**Marco D (08/11/2026): fatia entregue (painel anti-glosa na tela 05).**

- Módulo `spike`: `SpikeClient` (fetch, timeout de 6 s, falha vira `null`), `SpikeService` (pseudonimização `pa_` + HMAC-SHA256 em base32 com `TISSIA_AI_SAL_PACIENTE`, payload `v1`, persistência em `analises_tissia_ai` e `decisoes_analise`) e `POST /api/v1/atendimentos/{id}/analise` (gatilho: a tela dispara 2,5 s depois de salvar a evolução ou mudar os pedidos) e `POST /api/v1/analises/{id}/decisoes` (aceito, editado, rejeitado; repassa o feedback à Spike sem bloquear).
- O corpo enviado fica gravado (`analises_tissia_ai.payload`) e aparece na aba "Enviado" do painel. Teste garante que nome, carteirinha e nascimento nunca saem; só `ref`, sexo e idade.
- Falha, timeout ou ausência de `TISSIA_AI_URL`/`TISSIA_AI_API_KEY` = análise `falhou` ("análise parcial", tela 14); `degradado` da Spike = `parcial`. Fechar o atendimento nunca depende da análise.
- **Desvios:** sem fila BullMQ, SSE nem retry; sem `vinculos_tissia_ai` (o sal é uma variável de ambiente, não KMS por clínica); uma análise por chamada, sem idempotência por revisão; sem tabelas separadas de alertas, critérios e sugestões (a resposta mapeada fica em `jsonb`); só `tipo_guia = sp_sadt`. O texto da evolução vai por seção, sem os rótulos S/O/A/P. O seed da demo põe o registro ANS 326305 (o do golden set) em todas as operadoras sintéticas.
- Rodar local: Spike (`tissia-ai`) com `uv run uvicorn app.main:app --port 8000`, um integrador (`scripts/criar_integrador.py`) e uma clínica `un_<id da unidade>` em `modo=visivel` (`scripts/configurar_clinica.py`); no `apps/api/.env`: `TISSIA_AI_URL`, `TISSIA_AI_API_KEY`.

**Circuit breaker e modo por unidade (09/11/2026).**

- Circuit breaker no `SpikeClient`: `TISSIA_AI_CB_FALHAS` (3) falhas ou timeouts seguidos (rede ou HTTP 5xx; 4xx não conta) abrem o circuito por `TISSIA_AI_CB_ABERTO_MS` (30000). Aberto, nenhuma requisição sai e a análise vira `falhou` na hora; passado o tempo, uma única chamada de prova decide (sucesso fecha, falha reabre). O estado é por processo e vale para análise, feedback e Scribe juntos (um circuito só para a Spike).
- Modo por unidade: coluna `unidades.modo_ia` (`silencioso` | `assistido`, padrão `assistido`; migração `0009_modo_ia_unidade`). `GET/PUT /api/v1/unidades/{id|atual}/modo-ia`: leitura com `organizacao:ler`, escrita com `admin:configurar`; só a unidade ativa da sessão (outra unidade = 403).
- A análise sempre roda e é gravada. No `silencioso` a API devolve a análise sem risco, alertas, sugestões e payload, sem pendência e sem risco no espelho da guia (Mesa com risco nulo); `Atendimento.modoIa` leva o modo à UI, que mostra um aviso neutro no lugar do painel. O relatório "glosa evitada" (lote sintético) não muda. Voltar a `assistido` expõe de novo as análises já gravadas (o espelho da guia só se atualiza na próxima análise).
- Desvio: a tela Modos grava ao selecionar (o design não tem botão salvar) e as opções de comportamento e as exceções por profissional seguem estáticas (P1).

**IA real na Spike (09/11/2026).** A Spike passou a usar o Gemini 3.x no Vertex AI (`IA_PROVEDOR=vertex`) para decidir os critérios DUT, extrair termos, redigir o Campo 23 e, no Scribe, transcrever e estruturar a consulta ([[ADR-IA-0010-vertex-global-e-gemini-3x|ADR-0010 da Spike]]: endpoint `global` permitido, só modelos 3.x vigentes). Cada saída é validada contra o texto: evidência precisa ser trecho literal, fato da justificativa precisa existir na evolução, e qualquer falha volta para a heurística local, sem 5xx.

- Com o Gemini a análise leva de 6 a 10 s (3 a 4 chamadas), acima do timeout padrão do PEP de 6 s. Em dev use `TISSIA_AI_TIMEOUT_MS=30000` no `.env` da API; a UI já mostra "Analisando…" e nunca trava o atendimento.
- A transcrição e a estruturação do Scribe usam `TISSIA_AI_SCRIBE_TIMEOUT_MS` (40 s por padrão).
- A decisão de residência do LLM (ADR-0010) não muda o que o PEP envia: só `ref`, sexo e idade do paciente e o texto clínico minimizado.


**Análise em duas fases (09/11/2026).** `SpikeService.analisar` faz uma 1ª chamada só com regras TISS/ANS e checklist da DUT (`opcoes.gerar_justificativa=false`, `sugerir_codigos=false`; p95 medido 2,6 s), grava e devolve a análise com `enriquecimento: "pendente"`; em segundo plano faz a chamada completa e atualiza a MESMA linha (`id` do PEP estável; `tissiaAiAnaliseId`, usado no feedback, passa ao da análise completa). Falha ou `degradado` na 2ª fase não perde nada: a linha fica com `enriquecimento: "indisponivel"` (risco e alertas valem; Reanalisar tenta de novo). O painel consulta a cada 1,5 s enquanto `pendente`. `TISSIA_AI_ANALISE_EM_DUAS_FASES=false` volta ao modo de uma chamada. **Desvios assumidos:** o espelho de risco da guia é sincronizado após a 1ª fase (a 2ª fase não altera risco nem alertas); o enriquecimento roda no processo da API (sem fila), e uma queda do processo no meio deixa `pendente` até a próxima análise; duas chamadas ao modelo por análise aumentam o custo de IA (a medir).

**Onda 4, parte 1 (10/11/2026): Modos completo, status real da IA e campos da RFC-PEP-012.** Migração 0016.

- **`analises_tissia_ai.modo`, `profissional_id` e `exibida_em`:** o modo efetivo é gravado em toda análise (exceção do profissional, senão o da unidade) e **não muda depois**: é a linha de base da RFC-PEP-012. `exibida_em` é gravado na criação, em modo assistido e com a Spike respondendo (a resposta daquela chamada é o que a tela mostra); análise `falhou` e silenciosa ficam sem ele. A migração preenche o `modo` das análises antigas com o da unidade.
- **Mudança de comportamento:** antes, voltar a `assistido` expunha as análises feitas em silêncio; agora elas continuam silenciosas e só a próxima análise já nasce visível (o teste `spike.spec.ts` foi ajustado). Um atendimento aberto durante a virada não mostra painel até a próxima análise (mudar a evolução ou os pedidos, ou Reanalisar).
- **Exceções por profissional:** tabela `tissia_ai_config_profissional`; a exceção vale sobre o modo da unidade e só aceita profissionais com vínculo médico na unidade (422 fora disso). `GET/PUT /api/v1/unidades/{id|atual}/config-ia` (leitura `organizacao:ler`, escrita `admin:configurar`); `modo-ia` continua como estava.
- **Comportamento do assistido** (`tissia_ai_config_unidade`): `exigirMotivoAoIgnorar` (o servidor rejeita `rejeitado` sem `textoFinal` com 422 e o painel pede o motivo) e `mostrarInformativos` (o servidor tira alertas `info` da resposta ao médico). `bloquearEnvioComBloqueante` é salvo, mas **inerte** até o envio ao faturista (RFC-PEP-006): a tela o mostra desabilitado, com a RFC no `title`. **Padrões** (`COMPORTAMENTO_IA_PADRAO`, migração 0017): os do canvas, ou seja, bloquear sim, exigir motivo **sim**, mostrar informativos **não**. Unidades que já operavam sem a configuração passam a ver esse padrão até o admin mudar (ainda não há dado de produção).
- **Status da IA na Topbar:** `GET /api/v1/ia/status` (`organizacao:ler`): circuito aberto = `indisponivel`; senão decide a análise mais recente dos últimos 15 minutos (`falhou`, `parcial` ou 2ª fase indisponível = `parcial`). Considera também análises silenciosas (é saúde do sistema, sem conteúdo clínico). A Topbar consulta a cada 30 s e invalida ao analisar; a página ainda pode forçar "parcial" pela análise que mostra. Sem SSE por enquanto.
- **Fora desta parte:** consumidores do outbox com uma fila por consumidor e retry do feedback (o `worker.ts` ainda publica uma fila por tipo de evento e ninguém consome; o feedback segue sem retry), SSE da análise e **catálogo TUSS vindo da Spike**, que não existe no contrato dela: proposta em [[SPK-09-catalogo-tuss|SPK-09]].
- Testes: 3 de API (exceção + modo gravado, comportamento, status com circuito), E2E `apps/e2e/specs-demo/modos.spec.ts`; o seed da demo limpa as tabelas de configuração e volta a unidade a `assistido`. Os usuários do `seed-dev` e do realm podem ter o mesmo nome no banco local, então a tela pode listar o profissional duas vezes (o E2E trata).

**Onda 4, parte 2 (11/11/2026): fila por consumidor, feedback com retry e SSE da análise.** Migração 0018.

- **Consumidores do outbox (ADR-PEP-0008):** `apps/api/src/worker-runtime.ts`. O relay lê o `outbox` (`FOR UPDATE SKIP LOCKED`) e entrega cada evento a **cada fila que o assina**, `consumidor.<nome>`, com `jobId = consumidor_evento` (o BullMQ rejeita `:`). Um `Worker` por consumidor, concorrência 5, idempotente por `(consumidor, evento.id)` (`processarUmaVez`). Retry exponencial: `WORKER_TENTATIVAS` (8) e `WORKER_BACKOFF_MS` (60 s, ou seja, 1 min → ~1 h entre as últimas tentativas; a RFC fala em até 6 h: ajustar o backoff em produção). Esgotadas as tentativas, o job vai para a fila `dlq` com `logger.error` (só consumidor, id do evento e tentativas, nunca o payload). O reprocessamento manual auditado da ADR **não** foi feito.
- **Onde roda:** em dev, dentro da API (`WORKER_EMBUTIDO`, padrão `true` fora de produção); em produção, no processo `worker` (`pnpm --filter @pep/api start:worker` / `dev:worker`, `WORKER_EMBUTIDO=false` na API). Redis ou Postgres fora do ar não derrubam a API: o worker embutido registra o erro e o outbox, que é durável, é entregue quando o relay voltar. O relay antigo (uma fila por **tipo** de evento, sem ninguém consumindo) foi substituído.
- **Consumidor `tissia-ai.feedback`:** `registrarDecisoes` grava a decisão e o evento `FeedbackRegistrado` **na mesma transação** (antes: `void fetch` sem retry). O consumidor envia à Spike as decisões ainda sem `feedback_enviado_em` da análise (até 100 itens numa chamada, `papel: medico`). A Spike fora do ar ou respondendo 5xx lança para a fila repetir; 404 descarta (marca entregue). Decisão alterada volta a pendente e é reenviada. Decidir nunca espera pela Spike.
- **Eventos publicados:** `AnaliseTissiaAiConcluida { analiseId, atendimentoId, estado, risco, enriquecimento }` na 1ª fase e ao fim da 2ª, para BI e faturamento (ainda sem consumidor: o faturamento segue sincronizando direto). Só ids e códigos.
- **SSE:** `GET /api/v1/atendimentos/{id}/analise/stream` (`prontuario:escrever`, Bearer; nunca token em query string), eventos `analise` com `{ estado }` (`completa` na 1ª fase, `concluido` ou `indisponivel` na 2ª) e `ping` a cada 25 s. A tela abre o stream enquanto a análise está `pendente` e, ao receber, busca o atendimento de novo; a consulta periódica caiu de 1,5 s para 5 s e é a rede de segurança se o stream cair. O emissor é em memória, por processo: serve enquanto a 2ª fase roda na mesma instância da API (`# ponytail`: com várias instâncias, trocar por Redis pub/sub). Não há SSE por guia nem para a revalidação em massa.
- **Fora desta parte:** o **catálogo TUSS vindo da Spike** ([[SPK-09-catalogo-tuss|SPK-09]]: a Spike não expõe listagem), o status da IA na Topbar por SSE (segue em consulta de 30 s) e o consumo de `AnaliseTissiaAiConcluida` pelo faturamento.
- Testes: 5 de API (entrega com retry e idempotência, 404, eventos só com ids, **worker real com Redis/BullMQ** e SSE com isolamento por tenant/401); o roteiro E2E exige o stream em 200 e o feedback chegou à Spike no Vertex (204). O teste com Redis pegou o `:` no `jobId`.

**Onda 4, parte 3 (11/11/2026): catálogo TUSS e CID-10 da Spike ([[SPK-09-catalogo-tuss|SPK-09]], implementado na Spike pela RFC-011).** Migração 0019.

- **Fonte única na Spike, sem réplica local:** `CatalogoService` (módulo `tissia-ai`) consulta `GET /v1/catalogos/{tuss,cid}` pelo mesmo `SpikeClient` (circuit breaker e timeout). Cache em memória por `(catálogo, termo normalizado)`, TTL de 10 min, invalidado quando `versao_bases` muda; mapa código→termo de tudo que a Spike já devolveu. `ponytail:` por processo; vira Redis só se várias instâncias repetirem muitas buscas.
- **Rotas do PEP:** `GET /api/v1/catalogos/tuss?q=&limite=` e `/cid` (`prontuario:escrever`), devolvem `{ itens, origem: 'spike' | 'reduzida' }`. Sem `tabela`: o PEP usa só a Tabela 22 (padrão da Spike).
- **Redundância (invariante):** Spike fora, sem chave ou erro → último resultado em cache (mesmo vencido) e, sem cache, `CATALOGO_TUSS_DEMO` filtrado (`origem: 'reduzida'`, aviso na tela). Para CID não há lista de contingência: o médico digita o código (formato validado por `CID_REGEX`) e tecla Enter.
- **Pedido de exame:** o código é validado contra o catálogo disponível (cache, lista reduzida ou `GET /v1/catalogos/tuss/{codigo}`, nunca a rede como bloqueio de um código já conhecido). **Desvio:** `pedidos_exame.termo` guarda o termo oficial no momento do pedido (migração 0019, nulável), para o histórico não depender de o catálogo mudar; pedidos antigos caem em `termoTuss`.
- **UI:** `BuscaCatalogo` (combobox acessível, debounce de 250 ms) substitui o `<select>` do "Procedimento" no atendimento e na revisão da escuta e o campo "Adicionar CID". O código do CID entra com ponto (`M23.2`), como o resto do prontuário. As sugestões de procedimento da escuta deixam de ser filtradas pela lista fixa (a Spike já valida).
- **Fora desta parte:** o faturamento ainda resolve o termo do procedimento pela lista fixa (`termoProcedimentoFaturamento`), caindo no código quando desconhecido; `CATALOGO_TUSS_DEMO` segue como contingência e massa de teste. A busca da Spike é lexical: "ressonância joelho" não acha a RM de articulação (o termo oficial não cita joelho); vale um sinônimo na Spike, não aqui.
- Testes: 6 de API (`apps/api/test/catalogo.spec.ts`: busca e cache, termo gravado no pedido, Spike fora, cache vencido, 404, permissão); E2E do roteiro agora escolhe CID e procedimentos pela busca e confere um código fora da lista fixa.

**Renomeação para Tissia AI (14/11/2026).** O que levava o nome da PoC passou para o nome oficial, na migração `0034_renomeia_spike_para_tissia_ai`: tabelas `analises_spike` → `analises_tissia_ai`, `spike_config_unidade` e `spike_config_profissional` → `tissia_ai_config_*`, coluna `spike_analise_id` → `tissia_ai_analise_id`, constraints e chaves primárias, evento `AnaliseSpikeConcluida` → `AnaliseTissiaAiConcluida`, origem de revisão `sugestao_spike` → `sugestao_tissia_ai`, consumidor `spike.feedback` → `tissia-ai.feedback`. A migração usa `RENAME` (nada é apagado) e atualiza as linhas existentes do outbox, de `eventos_processados` e de `guia_revisoes` (desligando o trigger append-only e o `FORCE` da RLS só dentro da transação).

- **Deploy com fila:** o nome da fila BullMQ vem do nome do consumidor (`consumidor.tissia-ai.feedback`). Jobs que estejam esperando na fila antiga (`consumidor.spike.feedback`) não serão consumidos: **esvazie a fila antiga antes de subir esta versão** (em dev, as chaves antigas foram apagadas do Redis; não havia job pendente).
- O banco do motor na infra de demonstração e na CI passou de `medscribe` para `tissia_ai`; uma demo já criada precisa de `./demo.sh down` (que remove o volume) e `./demo.sh up`, porque o banco só é criado na primeira inicialização do Postgres.


**Painel de IA do atendimento: uma vista só e sugestão proativa (fatia B da rodada de correções da guia).**

- **A aba "Enviado" saiu da interface clínica.** O corpo enviado à Spike é informação técnica de depuração, não clínica. Continua gravado em `analises_tissia_ai.payload` (trilha de auditoria) e **não sai mais pela API**: o campo `payloadEnviado` de `Analise` ficou opcional e obsoleto no contrato e a API deixou de preenchê-lo. A garantia de que só `ref`, sexo e idade do paciente saem passou a ser testada no `SpikeAdapter` (chaves enviadas) e contra o OpenAPI da `tissia-ai`.
- **Abas "Resumo", "Sugestões" e "Anti-glosa" viraram uma vista só.** "Resumo" era um texto estático de prévia (fase P2) sem dado algum: o resumo ao paciente é escrito pelo médico nos documentos. "Sugestões" e "Anti-glosa" mostravam as mesmas sugestões; hoje há duas seções na mesma vista (sugestões para o atendimento; anti-glosa com risco, alertas e checklist da DUT).
- **Tag "DUT 0":** a Spike devolve `dut: { numero: 0, conformidade: 'nao_aplicavel' }` para TUSS sem DUT na base ANS (`app/dut.py`, `app/api/guias.py`). Era o marcador de "sem DUT", exibido como se fosse a DUT número 0. Sem DUT, a UI não mostra tag; com DUT mostra `DUT <n> · Conforme | Não conforme | A confirmar | Não avaliada`.
- **Sugestão proativa durante o atendimento** (`POST /api/v1/atendimentos/:id/sugestoes-ia`, `SpikeAdapter.sugerir`): a partir da evolução **salva**, com no mínimo `MIN_CARACTERES_EVOLUCAO_IA` (80) caracteres, debounce de 2,5 s depois de salvar e nova chamada só se o texto mudou 40 caracteres ou mais (ou por "Atualizar sugestões"), a IA sugere CID e exames (etapa 1, `guias/analisar` com `procedimentos: []`) e a indicação clínica para até 3 dos exames sugeridos (etapa 2, mesma rota com os TUSS). Só no modo `assistido` (unidade ou exceção do profissional); no `silencioso` não chama a Spike. Não grava nada: o médico aceita pelas rotas de evolução, pedido e indicação. Só convênio (a Spike exige o registro ANS). Lacunas do motor em [[SPK-14-sugestao-de-codigos-e-indicacao-a-partir-da-evolucao|SPK-14]].
