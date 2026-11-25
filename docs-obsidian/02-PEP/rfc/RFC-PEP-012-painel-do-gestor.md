---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-012-painel-do-gestor.md
---

# RFC-PEP-012 — Painel do gestor (BI)

**Fase:** P2 · **Status:** Implementada (fatia do Marco D sobre lote sintético e fatia 1 — projeções por evento e Painel do Gestor — entregues em 15/11/2026; fatias 2–3 planejadas, ver Notas de implementação) · **ADRs:** ADR-PEP-0006, ADR-PEP-0008, ADR-PEP-0003, ADR-PEP-0017
**Telas:** 24 (Painel do gestor); 15 (relatório "glosas que teriam sido evitadas", bloco do relatório)
**Requisitos:** RF-BI-01, RF-BI-02, RF-BI-03 (RF-BI-04, BI avançado, é P3)
**Depende de:** RFC-PEP-005 (análises com `modo` e decisões, P1), RFC-PEP-006 (guias, revisões, valores, `LoteEnviado`, P1), RFC-PEP-007 (`DesfechoRegistrado` com código da Tabela 38 por item), RFC-PEP-009 (guia marcada com origem "receita esquecida"), RFC-PEP-014 (perfil Gestor, auditoria); consultor de faturamento para o mapeamento alerta → Tabela 38 (Q2). A RFC-PEP-010 **não** é dependência: os valores vêm do desfecho, não do caixa.

## Objetivo

Provar valor em reais: glosa evitada no modo assistido contra a linha de base do modo silencioso, receita esquecida recuperada, aceite das sugestões da IA e produtividade por profissional. É o argumento de venda do PEP e da Spike. Cada número tem definição fixa, mostrada na tela, e é reproduzível a partir dos dados do PEP.

## Escopo

**Dentro (P2)**

- [RF-BI-01] Taxa de glosa por operadora, por código TUSS (ranking por valor glosado) e por médico.
- [RF-BI-02] Glosa evitada (R$) no modo assistido; taxa de glosa atual × linha de base do silencioso; evolução mensal com o marco "Assistido desde"; relatório "glosas que teriam sido evitadas" do período silencioso (telas 15 e 24, ADR-PEP-0006).
- [RF-BI-03] Receita esquecida recuperada; aceite das sugestões da IA (aceito / editado / ignorado); tempo médio de revisão por guia; atendimentos por dia por médico.
- Filtro por período (competências) e unidade (ativa ou todas as do gestor); exportação em PDF pela impressão do navegador.
- Projeções (`bi_*`) alimentadas por eventos e recalculáveis do zero.

**Fora**

- BI self-service, coortes, previsão de receita, armazém colunar, exportação para BI externo: P3 (RF-BI-04, RFC-PEP-016); API somente leitura e webhooks: RFC-PEP-014.
- Painel do médico com os próprios números (Q5).
- Indicadores de caixa, recebido e repasse: tela 23, RFC-PEP-010.
- Atribuição causal: o painel mostra associação por período e operadora, nunca "a IA causou".

## Design

### Princípio: recalcular a guia, não somar eventos

Cada evento que toca uma guia faz o mesmo trabalho: `BiService.recalcularGuia(guiaId)` lê o estado atual da guia nos serviços públicos dos módulos donos (`faturamento`, `spike`, `retorno`, `contratos`), aplica funções puras de `@pep/contracts` e regrava as linhas daquela guia nas projeções. Consequências:

- Ordem e duplicidade de eventos não importam (entrega ao menos uma vez, ADR-PEP-0008): o resultado é o mesmo.
- Reprocessar do zero não depende do outbox (que guarda só 30 dias): é rodar `recalcularGuia` para todas as guias (`pnpm --filter @pep/api bi:reprocessar --tenant=<id> [--desde=2026-06]`). Necessário quando muda o mapeamento alerta → Tabela 38 ou uma definição.
- O `bi` nunca lê tabela de outro módulo por SQL; usa o serviço público do dono (ADR-PEP-0002).
- Rede de segurança: job noturno `bi-reconciliar` recalcula as guias com atendimento nos últimos 120 dias (pega transição sem evento, como cancelamento).

`# ponytail:` consultas de agregação direto sobre `bi_guias` com índice por período; tabela agregada mensal ou view materializada só se o painel passar de p95 800 ms com 100 mil guias. Postgres no mesmo banco, sem armazém, até medir volume (resolve a antiga Q3).

### Modelo de dados (módulo `bi`)

Todas as tabelas de tenant com `tenant_id uuid NOT NULL` e RLS forçada por convenção em `drizzle/rls`, acesso só por `Db.comTenant`. Projeções podem ser regravadas (não são dado clínico nem de faturamento: são derivadas). **Nenhuma guarda `paciente_id`, nome, carteirinha ou texto clínico.**

```sql
CREATE TABLE bi_guias (                       -- grão: guia
  guia_id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  profissional_id uuid NOT NULL,
  operadora_id uuid NOT NULL, operadora_nome text NOT NULL,
  tipo text NOT NULL,                          -- consulta | sp_sadt
  origem text NOT NULL CHECK (origem IN ('atendimento','manual','receita_esquecida')),
  numero_prestador text NOT NULL,              -- para o relatório (G-004812); não é PII
  atendimento_em date NOT NULL,
  competencia char(7) NOT NULL,                -- do atendimento: '2026-10'
  status text NOT NULL,                        -- espelho do status da guia (RFC-PEP-006)
  modo_ia text NOT NULL CHECK (modo_ia IN ('desligado','silencioso','assistido')),  -- da análise de referência
  analise_referencia_id uuid,
  analise_exibida_em timestamptz,              -- 1ª exibição ao médico (assistido)
  ultima_decisao_medico_em timestamptz,
  valor_faturado numeric(12,2) NOT NULL DEFAULT 0,      -- revisão enviada (ou atual, se não enviada)
  enviada_em date,
  desfecho text CHECK (desfecho IN ('pago','glosado_parcial','glosado_total')),
  desfecho_em date,
  valor_pago numeric(12,2), valor_glosado numeric(12,2),
  valor_evitado numeric(12,2) NOT NULL DEFAULT 0,             -- regra E (assistido)
  valor_teria_sido_evitado numeric(12,2) NOT NULL DEFAULT 0,  -- regra S (silencioso)
  recalculado_em timestamptz NOT NULL
);
CREATE INDEX ix_bi_guias_unidade_comp ON bi_guias (tenant_id, unidade_id, competencia);
CREATE INDEX ix_bi_guias_prof_comp ON bi_guias (tenant_id, profissional_id, competencia);

CREATE TABLE bi_itens_ia (                    -- grão: guia × item da IA, deduplicado entre reanálises
  tenant_id uuid NOT NULL, guia_id uuid NOT NULL REFERENCES bi_guias(guia_id),
  chave text NOT NULL,                         -- estável: 'alerta:<tipo>:<tuss|guia>' | 'sug_cid:<cid>' | 'sug_tuss:<tuss>' | 'justificativa:<tuss|guia>'
  item_tipo text NOT NULL CHECK (item_tipo IN ('alerta','sugestao_cid','sugestao_tuss','justificativa')),
  alerta_tipo text, severidade text, codigo_tuss text,
  motivos_glosa text[] NOT NULL DEFAULT '{}',  -- códigos Tabela 38 pelo mapa vigente; vazio = sem mapeamento
  exibido boolean NOT NULL,                    -- emitido em análise do modo assistido
  decisao text CHECK (decisao IN ('aceito','editado','rejeitado')),  -- última decisão; NULL = ignorado
  decisao_papel text,                          -- medico | faturista
  decidido_em timestamptz,
  valor_item numeric(12,2) NOT NULL DEFAULT 0, -- valor do procedimento do item na revisão enviada; item de guia = valor_faturado
  PRIMARY KEY (guia_id, chave)
);

CREATE TABLE bi_glosas (                      -- grão: item glosado do desfecho
  tenant_id uuid NOT NULL, guia_id uuid NOT NULL REFERENCES bi_guias(guia_id),
  ordem smallint NOT NULL,
  codigo_tuss text,                            -- NULL = glosa da guia inteira
  codigo_glosa text NOT NULL,                  -- Tabela 38
  valor_glosado numeric(12,2) NOT NULL,
  teria_sido_evitada boolean NOT NULL DEFAULT false,   -- regra S
  PRIMARY KEY (guia_id, ordem)
);
CREATE INDEX ix_bi_glosas_tuss ON bi_glosas (tenant_id, codigo_tuss);

CREATE TABLE bi_producao_diaria (             -- grão: unidade × profissional × dia
  tenant_id uuid NOT NULL, unidade_id uuid NOT NULL, profissional_id uuid NOT NULL, dia date NOT NULL,
  atendimentos int NOT NULL,
  PRIMARY KEY (tenant_id, unidade_id, profissional_id, dia)
);

-- Referência global (sem tenant, como TUSS/CID): curada pelo consultor de faturamento (Q2).
-- Nasce vazia: nenhum código da Tabela 38 é inventado em seed, teste ou exemplo (usar <TABELA38_…>).
CREATE TABLE bi_mapa_alerta_glosa (
  alerta_tipo text NOT NULL,                   -- tipo de alerta da Spike
  codigo_glosa text NOT NULL,                  -- Tabela 38 [validar versão]
  vigencia_inicio date NOT NULL, vigencia_fim date,
  validado_por text NOT NULL,                  -- quem conferiu na fonte e quando
  PRIMARY KEY (alerta_tipo, codigo_glosa, vigencia_inicio)
);
```

### Definições (fixas, em `@pep/contracts`, mostradas na tela)

As regras vivem em `packages/contracts/src/gestao.ts` como funções puras (`projetarGuiaBi`, `calcularPainel`) e a constante `DEFINICOES_PAINEL` (texto do rodapé e do "Como calculamos" da tela 15). A API e os testes usam as mesmas funções.

| Termo | Definição |
| :--- | :--- |
| **Análise de referência** | Última análise `completa` ou `parcial` da guia feita sobre revisão anterior ou igual à enviada (se ainda não enviada, a última). O `modo_ia` da guia é o modo gravado nessa análise (`analises_tissia_ai.modo`, ADR-PEP-0006); sem análise, `desligado` |
| **Taxa de glosa** | Σ `valor_glosado` / Σ `valor_faturado` das guias **com desfecho** no período (competência do atendimento). Guia sem desfecho não entra no denominador |
| **Linha de base** | Taxa de glosa das guias em modo `silencioso` das últimas 3 competências em silencioso da unidade até o fim do período. Abaixo de 100 guias com desfecho, a API devolve `amostraInsuficiente: true` e a tela mostra "amostra insuficiente (N guias)" em vez do número (Q1) |
| **Taxa atual** | Taxa de glosa das guias em modo `assistido` no período |
| **Glosa evitada (regra E)** | Em guia `assistido` com desfecho: para cada item `alerta` exibido, com decisão `aceito` ou `editado` e `motivos_glosa` não vazio, se não houver em `bi_glosas` glosa com código desses motivos no mesmo TUSS (ou na guia, se o alerta for de guia), soma `valor_item`. Cada item conta uma vez; o total da guia é limitado ao `valor_faturado`. Alerta sem mapeamento não conta |
| **Glosa que teria sido evitada (regra S)** | Em guia `silencioso` glosada: cada item glosado cujo código está em `motivos_glosa` de um alerta emitido (e não exibido) na análise de referência, no mesmo TUSS ou de guia, vira `teria_sido_evitada`; soma o `valor_glosado` desses itens |
| **Aceite das sugestões da IA** | Sobre itens `exibido` no período: `aceito` %, `editado` %, `ignorado` % (rejeitado ou sem decisão quando a guia foi enviada). Soma 100% |
| **Tempo médio de revisão** | Média, por guia `assistido` com ao menos uma decisão de médico, de `ultima_decisao_medico_em − analise_exibida_em`; intervalos acima de 10 min são descartados (sessão abandonada) e contados à parte |
| **Receita esquecida recuperada** | Σ `valor_pago` das guias com `origem = receita_esquecida` com desfecho `pago` ou `glosado_parcial` no período; subtítulo = guias com essa origem criadas no período |
| **Atendimentos/dia** | Média de `bi_producao_diaria.atendimentos` nos dias com atendimento do profissional no período |
| **Assistido desde** | Menor `atendimento_em` de guia `assistido` da unidade depois da última guia `silencioso` |

Rodapé fixo (texto do design, mantido): "Linha de base = 3 meses em modo silencioso. Glosa evitada conta apenas alertas aceitos cujas guias foram pagas sem glosa pelo mesmo motivo." O "Como calculamos" da tela 15 mostra a tabela acima.

### O que esta RFC exige das outras

| RFC | Precisa entregar | Situação no código hoje |
| :--- | :--- | :--- |
| 005 (P1) | `analises_tissia_ai.modo` gravado em toda análise; vínculo da análise com `guia_id`/`guia_revisao_id`; `analises_tissia_ai.exibida_em` gravado na primeira entrega ao médico em modo assistido (um `UPDATE … WHERE exibida_em IS NULL`); `AnaliseTissiaAiConcluida` e `FeedbackRegistrado` pelo outbox; decisões com `papel` | `analises_tissia_ai` liga só ao atendimento, sem `modo` nem `exibida_em`; `decisoes_analise` sem `papel` |
| 006 (P1) | `guias.operadora_id`, valor por procedimento da revisão enviada, `GuiaRascunhoCriada { guiaId, origem }`, `GuiaRevisada`, `GuiaValidada`, `LoteEnviado { loteId, guiaIds }` | Guia com `valor_total_centavos` sintético; sem revisões nem lote |
| 007 | `DesfechoRegistrado { guiaId }` e, no serviço, itens glosados com `codigo_tuss`, `codigo_glosa` (Tabela 38) e valor | Rascunho |
| 009 | Guia criada a partir da lista com `origem = receita_esquecida` | Rascunho |
| 004 | `AtendimentoFechado { atendimentoId, unidadeId, profissionalId, fechadoEm }` | Publicado (D2) |

### Eventos

- **Consome** (todos por `processarUmaVez`, consumidor `bi`):

| Evento | Efeito |
| :--- | :--- |
| `GuiaRascunhoCriada`, `GuiaRevisada`, `GuiaValidada` | `recalcularGuia(guiaId)` |
| `AnaliseTissiaAiConcluida`, `FeedbackRegistrado` | `recalcularGuia(guiaId)` |
| `LoteEnviado` | `recalcularGuia` de cada guia do lote |
| `DesfechoRegistrado` | `recalcularGuia(guiaId)` |
| `AtendimentoFechado` | recontagem de `bi_producao_diaria` do (unidade, profissional, dia) pelo serviço do prontuário |

`RetornoImportado` não é consumido (os desfechos chegam um a um por `DesfechoRegistrado`).
- **Publica:** nada.

### API (módulo `bi`)

Permissão nova `gestao:ler` em `@pep/contracts`, só no perfil `gestor` (a matriz da tela 25 é da RFC-PEP-014). A rota da demo `GET /api/v1/guias/glosa-evitada` (`guias:ler`) não muda.

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/gestao/painel?de=2026-06&ate=2026-10&unidade=atual` | Tudo que a tela 24 mostra, em uma resposta. `unidade=atual` (padrão, unidade da sessão) ou `todas` (unidades em que o usuário tem vínculo de gestor no tenant). Período máximo de 24 competências |
| `GET` | `/api/v1/gestao/glosas-evitaveis?de=&ate=&unidade=&modo=silencioso&operadora=&cursor=` | Relatório da tela 15 (e detalhe do card da 24): uma linha por item glosado evitável (`modo=silencioso`, regra S) ou por item evitado (`modo=assistido`, regra E), com número da guia, operadora, competência, TUSS, código e rótulo do motivo, valor e tipo de alerta. Paginação por cursor |

Exemplo de resposta do painel (valores fictícios; códigos como placeholder):

```json
{
  "periodo": { "de": "2026-06", "ate": "2026-10" },
  "unidade": "atual",
  "definicoes": "…DEFINICOES_PAINEL…",
  "dadosAte": "2026-11-09T03:00:00-03:00",
  "kpis": {
    "glosaEvitadaCentavos": 2138000,
    "taxaGlosa": { "atualPct": 3.2, "linhaBasePct": 8.4, "janelaBase": ["2026-06", "2026-08"], "guiasBase": 412, "amostraInsuficiente": false },
    "receitaRecuperada": { "valorCentavos": 647000, "guiasGeradas": 29 },
    "aceiteIa": { "aceitoPct": 81, "editadoPct": 12, "ignoradoPct": 7, "itens": 1530 },
    "tempoRevisaoS": { "media": 12, "descartadas": 14 },
    "alertasSemMapeamento": 3
  },
  "porOperadora": [{ "operadoraId": "…", "nome": "Operadora A", "linhaBasePct": 9.1, "assistidoPct": 3.0 }],
  "mensal": [{ "competencia": "2026-09", "taxaPct": 5.1, "modo": "assistido", "guiasComDesfecho": 140 }],
  "assistidoDesde": "2026-09-01",
  "porMedico": [{ "profissionalId": "…", "nome": "Dra. Exemplo A", "guias": 318, "taxaGlosaPct": 2.4, "aceiteIaPct": 88, "revisaoMediaS": 9, "atendimentosDia": 17 }],
  "tussMaisGlosados": [{ "codigoTuss": "<TUSS_…>", "termo": "…", "valorGlosadoCentavos": 183600 }]
}
```

Campos sem dado (antes da RFC-PEP-007, por exemplo) vêm `null`, e a resposta traz `pendencias: ["desfechos"]` para a UI mostrar o estado certo. Schema estrito em `@pep/contracts` (`PainelGestorSchema`, `GlosasEvitaveisPaginaSchema`). Meta: p95 ≤ 800 ms com 100 mil guias no período.

### Regras

- Guia `cancelada` sai de todas as contas. Guia sem desfecho entra só em aceite da IA, tempo de revisão, guias e produção.
- `por médico` ordena por nome, nunca por taxa de glosa (evita ranking punitivo); só o perfil Gestor vê.
- Mudar o mapa alerta → Tabela 38 exige `bi:reprocessar` das competências afetadas; o painel mostra "Dados até <data do último recálculo>".
- Taxas com denominador zero vêm `null` ("sem guias com desfecho"), nunca 0%.
- Valores em `numeric(12,2)` no banco e em centavos inteiros na API (como o relatório do Marco D).
- O painel nunca chama a Spike: usa `AnaliseTissiaAi`, decisões e desfechos já gravados no PEP. O relatório de piloto da Spike (Spike RFC-007) serve só para conferência cruzada no piloto.

## UI

Tela 24 em `/gestao` (`apps/ui/src/modulos/financeiro/PainelGestor.tsx`, design G24). A P2 volta ao layout do design: os blocos "Por motivo da glosa" e "Por operadora · lote sintético" da demo saem da 24 (o "por motivo" vira o relatório da tela 15). Período e unidade na URL via `nuqs`.

| Bloco do design | Alimentado por | Estado |
| :--- | :--- | :--- |
| Selo "Dados ilustrativos" | — | Some com dado real; na demo continua "Simulação sobre guias sintéticas" |
| Chip "Período: jun–out/2026" | `de`/`ate` | Real (fatia 1) |
| Chip "Todas as unidades" | `unidade` | Real (fatia 1) |
| "Exportar PDF" | `window.print()` com folha de estilo de impressão | Real (fatia 1). `# ponytail:` PDF no servidor só se pedirem envio agendado por e-mail |
| KPI "Glosa evitada (modo assistido)" | `kpis.glosaEvitadaCentavos` | Fatia 2 (RFC-PEP-007 + Q2). Antes: valor "—" e "Sem desfechos importados (RFC-PEP-007)" |
| KPI "Taxa de glosa" + "linha de base 8,4% (jun–ago)" | `kpis.taxaGlosa` | Fatia 2 |
| KPI "Receita esquecida recuperada" | `kpis.receitaRecuperada` | Fatia 3 (RFC-PEP-009); antes, inerte com `title` |
| KPI "Aceite das sugestões da IA" | `kpis.aceiteIa` | Fatia 1 (só depende da RFC-PEP-005) |
| KPI "Tempo médio de revisão" | `kpis.tempoRevisaoS` | Fatia 1 |
| "Taxa de glosa por operadora" (base × assistido) | `porOperadora` | Fatia 2 |
| "Evolução mensal da taxa de glosa" + selo "Assistido desde 01/09" e faixa sombreada | `mensal`, `assistidoDesde` | Fatia 2 |
| Tabela "Por médico · set–out" | `porMedico` | Fatia 1: guias, aceite IA, revisão média, atend./dia; fatia 2: taxa de glosa |
| "Códigos TUSS mais glosados" (valor glosado, top 5) | `tussMaisGlosados` | Fatia 2 |
| Rodapé com as definições | `DEFINICOES_PAINEL` | Real |

Bloco sem dado fica visível e inerte (`aria-disabled` + `title` com a RFC), nunca removido. Aviso "N alertas sem mapeamento para a Tabela 38" quando `alertasSemMapeamento > 0`.

Tela 15 (`/configuracoes/modos-ia`, RFC-PEP-005): só o bloco do relatório "glosas que teriam sido evitadas" e o "Como calculamos" são desta RFC, via `GET /gestao/glosas-evitaveis?modo=silencioso`, visíveis a quem tem `gestao:ler`.

Demo: enquanto existir o ambiente de demonstração (`VITE_AMBIENTE_DEMO=true`), a 24 continua montada sobre `GET /guias/glosa-evitada` e o lote sintético. Quando o ambiente for descartado (release plan §12.9), apagar `lote_sintetico_guias`, a rota, o seed e o componente da demo.

## Segurança e LGPD

- Projeções sem PII de paciente nem texto clínico (teste de schema garante que nenhuma tabela `bi_*` tem `paciente_id`, nome, carteirinha ou campo de texto livre clínico).
- Métricas por médico são dado pessoal do profissional (desempenho): finalidade de gestão registrada no ROPA, acesso só com `gestao:ler`, abertura do painel gera `gestao.painel_visualizado` na auditoria (RFC-PEP-014).
- `glosas-evitaveis` traz número da guia (identificador indireto); sem nome do paciente.
- Logs só com ids e tempos; nunca o corpo da resposta.
- RLS em todas as `bi_*` de tenant; `bi_mapa_alerta_glosa` é referência global (sem dado de tenant), escrita só por migração/seed revisado.

## Critérios de aceite

- [ ] `recalcularGuia` é idempotente e independente de ordem: aplicar os eventos de uma guia em todas as permutações, com duplicatas, gera a mesma linha (teste de propriedade).
- [ ] `bi:reprocessar` do zero produz exatamente o estado obtido de forma incremental (teste comparando as tabelas).
- [ ] Tabela de casos das regras E e S em `@pep/contracts` (teste unitário): alerta aceito + guia paga → evitada; aceito + glosada pelo mesmo motivo → não; aceito + glosada por outro motivo → evitada; rejeitado ou ignorado → não; alerta sem mapeamento → não, e conta em `alertasSemMapeamento`; silencioso com alerta emitido + glosa do mesmo motivo e TUSS → teria sido evitada; vários alertas na mesma guia → limitado ao `valor_faturado`.
- [ ] Linha de base com menos de 100 guias com desfecho → `amostraInsuficiente: true` e `linhaBasePct: null`.
- [ ] Médico, faturista e recepção recebem `403` em `/gestao/*`; gestor recebe `200`.
- [ ] RLS: gestor do tenant A não vê projeções do tenant B; `unidade=todas` só soma unidades com vínculo de gestor.
- [ ] Nenhuma tabela `bi_*` tem coluna de PII de paciente (teste de schema).
- [ ] Guia cancelada não aparece em nenhuma soma.
- [ ] `GET /gestao/painel` com 100 mil guias no período: p95 ≤ 800 ms.
- [ ] Tela 24 ligada à API com dados reais: blocos sem dado mostram o estado "sem desfechos" com a RFC no `title`; impressão gera PDF legível (A4, paisagem).
- [ ] Tela 15: relatório "glosas que teriam sido evitadas" paginado e coerente com o card da 24 (mesma soma).

## Riscos

- Atribuição causal fraca (sazonalidade, mudança de mix de operadora): mostrar por operadora e período comparável; texto nunca promete causalidade.
- Métrica por médico pode virar ranking punitivo: só Gestor, ordem alfabética.
- Sem desfechos importados (RFC-PEP-007) o núcleo do painel fica vazio: a fatia 1 entrega o que não depende deles para o painel já ser útil no piloto.
- Mapeamento alerta → Tabela 38 errado infla ou zera a glosa evitada: mapa versionado, validado por consultor, com reprocessamento.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | Linha de base de 3 competências é viável no piloto (o design de produto fala em 30 dias de silencioso)? A amostra mínima de 100 guias com desfecho serve? | Produto | Não (padrão: 3 competências, 100 guias; ajustável na constante) |
| Q2 | Mapeamento tipo de alerta da Spike → códigos da Tabela 38 | Consultor de faturamento **[validar]** | Fatia 2 (KPI de glosa evitada e relatório da 15) |
| Q3 | Valor da glosa evitada: valor do item do alerta (padrão desta RFC) ou da guia inteira? | Produto + consultor | Não |
| Q4 | Período por competência do atendimento (padrão) ou do pagamento? | Gestor do piloto | Não |
| Q5 | O médico vê os próprios números (sem os dos colegas)? | Produto + CMO | Não (fora da P2) |
| Q6 | "Todas as unidades" deve consolidar por clínica (CNPJ) em vez de por tenant? | Produto | Não |

## Fatias de entrega

| # | Entrega | Pronto quando |
| :--- | :--- | :--- |
| 0 | **Entregue (Marco D, 09/11/2026):** painel sobre o lote sintético (Notas de implementação) | — |
| 1 | Tabelas `bi_*` e mapa vazio, `gestao.ts` em `@pep/contracts` (schemas, `DEFINICOES_PAINEL`, `projetarGuiaBi`, `calcularPainel`), `recalcularGuia` + consumidores + `bi-reconciliar` + `bi:reprocessar`, permissão `gestao:ler`, `GET /gestao/painel` com aceite IA, tempo de revisão e por médico (sem taxa); tela 24 com filtros e impressão. Depende da 005 e 006 da P1 | Critérios de idempotência, reprocessamento, RLS, 403 e PII verdes; tela 24 com os blocos da fatia 1 reais |
| 2 | Desfechos: taxa atual, linha de base, regras E e S, por operadora, mensal com marco, TUSS, taxa por médico; `GET /gestao/glosas-evitaveis`; relatório da tela 15. Depende da 007 e da Q2 | Tabela de casos E/S verde; card e relatório da 15 somam igual; p95 medido |
| 3 | Receita esquecida recuperada. Depende da 009 | KPI real com guias de `origem = receita_esquecida` |

## Notas de implementação

**Marco D / D4 (09/11/2026): painel de "Glosas que teriam sido evitadas" sobre o lote sintético (telas 15 e 24).**

- Tabela `lote_sintetico_guias` (módulo `faturamento`, com RLS por tenant) semeada por `seed-demo.ts` a partir de `gerarLoteSinteticoDemo()` (`@pep/contracts`): 200 guias sintéticas derivadas dos casos do golden set da Spike (Ortopedia DUT 65/110 e Cardiologia DUT 3/10/38/54) e precificadas pela `TABELA_PRECOS_SINTETICA_CENTAVOS`. Regra determinística de desfecho: `nao_conforme` → `glosada`, `conforme` → `paga`.
- `GET /api/v1/guias/glosa-evitada` (`guias:ler`) agrega em tempo real os totais (`guiasAnalisadas`, `guiasGlosadas`, `comAlertaPrevio`, `valorFaturadoCentavos`, `valorGlosadoCentavos`, `valorEvitavelCentavos`, `taxaGlosaBasePct`, `taxaGlosaSimuladaAssistidoPct`) e as quebras `porMotivo`, `porOperadora` e `porProcedimento`.
- Telas `/gestao` (`apps/ui/src/pages/PainelGestor.tsx`, tela 24) e `/modos` (`apps/ui/src/pages/Modos.tsx`, tela 15) ligadas à API real, com o selo e aviso obrigatório em destaque: *"Simulação sobre guias sintéticas"*.
- Fidelidade ao design (G24/E15): o selo amarelo "Simulação sobre guias sintéticas" substitui "Dados ilustrativos" e o aviso completo vai para o rodapé. Sem dados para "Evolução mensal", "Por médico", "Receita esquecida", "Aceite das sugestões" e "Tempo médio de revisão": no lugar entram "Por motivo da glosa", tabela por operadora e KPIs do lote (guias analisadas, valor faturado, cobertura de alertas). Na tela 15 o modo atual é "Assistido" (é o que a demo de fato faz); rádios e opções de comportamento ficam desabilitados até o modo por unidade (D3).
- **Desvios:** cálculo direto sobre `lote_sintetico_guias` em vez de projeções alimentadas por eventos (`fato_guia`, `fato_analise`, `agg_glosa_mensal`); sem exportação em PDF e sem filtro por período/médico real (dependem da ingestão de retorno na RFC-PEP-007, P2).
- *(Nota de 09/11/2026, ao detalhar a RFC: as telas foram depois movidas para `apps/ui/src/modulos/financeiro/PainelGestor.tsx` e `apps/ui/src/modulos/antiglosa/Modos.tsx`. As projeções `fato_*`/`agg_*` do rascunho viraram `bi_guias`, `bi_itens_ia`, `bi_glosas` e `bi_producao_diaria`, sem tabela agregada.)*

**Etapa 3 do §13.4 / Fatia 1 (15/11/2026): projeções por evento e Painel do Gestor real.**

- Tabelas `bi_guias`, `bi_itens_ia`, `bi_glosas`, `bi_producao_diaria` (RLS forçada por tenant) e `bi_mapa_alerta_glosa` (global, vazia) criadas via migração Drizzle `0042_bi_projecoes.sql`.
- `@pep/contracts`: permissão `gestao:ler` concedida apenas ao perfil `gestor`; `packages/contracts/src/gestao.ts` com schemas estritos (`PainelGestorQuerySchema`, `PainelGestorSchema`), `DEFINICOES_PAINEL`, `RODAPE_PAINEL_FIXO`, `projetarGuiaBi` e `calcularPainel` (incluindo regras E e S e linha de base com amostra mínima de 100 guias).
- `@pep/api`: módulo `bi` (`BiService`, `BiController` em `GET /api/v1/gestao/painel` com auditoria `gestao.painel_visualizado`), consumidor `bi.projecoes` no outbox/BullMQ, job noturno `bi-reconciliar` e script `pnpm --filter @pep/api bi:reprocessar`.
- `@pep/ui`: `/gestao` (`PainelGestor.tsx`) lê `GET /api/v1/gestao/painel` fora da demo (`VITE_AMBIENTE_DEMO !== 'true'`), com filtros `de`/`ate`/`unidade` via `nuqs`, exportação em PDF (`window.print()` em A4 paisagem) e blocos da fatia 2/3 visíveis e inertes (`aria-disabled` + `title` da RFC); na demo (`VITE_AMBIENTE_DEMO === 'true'`), o painel sintético continua intacto.
- **Desvios da fatia 1:**
  - `bi_guias.operadora_id` e `bi_guias.profissional_id` são `NULL`-able no schema porque `guias` ainda não possui `operadora_id` (entra com a RFC-PEP-008) e permite rascunho sem profissional.
  - `bi_guias.origem` assume `'atendimento'` na fatia 1 até a entrada de `receita_esquecida` (RFC-PEP-009).
  - `decisoes_analise` ainda não persiste `papel`; `TissiaAiService.analisesParaBi` assume `'medico'` para as decisões existentes na fatia 1.
  - `GET /api/v1/gestao/glosas-evitaveis`, relatório na tela 15 e benchmark p95 de 100 mil guias ficam para a fatia 2 junto com desfechos (RFC-PEP-007) e mapa Tabela 38 (Q2).

**Demo de dados fictícios (19/11/2026, PR #30 do `tissia`, em rascunho).** Ver [[demo-dados-ficticios]] e [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026]].

- **Desvio:** na demo, `/gestao` passou a ler o **Painel real** (`GET /api/v1/gestao/painel`) sobre o histórico semeado, e o lote sintético do Marco D virou a seção "Simulação sobre o lote sintético" abaixo do painel (`SimulacaoLoteSintetico`, só com `VITE_AMBIENTE_DEMO=true`; usa o endpoint de glosa evitada já existente). Sem a flag, `PainelGestor.tsx` renderiza só o painel real, como antes. Na demo há o selo "Histórico fictício de demonstração" e o sufixo "resultado não real" nos KPIs de glosa evitada e taxa.
- **Narrativa estável por data** (módulo `receita` do seed, `bi` reprocessado ao final): taxa de glosa 18,5% (linha de base) para 9,2% (assistido), glosa evitada R$ 14.792 (`1479200` centavos) e as quatro operadoras A–D com redução; `alertasSemMapeamento = 0`. Simulação do lote sintético: 200 guias, 46 glosadas, 39 com alerta, R$ 39.956.
- **UI:** "Taxa de glosa (por valor)" (o cálculo é por valor, `calcularTaxaPct`); o KPI "Receita esquecida recuperada" mostra "—" e "disponível em breve" (fatia 3, RFC-PEP-009 não implementada).
- **Armadilha corrigida:** o módulo `ia` do seed apagava as análises do tenant inteiro e destruía a linha de base (Painel com `linhaBasePct: null`); agora só toca o elenco e as guias sem atendimento. Os códigos TUSS do lote sintético (`gerarLoteSinteticoDemo`) foram trocados pelos oficiais (30733073, 30713153, 30713137, 41101316, 40701140, 20102038), valores idênticos.

### Onda P3 — back-end: B2 ranking de operadoras glosadoras (2026-11-25)

- **Rotas** (permissão `gestao:ler` para leitura; decisão exige `glosas:escrever` ou `admin:configurar`): `GET /api/v1/gestao/ranking-operadoras-glosadoras`, `GET /api/v1/gestao/operadoras/:registroAns/regras-sugeridas` e `POST /api/v1/gestao/operadoras/:registroAns/regras/:regraId/decisao`.
- **Fonte:** agregação sobre `glosas` + `desfechos` + cadastro de operadoras (CADOP). Nenhum dado clínico, nome de paciente ou texto de evolução é lido, gravado ou auditado; os "motivos" exibidos são os códigos/textos operacionais de glosa já existentes (limitados a 300 caracteres).
- **Regras sugeridas:** determinísticas (`sugerirRegrasGlosa` em `@pep/contracts`), geradas a partir de motivos com pelo menos 3 ocorrências nos últimos 90 dias e persistidas em `regras_glosa_sugeridas` (migração `0077_onda_p3_b2_ranking_glosas.sql`, RLS forçada). Regera o mesmo conjunto em chamadas repetidas (idempotência por tenant + operadora + motivo). Aplicar/rejeitar exige justificativa de 5–500 caracteres, grava evento no outbox e trilha selada (`gestao.regra_glosa_decidida`); uma sugestão decidida não pode ser decidida de novo (409).
- **Desvio documentado:** o `faturadoCentavos` usado no denominador da taxa vem dos desfechos das guias glosadas no período (e não do total faturado bruto do painel), para manter o ranking auditável sem ler prontuários.
- **Testes:** `onda-p3-b2-ranking-glosas.spec.ts` cobre cálculo da taxa, principais motivos, validação de filtro/permissão, determinismo das sugestões, auditoria da decisão, idempotência da decisão e isolamento RLS entre tenants.
