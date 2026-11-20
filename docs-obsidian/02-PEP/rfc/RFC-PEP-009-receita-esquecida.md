---
projeto: PEP
tipo: rfc
status: planejada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-009-receita-esquecida.md
---

# RFC-PEP-009 — Receita esquecida

**Fase:** P2 · **Status:** Planejada (detalhada em 09/11/2026; fatias 1–3 podem começar com a P1 concluída; fatia 4 depende da Q7; fatia 5 usa a ADR-PEP-0014, já aceita) · **ADRs:** ADR-PEP-0005, ADR-PEP-0006, ADR-PEP-0008, ADR-PEP-0014, ADR-PEP-0017
**Telas:** 22 (Receita esquecida); widget no shell (02) e no painel do gestor (24)
**Requisitos:** RF-ESQ-01, RF-ESQ-02; propõe ao PRD-001 RF-ESQ-03 a RF-ESQ-05 (ver Escopo)
**Depende de:** RFC-PEP-003 (tipo e TUSS previsto do agendamento), RFC-PEP-004 (atendimento fechado, `pedidos_exame`), RFC-PEP-005 (análises gravadas, decisões, feedback, `paciente.ref`), RFC-PEP-006 (guias, revisões e máquina de estados), RFC-PEP-008 (prazo de envio, preço vigente, dias de retorno; com fallback por unidade), RFC-PEP-010 (recebível particular, fatia 1), RFC-PEP-007 (`DesfechoRegistrado`, para o "recuperado"), RFC-PEP-014 (auditoria)

## Objetivo

Todo dia, cruzar os atendimentos realizados com as guias e cobranças emitidas e mostrar o que foi feito e não cobrado, com valor em risco e prazo de envio, para que o faturista gere a guia faltante com 1 clique e revisão humana. A detecção básica é local e funciona com a Spike desligada; a Spike entra como segunda opinião (histórico de guias fora do PEP) e como fonte clínica (procedimento citado na evolução e ausente da guia).

## Escopo

**Dentro (P2)**

- [RF-ESQ-01] Rotina diária por unidade (06:00, fuso da agenda) e execução manual ("Executar agora"), cruzando atendimentos fechados × guias, com valor em risco e prazo de envio. Consulta `POST /v1/guias/esquecidas` da Spike quando o modo da unidade não é `desligado`.
- [RF-ESQ-02] Gerar a guia faltante a partir da pendência com 1 clique (guia `rascunho` que segue a análise normal da RFC-PEP-005 e a máquina da RFC-PEP-006), ou incluir o procedimento na guia existente ainda não enviada.
- [RF-ESQ-03] *(proposto)* Pendências clínicas evolução × guia: procedimento citado na evolução como realizado na unidade e ausente da guia, a partir de `sugestoes_codigos.procedimentos[]` com `situacao: "ausente_na_guia"` das análises já gravadas, com o trecho como evidência e aceite humano.
- [RF-ESQ-04] *(proposto)* Regras locais sem Spike: atendimento particular sem recebimento (recibo) e retorno sem cobrança (retorno fora da janela de retorno do contrato).
- [RF-ESQ-05] *(proposto)* "Não cobrar" com motivo fechado, registrado na trilha de auditoria e devolvido à Spike como feedback quando a pendência veio dela; KPI "recuperado este mês".
- Filtros por tipo, profissional e operadora; ordenação por prazo de envio.

**Fora**

- Materiais (Tabela 19), medicamentos e taxas: dependem da Q1 e de a Spike sugerir esses códigos (hoje a base de sugestões é só de procedimentos). A linha "Material não lançado" do design fica como prévia.
- Silenciar automaticamente um TUSS por profissional depois de N recusas ("aprendizado"): na P2 só a taxa de "não cobrar" por profissional × TUSS aparece no resumo; silenciar fica para depois de medir o ruído no piloto.
- Cobrança de falta (no-show) particular: RFC-PEP-003 Q2.

## Design

### Tipos de pendência

| `tipo` | Detecção | Origem | Ação principal |
| :--- | :--- | :--- | :--- |
| `sem_guia` | Atendimento de convênio fechado sem guia ativa (status ≠ `cancelada`) | `regra_local`, anotada por `tissia_ai_esquecidas`; ou `tissia_ai_sugestao` quando o TUSS veio da evolução | Gerar guia |
| `guia_so_consulta` | Guia existe só com consulta e a evolução cita procedimento realizado | `tissia_ai_sugestao` | Incluir na guia (SP/SADT) ou gerar guia SP/SADT complementar (guia de consulta) |
| `procedimento_nao_lancado` | Guia SP/SADT existe e não tem o procedimento citado | `tissia_ai_sugestao` | Incluir na guia |
| `particular_sem_recibo` | Atendimento particular fechado com recebível em aberto (RFC-PEP-010) | `regra_local` | Emitir recibo (tela 23) |
| `retorno_sem_cobranca` | Agendamento `retorno` de convênio, sem guia, com intervalo desde a consulta anterior (mesmo paciente × profissional) acima de `dias_retorno` do contrato, ou sem regra cadastrada | `regra_local` | Confirmar: gerar guia de consulta ou não cobrar (`retorno_no_prazo`) |

Retorno de convênio sem guia **dentro** da janela não é pendência. Atendimento de convênio sem guia cujo agendamento é `retorno` vira `retorno_sem_cobranca`, nunca `sem_guia` (uma pendência por atendimento × tipo × TUSS).

> Hoje (Marco D) a guia só nasce quando há pedido de exame de convênio (RFC-PEP-004, notas). Até a RFC-PEP-006 criar a guia de consulta a partir de todo atendimento de convênio (RF-FAT-01), toda consulta de convênio sem pedido aparece como `sem_guia`. É o comportamento correto: essa consulta de fato não foi cobrada.

### Modelo de dados (módulo `faturamento`, submódulo `esquecidas`)

```sql
-- todas as tabelas: tenant_id uuid NOT NULL, RLS forçada por convenção em drizzle/rls (ADR-PEP-0019);
-- ids externos (atendimento, paciente, profissional, operadora, análise) sem FK entre módulos (ADR-PEP-0017)

CREATE TABLE config_receita_esquecida (
  unidade_id                uuid PRIMARY KEY,
  tenant_id                 uuid NOT NULL,
  ativo                     boolean NOT NULL DEFAULT true,     -- além da flag `modulos.esquecidas` (release plan §5)
  hora_execucao             time NOT NULL DEFAULT '06:00',
  janela_dias               smallint NOT NULL DEFAULT 30 CHECK (janela_dias BETWEEN 1 AND 90),
  tuss_executaveis          text[] NOT NULL DEFAULT '{}',     -- TUSS realizados na unidade (consultório); fonte clínica só considera estes (Q7)
  tuss_consulta             text,                            -- TUSS da consulta quando o agendamento não traz procedimento; sem valor = pendência sem TUSS
  prazo_envio_padrao_dias   smallint,                        -- fallback sem contrato (RFC-PEP-008); NULL = "sem prazo cadastrado" (Q4)
  dias_retorno_padrao       smallint,                        -- fallback sem contrato; NULL = todo retorno sem guia pede confirmação (Q3)
  max_analises_por_execucao smallint NOT NULL DEFAULT 200,   -- teto de chamadas `analisar` para atendimentos sem análise (custo de IA)
  atualizado_por            uuid NOT NULL,
  atualizado_em             timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE execucoes_receita_esquecida (
  id                      uuid PRIMARY KEY,
  tenant_id               uuid NOT NULL,
  unidade_id              uuid NOT NULL,
  gatilho                 text NOT NULL CHECK (gatilho IN ('agendado','manual')),
  solicitado_por          uuid,                              -- NULL no agendado
  janela_inicio           date NOT NULL,
  janela_fim              date NOT NULL,
  status                  text NOT NULL CHECK (status IN ('rodando','concluida','falhou')),
  tissia_ai_estado            text NOT NULL CHECK (tissia_ai_estado IN ('ok','indisponivel','desligado')),
  atendimentos_verificados integer, com_cobranca_completa integer,
  pendencias_novas        integer, pendencias_abertas integer, pendencias_resolvidas integer,
  valor_em_risco_centavos bigint,
  analises_solicitadas    integer,                           -- chamadas `analisar` feitas por esta execução
  erro                    text,                              -- código curto, nunca corpo de resposta
  iniciada_em             timestamptz NOT NULL DEFAULT now(),
  concluida_em            timestamptz
);
CREATE UNIQUE INDEX ux_exec_esq_rodando ON execucoes_receita_esquecida (unidade_id) WHERE status = 'rodando';
CREATE INDEX ix_exec_esq_unidade ON execucoes_receita_esquecida (unidade_id, iniciada_em DESC);

CREATE TABLE pendencias_cobranca (
  id                      uuid PRIMARY KEY,
  tenant_id               uuid NOT NULL,
  unidade_id              uuid NOT NULL,
  atendimento_id          uuid NOT NULL,
  paciente_id             uuid NOT NULL,
  profissional_id         uuid NOT NULL,
  operadora_id            uuid,                              -- NULL em particular
  data_atendimento        date NOT NULL,
  tipo                    text NOT NULL CHECK (tipo IN ('sem_guia','guia_so_consulta','procedimento_nao_lancado','particular_sem_recibo','retorno_sem_cobranca')),
  origem                  text NOT NULL CHECK (origem IN ('regra_local','tissia_ai_esquecidas','tissia_ai_sugestao')),
  codigo_tuss             text,                              -- NULL quando não há TUSS (particular, consulta sem tuss_consulta)
  descricao               text NOT NULL,                     -- termo TUSS ou rótulo do tipo ("Consulta realizada"); nunca texto da evolução
  historico_tissia_ai         text CHECK (historico_tissia_ai IN ('sem_registro','possivel_registro')),  -- anotação de /v1/guias/esquecidas
  analise_spike_id        uuid,                              -- análise de onde veio a sugestão (tissia_ai_sugestao)
  sugestao_item_id        text,                              -- "sug_tuss:<codigo>" (convenção do feedback, RFC-PEP-005)
  guia_id                 uuid,                              -- guia existente (incluir) ou guia gerada a partir da pendência
  guia_id_reservado       uuid,                              -- id que a guia gerada usará (mesmo guia_ref de uma análise feita sem guia)
  valor_estimado_centavos bigint CHECK (valor_estimado_centavos >= 0),  -- NULL = sem preço vigente
  prazo_envio             date,                              -- NULL em particular ou sem prazo cadastrado
  status                  text NOT NULL DEFAULT 'aberta'
                          CHECK (status IN ('aberta','guia_gerada','incluida','recibo_emitido','nao_cobrar','resolvida')),
  motivo_nao_cobrar       text CHECK (motivo_nao_cobrar IN ('nao_realizado','retorno_no_prazo','cortesia','incluso_em_pacote','cobrado_fora_do_pep','duplicidade','outro')),
  observacao              text CHECK (char_length(observacao) <= 280),   -- obrigatória com 'outro'; sem dado clínico (aviso na UI)
  decidido_por            uuid, decidido_em timestamptz,     -- NULL em 'resolvida' (resolução pelo sistema)
  primeira_execucao_id    uuid NOT NULL REFERENCES execucoes_receita_esquecida(id),
  ultima_execucao_id      uuid NOT NULL REFERENCES execucoes_receita_esquecida(id),
  criado_em               timestamptz NOT NULL DEFAULT now(),
  atualizado_em           timestamptz NOT NULL DEFAULT now(),
  CHECK ((status = 'nao_cobrar') = (motivo_nao_cobrar IS NOT NULL)),
  CHECK (motivo_nao_cobrar IS DISTINCT FROM 'outro' OR observacao IS NOT NULL)
);
-- uma pendência por atendimento × tipo × TUSS, em qualquer status: decidida não reabre
CREATE UNIQUE INDEX ux_pendencia ON pendencias_cobranca (atendimento_id, tipo, coalesce(codigo_tuss, ''));
CREATE INDEX ix_pendencia_lista ON pendencias_cobranca (unidade_id, status, prazo_envio NULLS LAST);
```

`pendencias_cobranca` é mutável só em `status` e nos campos de decisão: a transição sai de `aberta` uma única vez (trigger rejeita mudança a partir de status final), e toda decisão grava `auditoria_eventos` (RFC-PEP-014). Valores em centavos inteiros (`bigint`, como na RFC-PEP-007; `guias.valor_total_centavos` é `integer` hoje); o modelo de dados geral ainda fala em `numeric(12,2)` e precisa ser alinhado.

`# ponytail:` o trecho da evolução não é copiado para a pendência; o detalhe lê a `evidencia` da análise gravada (RFC-PEP-005). Um lugar só para dado clínico, uma regra de acesso só.

### Máquina de estados da pendência

```
aberta ──(gerar guia)──────────▶ guia_gerada
aberta ──(incluir na guia)─────▶ incluida
aberta ──(recebimento particular, evento RecebimentoRegistrado)──▶ recibo_emitido
aberta ──(não cobrar + motivo)─▶ nao_cobrar
aberta ──(execução seguinte não acha mais a falta: guia criada por outro caminho, atendimento cancelado)──▶ resolvida
```

Todos os estados além de `aberta` são finais. Desfazer "não cobrar" não existe: se foi engano, o faturista cria a guia pela mesa (RFC-PEP-006) e a pendência continua registrada como decidida.

### Execução

Job BullMQ `esquecidas.executar` com chave `unidade_id` (um por unidade; o índice `ux_exec_esq_rodando` impede dois ao mesmo tempo). Um job repetível (`esquecidas.agendador`, a cada 15 min) lista, via `DbAdmin` auditado (só `tenant_id`, `unidade_id`, `hora_execucao`; nenhum dado de domínio), as unidades com `modulos.esquecidas` ligado e `hora_execucao` vencida no dia, e enfileira uma execução por unidade. Todo o resto roda em `Db.comTenant` com o tenant e a unidade do job.

1. **Coleta local.** Atendimentos `fechado` da unidade com data em `[hoje − janela_dias, agora]`, com tipo e TUSS previsto do agendamento (RFC-PEP-003), guias ativas por atendimento (RFC-PEP-006), pedidos de exame (RFC-PEP-004) e, para particular, o recebível (RFC-PEP-010, só com `modulos.financeiro`).
2. **Regras locais** geram os candidatos `sem_guia`, `retorno_sem_cobranca` e `particular_sem_recibo` (tabela acima). Valor = preço vigente na data do atendimento (RFC-PEP-008; antes dela, sem valor); prazo = data do atendimento + prazo de envio do contrato, senão `prazo_envio_padrao_dias`.
3. **Spike — histórico** (modo ≠ `desligado`, ver abaixo): os candidatos de convênio com TUSS vão para `POST /v1/guias/esquecidas`, só eles, com `guia_ref: null`.
4. **Spike — fonte clínica** (só modo `assistido`): para cada atendimento de convênio da janela, lê a última análise gravada; procedimentos com `situacao: "ausente_na_guia"` viram candidatos se (a) o TUSS está em `tuss_executaveis`, (b) o TUSS **não** está em `pedidos_exame` do atendimento (pedido = solicitado, não realizado) e (c) o TUSS não está na guia atual. Atendimento de convênio sem análise chama `POST /v1/guias/analisar` uma vez (até `max_analises_por_execucao`), com `guia_ref = "gu_" + guia_id_reservado` e `procedimentos: []` quando não há guia.
5. **Mescla e upsert** pela chave `(atendimento_id, tipo, codigo_tuss)`: candidato novo → `aberta`; pendência `aberta` existente → atualiza valor, prazo, `historico_tissia_ai` e `ultima_execucao_id`; pendência decidida → intocada; pendência `aberta` que não voltou como candidata → `resolvida`.
6. **Fecho:** totais na execução, `status = concluida`, `ExecucaoReceitaEsquecidaConcluida` no outbox. Falha não tratada → `falhou` com código de erro; pendências já gravadas continuam válidas.

Meta: unidade com 3.000 atendimentos na janela em ≤ 60 s sem chamadas `analisar` (as chamadas, quando houver, rodam com concorrência 4 e não seguram o upsert dos outros tipos).

### Spike

**`POST /v1/guias/esquecidas`** (mapeamento em [[mapeamento-pep-tissia-ia|spike/README §3.7]]), pelo SpikeAdapter, timeout 30 s, até 2.000 itens por chamada (paginado), 2 novas tentativas:

```json
{
  "clinica_ref": "un_0193a1…",
  "atendimentos": [
    { "atendimento_ref": "at_0193b7…", "paciente_ref": "pa_K7QX…", "data_atendimento": "2026-10-09",
      "registro_ans": "<REGISTRO_ANS>", "codigo_tuss": "<TUSS_…>", "guia_ref": null, "valor_estimado": 190.00 }
  ]
}
```

- `valor_estimado` vai em reais (`centavos / 100`), como o contrato da Spike pede; a resposta (`"180.00"`) volta para centavos no adapter.
- Só candidatos sem guia vão, com `guia_ref: null`. Motivo: a Spike considera "conhecida" só a guia que ela analisou ou recebeu em lote; uma guia válida do PEP em modo `desligado` voltaria como esquecida. Mandar só os candidatos também minimiza dado (LGPD).
- Atendimento que a Spike **não** devolve está, para ela, no histórico (`historico_guias_tiss`, lotes ingeridos pela RFC-PEP-007, inclusive de antes do PEP). A Spike casa só por data × registro ANS × TUSS, sem paciente: num dia com várias consultas iguais na mesma operadora, uma guia do histórico "cobre" todas. Por isso a resposta **anota, não remove**: devolvido → `historico_tissia_ai = 'sem_registro'` e `origem = 'tissia_ai_esquecidas'`; não devolvido → `'possivel_registro'` e selo "Pode ter sido faturado fora do PEP — conferir". Proposta de SPK para casar por paciente (Perguntas abertas).
- Particular nunca vai à Spike. Candidato sem TUSS não vai (o campo é obrigatório no contrato).
- Timeout, 5xx, 422 ou circuito aberto → `tissia_ai_estado = 'indisponivel'`; os candidatos ficam com `origem = 'regra_local'` e `historico_tissia_ai = NULL`. A lista aparece igual, com o aviso "Cruzamento com o histórico da Spike indisponível nesta execução". Nunca bloqueia.

**Fonte clínica e modos (ADR-PEP-0006):** `desligado` → nenhuma chamada, só regras locais. `silencioso` → chama `esquecidas` (cruzamento determinístico, não é conteúdo de IA), mas não cria pendência `tissia_ai_sugestao` nem chama `analisar` por conta própria: as análises seguem gravadas e a RFC-PEP-012 pode medir depois quanto teria sido recuperado. `assistido` → tudo.

### API (módulo `faturamento`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/receita-esquecida/resumo` | KPIs (atendimentos verificados, com cobrança completa, pendências abertas, valor em risco, vencem em 7 dias), última execução e "recuperado este mês" |
| `GET` | `/api/v1/receita-esquecida/pendencias?tipo=&profissional=&operadora=&status=aberta&cursor=` | Lista ordenada por `prazo_envio ASC NULLS LAST`, depois valor desc (cursor) |
| `GET` | `/api/v1/receita-esquecida/pendencias/{id}` | Painel lateral: dados do atendimento, guia proposta (tipo, TUSS, quantidade, valor), exigência de autorização (RFC-PEP-008), risco da análise de origem, trecho da evolução (só com permissão) e aviso legal |
| `POST` | `/api/v1/receita-esquecida/pendencias/{id}/gerar-guia` | `{ "codigo_tuss": "<TUSS_…>", "quantidade": 1 }` (ambos opcionais: default = proposta) → `201 { "guia_id": "…" }` |
| `POST` | `/api/v1/receita-esquecida/pendencias/{id}/incluir-na-guia` | `{ "guia_id": "…", "base": 3, "quantidade": 1 }` → nova revisão (RFC-PEP-006); `409 guia_enviada` se a guia já foi enviada; `409` se `base` desatualizada |
| `POST` | `/api/v1/receita-esquecida/pendencias/{id}/nao-cobrar` | `{ "motivo": "nao_realizado", "observacao": "…" }` (`observacao` obrigatória com `outro`) |
| `POST` | `/api/v1/receita-esquecida/execucoes` | "Executar agora" → `202 { "execucao_id": "…" }`; `409` com o id da execução em andamento |
| `GET` | `/api/v1/receita-esquecida/execucoes/{id}` | Status e totais (a UI consulta a cada 2 s enquanto `rodando`) |
| `GET/PUT` | `/api/v1/unidades/atual/receita-esquecida/config` | Configuração da unidade (`admin:configurar`) |

Ação sobre pendência fora de `aberta` → `409 pendencia_ja_decidida`. Contratos Zod estritos em `@pep/contracts` (`esquecidas.ts`), erros RFC 9457.

Permissões (RFC-PEP-001, sem permissão nova): leitura com `guias:ler`; ações e "Executar agora" com `guias:escrever` (faturista, médico) — "Executar agora" também com `financeiro:escrever` (gestor); trecho da evolução só com `prontuario:ler` ou `prontuario:ler_trecho_guia`. A recepção já tem `guias:ler` e vê as pendências `particular_sem_recibo` para registrar o recebimento (RFC-PEP-010).

### Ações

- **Gerar guia:** na mesma transação, cria a guia em `rascunho` pelo serviço público do faturamento (RFC-PEP-006) com `id = guia_id_reservado`, tipo `sp_sadt` (procedimento) ou `consulta` (consulta/retorno), beneficiário do atendimento, TUSS e quantidade, data de execução = data do atendimento, CIDs da evolução e **Campo 23 vazio**; revisão com `origem = 'sistema'` e `dados.pendencia_id`. Publica `GuiaRascunhoCriada` (a análise anti-glosa normal sugere o Campo 23, que exige aceite próprio). Se a origem é `tissia_ai_sugestao`, grava a decisão `aceito` do item `sug_tuss:<codigo>` na análise de origem (`FeedbackRegistrado` → feedback à Spike, RFC-PEP-005). A pendência vai a `guia_gerada` com `guia_id`.
- **Editar:** mesma rota com `codigo_tuss`/`quantidade` alterados (decisão `editado`), e a UI abre o editor da guia (tela 17).
- **Incluir na guia:** nova revisão da guia existente com o procedimento (`origem = 'sugestao_tissia_ai'`), que volta a `em_revisao` e sai de lote não enviado (regra da RFC-PEP-006). Guia `enviada` não muda: a UI oferece "Gerar guia complementar" (Q6). Guia de consulta não recebe procedimento SP/SADT: `guia_so_consulta` sobre guia de consulta mostra "Gerar guia SP/SADT" (Q5).
- **Não cobrar:** grava motivo; se a origem é `tissia_ai_sugestao`, decisão `rejeitado` no item da análise (feedback à Spike). `nao_realizado` é o sinal de falso positivo "solicitado ≠ realizado".
- **Emitir recibo:** a UI leva à tela 23 com o atendimento selecionado; o consumidor de `RecebimentoRegistrado` (RFC-PEP-010) muda a pendência para `recibo_emitido`.

### Eventos

- **Publica:** `ExecucaoReceitaEsquecidaConcluida { execucao_id, unidade_id, pendencias_novas, valor_em_risco_centavos }` (consumido por `notificacoes` e `bi`); `PendenciaCobrancaDecidida { pendencia_id, status, guia_id }` (consumido por `bi`, projeção `fato_recuperacao` da RFC-PEP-012). Também dispara os já existentes `GuiaRascunhoCriada`, `GuiaRevisada` e `FeedbackRegistrado` pelas ações. Os dois novos entram na lista de auxiliares do README.
- **Consome:** `RecebimentoRegistrado` (RFC-PEP-010, origem particular) → `recibo_emitido`. A detecção não consome eventos: é um lote diário sobre o estado atual.
- Payload só com ids e números (ADR-PEP-0008).

### Regras

- **Recuperado este mês:** soma, nas guias com `guia_id` vindo de pendência `guia_gerada` ou `incluida`, do valor pago (`DesfechoRegistrado` de status `pago` ou `glosado_parcial`, RFC-PEP-007) com pagamento no mês; em `incluida`, só o valor do item incluído. Calculado na leitura, sem tabela própria. Sem a RFC-PEP-007, o cartão mostra "Cobrado a partir da lista" (valor das guias geradas) com o selo "Prévia".
- **Com cobrança completa** = atendimentos verificados sem pendência `aberta`.
- **Notificação** (Q2, padrão até decidir): fim de execução com pendência nova de prazo ≤ 7 dias → aviso no shell (tela 02) para faturistas e gestor da unidade. Nunca por WhatsApp ou e-mail com dado de paciente.
- **Prazo vencido** continua na lista, no topo (ordenação por prazo), com selo "Prazo vencido"; não some sozinho.
- Nada sugerido pela Spike entra em guia sem o clique de uma pessoa com `guias:escrever`; a guia gerada ainda passa pela revisão da RFC-PEP-006 (médico aprova quando a unidade exige).

## UI (tela 22, rota `/faturamento/receita-esquecida`)

Hoje estática em `apps/ui/src/modulos/faturamento/ReceitaEsquecida.tsx` (rota `/receita`); migrar para a rota canônica da RFC-PEP-015, filtros em `nuqs` (`?tipo=&profissional=&operadora=`), dados com TanStack Query.

| Bloco do design | Fonte | Observação |
| :--- | :--- | :--- |
| Cabeçalho "última execução hoje 06:00" + "Executar agora" | `resumo.ultima_execucao`, `POST /execucoes` | Botão vira "Executando…" com consulta ao status; aviso se `tissia_ai_estado = indisponivel` |
| KPIs: atendimentos (30 dias), com guia completa, pendências, valor em risco, "N vencem em 7 dias" | `GET /resumo` | "Com guia completa" passa a "Com cobrança completa" (inclui particular) |
| Chips Tipo / Profissional / Operadora; "Ordenado por prazo de envio" | `GET /pendencias` | Opção "Material não lançado" do design não aparece (Fora) |
| Tabela: atendimento (data · id curto `P-XXXX`), profissional, "Encontrado na evolução", pendência, valor, prazo, ação | `GET /pendencias` | "Encontrado na evolução" = `descricao` (termo TUSS), nunca a frase da evolução; selo "Pode ter sido faturado fora do PEP" quando `possivel_registro` |
| Rodapé "Mostrando N de M… saem da lista e ficam na trilha" | cursor + total | — |
| Painel "Gerar guia": cabeçalho, atendimento/operadora/elegibilidade, trecho da evolução assinada, guia proposta, autorização, risco | `GET /pendencias/{id}` | Trecho só com permissão (senão "Trecho disponível ao médico e ao faturista"); autorização da RFC-PEP-008 (antes dela: "Autorização não verificada", prévia); risco = análise de origem, rotulado "risco da análise do atendimento" |
| Aceitar e criar guia / Editar / Não cobrar | `gerar-guia`, editor, `nao-cobrar` | "Não cobrar" abre diálogo com motivos; atalhos `A`/`E`/`N` |
| Aviso legal | constante | Fixo sempre que o painel mostra item de origem `tissia_ai_sugestao` |
| "Recuperado este mês" | `resumo.recuperado_mes` | "Prévia" até a RFC-PEP-007 (ver Regras) |

Widget do shell (02) e cartão do painel do gestor (24) usam `GET /resumo` (valor em risco, vencem em 7 dias, recuperado). O selo "Prévia" do item no menu sai quando a fatia 1 entrar.

## Segurança e LGPD

- Para a Spike vão só `clinica_ref`, `atendimento_ref`, `paciente.ref`, data, registro ANS, TUSS, `guia_ref` e valor (`esquecidas`), e o corpo normal de `analisar` (RFC-PEP-005). Nada de nome, CPF, carteirinha ou data de nascimento.
- A lista mostra o id curto do paciente, nunca o nome. O trecho da evolução aparece só no detalhe, só com permissão de prontuário e só a frase de evidência; abrir o detalhe com trecho grava `prontuario.trecho_visualizado` na auditoria.
- `descricao` e `observacao` não levam texto clínico (a UI avisa no campo); a `observacao` é mascarada nos logs como qualquer corpo.
- Logs da execução: ids, contagens, latência e `tissia_ai_estado`; nunca o corpo enviado ou recebido.
- Toda decisão (`gerar-guia`, `incluir-na-guia`, `nao-cobrar`) e todo "Executar agora" grava `auditoria_eventos` com ator, pendência e motivo.

## Critérios de aceite

- [ ] RLS: usuário do tenant A não lista nem decide pendência do tenant B (teste com dois tenants).
- [ ] Com a Spike desligada (modo `desligado`) ou fora do ar (mock com timeout), a execução conclui, gera as pendências locais e marca `tissia_ai_estado` corretamente; nenhum atendimento ou faturamento é bloqueado.
- [ ] O corpo enviado a `/v1/guias/esquecidas` só tem candidatos sem guia, com `guia_ref: null`, e passa no teste de contrato contra o snapshot OpenAPI da Spike (sem campo extra, sem PII).
- [ ] Atendimento não devolvido pela Spike continua na lista com `possivel_registro` (não é removido).
- [ ] Sugestão `ausente_na_guia` de TUSS que está em `pedidos_exame` do atendimento ou fora de `tuss_executaveis` não vira pendência; no modo `silencioso` nenhuma pendência `tissia_ai_sugestao` é criada.
- [ ] Duas execuções seguidas não duplicam pendências; pendência `nao_cobrar` não reabre; pendência cuja guia foi criada pela mesa vira `resolvida`.
- [ ] Duas execuções simultâneas da mesma unidade → a segunda recebe `409`.
- [ ] "Gerar guia" cria exatamente uma guia `rascunho` com Campo 23 vazio, publica `GuiaRascunhoCriada` e, se a origem é a Spike, grava a decisão e envia o feedback uma vez (com reentrega do job).
- [ ] "Incluir na guia" em guia `enviada` → `409 guia_enviada`; em guia `em_lote` → nova revisão e a guia sai do lote.
- [ ] `nao-cobrar` com `outro` sem observação → `422`; toda decisão aparece na trilha de auditoria.
- [ ] Unidade com 3.000 atendimentos na janela: execução sem chamadas `analisar` em ≤ 60 s; lista de pendências em p95 ≤ 300 ms.
- [ ] Tela 22 ligada à API na rota `/faturamento/receita-esquecida`, com estados de carregando, vazio ("Nenhuma pendência nos últimos 30 dias") e erro.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | Materiais (Tabela 19) entram na P2 ou só procedimentos? **[validar]** | Consultor de faturamento | Nada (fora até decidir) |
| Q2 | Quem recebe a notificação diária: faturista, gestor ou o médico do atendimento? (padrão adotado: faturistas + gestor, só no shell) | Produto | Nada |
| Q3 | Onde fica a janela de retorno sem cobrança (dias) por operadora? Proposta: campo do contrato na RFC-PEP-008, com `dias_retorno_padrao` da unidade como fallback **[validar regra usual]** | RFC-PEP-008 | Fatia 5 |
| Q4 | Prazo de envio padrão quando não há contrato cadastrado: deixar sem prazo ou usar um valor da unidade? **[validar]** | Consultor de faturamento | Nada (default: sem prazo) |
| Q5 | Procedimento de consultório no mesmo dia da consulta: vai na mesma SP/SADT (consulta + procedimento) ou em guia SP/SADT separada da guia de consulta? **[validar por operadora]** | Consultor de faturamento | Fatia 2 (só o rótulo da ação) |
| Q6 | Guia complementar para atendimento cuja guia já foi enviada é aceita pelas operadoras do piloto, ou o caminho é recurso/reapresentação? **[validar]** | Consultor de faturamento | Fatia 2 (ação "Gerar guia complementar") |
| Q7 | A regra "só procedimentos de consultório da lista `tuss_executaveis` e fora dos pedidos de exame" basta para separar realizado de solicitado? Quem cura a lista por unidade? **[validar com CMO]** | CMO + gestor piloto | Fatia 4 |
| Q8 | Proposta de SPK a abrir: `sugestoes_codigos.procedimentos[]` com `natureza` (`realizado` \| `solicitado` \| `planejado`), para não depender só da Q7 | Pedro | Nada (melhora a fatia 4) |
| Q9 | Proposta de SPK a abrir: `/v1/guias/esquecidas` casar também por `paciente_ref` (ou número da guia no prestador) no histórico; hoje data × ANS × TUSS faz uma guia cobrir várias consultas do mesmo dia | Pedro | Nada (o PEP só anota) |
| Q10 | Se Q1 = sim: proposta de SPK a abrir para sugestão de materiais (Tabela 19) na análise | Pedro | — |

## Fatias de entrega

1. **Detecção local e lista** — `config_receita_esquecida`, `execucoes_receita_esquecida`, `pendencias_cobranca`; execução manual; `sem_guia` e `retorno_sem_cobranca` com `dias_retorno_padrao`; `GET /resumo`, `GET /pendencias`, `nao-cobrar`; tela 22 com KPIs, filtros e tabela (painel sem trecho). Funciona com a Spike desligada. Depende da P1 (RFC-PEP-003/004/006).
2. **Gerar guia e incluir na guia** — `gerar-guia`, `incluir-na-guia`, painel lateral com guia proposta. Depende de `guia_revisoes` e da máquina de estados completa da RFC-PEP-006 (hoje desvio do Marco D).
3. **Rotina diária e histórico da Spike** — agendador 06:00, `esquecidas.executar` com `/v1/guias/esquecidas` (anotação `historico_tissia_ai`), notificação no shell, `ExecucaoReceitaEsquecidaConcluida`. Depende do SpikeAdapter com fila (RFC-PEP-005).
4. **Fonte clínica** — pendências `tissia_ai_sugestao` (`guia_so_consulta`, `procedimento_nao_lancado`, `sem_guia` com TUSS da evolução), análise de atendimentos sem análise com `guia_id_reservado`, trecho no painel, decisões e feedback. Depende da Q7 e das tabelas de análise da RFC-PEP-005.
5. **Particular e retorno pelo contrato** — `particular_sem_recibo` (consome `RecebimentoRegistrado`) e janela de retorno, prazo e preço vindos do contrato. Depende da RFC-PEP-010 fatia 1 (ADR-PEP-0014) e da RFC-PEP-008.
6. **Recuperado e widgets** — "Recuperado este mês", `PendenciaCobrancaDecidida`, widget do shell (02) e cartão do painel do gestor (24). Depende da RFC-PEP-007.

## Notas de alinhamento (20/11/2026)

- **Alinhamento pós-`tissia#31`, `feat/guia-correcoes-int` e `tissia-workspace#2`:**
  - Com a `ADR-PEP-0029` (consulta sempre cobrada em SP/SADT única ou Guia de Consulta separada), a nota em *Tipos de pendência* sobre "consulta de convênio sem pedido não gera guia" deixou de ocorrer para consultas novas, mantendo-se o cruzamento para atendimentos legados, procedimentos de consultório não lançados (`guia_so_consulta`) e itens sem preço no contrato (`itens_sem_preco`).
  - A [[ADR-PEP-0030-separacao-pedido-externo-execucao-local-e-retorno|ADR-PEP-0030]] e a [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023]] endereçam diretamente as perguntas abertas **Q3, Q5, Q7 e Q8** desta RFC: pedidos marcados como `solicitacao_externa` não entram na guia faturável da clínica, e retornos fechados dentro de `dias_retorno` são reconhecidos como regulares sem gerar falso positivo `sem_guia`.
