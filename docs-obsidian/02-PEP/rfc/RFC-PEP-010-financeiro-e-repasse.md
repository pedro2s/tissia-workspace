---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-010-financeiro-e-repasse.md
---

# RFC-PEP-010 — Financeiro e repasse médico

**Fase:** P2 (fatia 1 antecipável para a P1) · **Status:** Em implementação (fatia 1 — particular simples com recibo PDF — entregue em 15/11/2026; ADR-PEP-0014 aceita; fatias 2–6 planejadas, ver Notas de implementação) · **ADRs:** ADR-PEP-0008, ADR-PEP-0014, ADR-PEP-0017, ADR-PEP-0019
**Telas:** 23 (Financeiro e repasse)
**Requisitos:** RF-FIN-01 a RF-FIN-04; propõe ao PRD-001 RF-FIN-05 e RF-FIN-06 (ver Escopo)
**Depende de:** RFC-PEP-003 (atendimento particular na agenda), RFC-PEP-004 (`AtendimentoFechado`), RFC-PEP-006 (`LoteEnviado`, valor e profissional da guia, `sequencias`), RFC-PEP-007 (`RetornoImportado`, `DesfechoRegistrado`: pago, glosado, revertido), RFC-PEP-008 (prazo de pagamento do contrato; com fallback), RFC-PEP-014 (auditoria); ADR-PEP-0014 (particular simples no MVP, em Proposta)

## Objetivo

Contas a receber e a pagar por unidade, de particular e de convênio, conciliação bancária por arquivo e **repasse médico calculado sobre o valor realmente recebido**, já descontadas as glosas — o diferencial que só existe porque o PEP fecha o ciclo da guia. Todo valor é rastreável até a guia ou o atendimento que o originou, e nenhum valor aprovado é editado: correção é movimento novo.

## Escopo

**Dentro (P2)**

- [RF-FIN-04] Particular simples (antecipável para a P1, ADR-PEP-0014): tabela de preços particular por unidade, recebível criado ao fechar o atendimento particular, registro do recebimento (forma, valor, desconto), recibo em PDF numerado e estorno com motivo. Sem parcelamento a receber, maquininha integrada ou NFS-e.
- [RF-FIN-01] Contas a receber: convênio por guia (agrupadas por lote, criadas no envio do lote) e particular por atendimento; saldo calculado a partir de movimentos imutáveis (glosa, reversão, recebimento, estorno, baixa).
- [RF-FIN-01] Contas a pagar: lançamento manual, recorrências mensais (aluguel, laboratório parceiro) e conta gerada pelo repasse aprovado; pagamento registrado manualmente ou pela conciliação.
- [RF-FIN-03] Conciliação bancária por arquivo OFX (CNAB 240 **[validar necessidade]**, Q1): importação idempotente, sugestão de vínculo crédito ↔ retorno/lote e débito ↔ conta a pagar por valor e data, vínculo manual, desfazer com motivo. Nunca concilia sozinha.
- [RF-FIN-02] Repasse: regra por profissional com vigência (percentual sobre o recebido, dedução de impostos e taxa), cálculo por competência, decomposição "faturado − glosado − não pago = recebido", aprovação, demonstrativo exportável (PDF/CSV) e ajuste na competência seguinte.
- [RF-FIN-05] *(proposto)* Contas a receber por idade (0–30, 31–60, 61–90, > 90 dias), a pagar em 7 dias e conciliação pendente no painel lateral.
- [RF-FIN-06] *(proposto)* Imutabilidade financeira: movimentos append-only, repasse aprovado congelado, correção por estorno ou ajuste, trilha de auditoria em toda alteração.

**Fora**

- NFS-e e integração com ERP/contabilidade: P3, pela API pública (RFC-PEP-014). Split de pagamento.
- Integração bancária por API (Open Finance, PIX via PSP): arquivo na P2 (decisão do rascunho mantida).
- Regras de repasse por procedimento fixo, faixa ou produção: só `percentual_sobre_recebido` na P2 (tipos fechados; Q2).
- Conciliação automática de recebimentos de cartão por adquirente (taxa, antecipação): o crédito do adquirente é vinculado manualmente aos recebimentos particulares do dia.

## Design

### Conceitos

- **Recebível:** o que a clínica tem a receber por **uma** guia (convênio) ou **um** atendimento (particular), com `profissional_id` (executante, base do repasse) e `competencia_producao` (mês do atendimento).
- **Movimento:** fato financeiro imutável sobre um recebível. `saldo = previsto − glosa + estorno_glosa − recebimento + estorno_recebimento − baixa`.
- **Recebido** é dinheiro em conta: no convênio, o crédito conciliado (data do crédito no extrato); no particular, o registro do recebimento na recepção. O demonstrativo da operadora (RFC-PEP-007) informa glosa e valor pago por guia, mas só a conciliação transforma o pago em recebido (Q4).
- **Competência:** `char(7)` `AAAA-MM`, como `lotes.competencia`.

### Modelo de dados (módulo `financeiro`)

```sql
-- todas as tabelas: tenant_id uuid NOT NULL, RLS forçada por convenção em drizzle/rls (ADR-PEP-0019);
-- ids de outros módulos (guia, lote, atendimento, paciente, profissional, operadora, retorno) sem FK (ADR-PEP-0017).
-- Dinheiro em centavos inteiros (bigint, como na RFC-PEP-007; guias.valor_total_centavos é integer hoje); percentuais em pontos-base (6000 = 60%).

CREATE TABLE config_financeiro (
  unidade_id                  uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  prazo_pagamento_padrao_dias smallint NOT NULL DEFAULT 30,   -- vencimento previsto sem contrato (RFC-PEP-008) [validar]
  dia_pagamento_repasse       smallint NOT NULL DEFAULT 10 CHECK (dia_pagamento_repasse BETWEEN 1 AND 28),
  atualizado_por uuid NOT NULL, atualizado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tabela_particular (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  chave text NOT NULL,                   -- tipo do agendamento ('consulta','retorno') ou código TUSS do procedimento
  descricao text NOT NULL, valor_centavos bigint NOT NULL CHECK (valor_centavos >= 0),
  vigencia_inicio date NOT NULL, vigencia_fim date,          -- nova vigência encerra a anterior; nunca sobrescreve
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_tab_part ON tabela_particular (unidade_id, chave, vigencia_inicio DESC);

CREATE TABLE recebiveis (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  origem text NOT NULL CHECK (origem IN ('guia','particular')),
  guia_id uuid, lote_id uuid, operadora_id uuid,             -- convênio
  atendimento_id uuid NOT NULL, paciente_id uuid NOT NULL, profissional_id uuid NOT NULL,
  competencia_producao char(7) NOT NULL,
  data_emissao date NOT NULL,                                -- envio do lote (convênio) ou atendimento (particular); base do aging
  vencimento_previsto date NOT NULL,
  valor_previsto_centavos bigint CHECK (valor_previsto_centavos >= 0),  -- NULL só em particular sem preço, até o 1º recebimento
  saldo_centavos bigint NOT NULL,                           -- espelho dos movimentos, atualizado na mesma transação
  status text NOT NULL CHECK (status IN ('aberto','pago_parcial','pago','glosado_total','baixado')),  -- espelho
  criado_em timestamptz NOT NULL DEFAULT now(),
  CHECK ((origem = 'guia') = (guia_id IS NOT NULL))
);
CREATE UNIQUE INDEX ux_receb_guia ON recebiveis (guia_id) WHERE guia_id IS NOT NULL;
CREATE UNIQUE INDEX ux_receb_particular ON recebiveis (atendimento_id) WHERE origem = 'particular';
CREATE INDEX ix_receb_aberto ON recebiveis (unidade_id, status, data_emissao);
CREATE INDEX ix_receb_prof ON recebiveis (unidade_id, profissional_id, competencia_producao);

-- append-only (trigger + REVOKE UPDATE, DELETE no papel app_rw)
CREATE TABLE movimentos_recebivel (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  recebivel_id uuid NOT NULL REFERENCES recebiveis(id),
  tipo text NOT NULL CHECK (tipo IN ('glosa','estorno_glosa','recebimento','estorno_recebimento','baixa')),
  valor_centavos bigint NOT NULL CHECK (valor_centavos > 0),
  data_efetiva date NOT NULL,                                -- data do crédito, do desfecho ou do recebimento
  origem_ref text NOT NULL,                                  -- 'desfecho_evento:<id>' | 'conciliacao:<id>' | 'recebimento:<id>' | 'manual:<id>'
  motivo text,                                               -- baixa e estorno: obrigatório
  criado_por uuid,                                           -- NULL = sistema (consumidor de evento)
  criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (recebivel_id, tipo, origem_ref)                    -- idempotência dos consumidores
);
CREATE INDEX ix_mov_data ON movimentos_recebivel (recebivel_id, data_efetiva);

CREATE TABLE recebimentos_particulares (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  recebivel_id uuid NOT NULL REFERENCES recebiveis(id),
  valor_centavos bigint NOT NULL CHECK (valor_centavos > 0),
  desconto_centavos bigint NOT NULL DEFAULT 0 CHECK (desconto_centavos >= 0),   -- vira movimento 'baixa' (motivo desconto)
  forma text NOT NULL CHECK (forma IN ('dinheiro','pix','cartao_debito','cartao_credito','outro')),
  numero_recibo text NOT NULL,                               -- R-000123, sequencial por unidade (`sequencias`, RFC-PEP-006)
  recebido_em timestamptz NOT NULL DEFAULT now(), recebido_por uuid NOT NULL,
  estornado_em timestamptz, estornado_por uuid, motivo_estorno text,   -- única mudança permitida (NULL → valor), por trigger
  UNIQUE (unidade_id, numero_recibo)
);

CREATE TABLE contas_pagar (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  categoria text NOT NULL CHECK (categoria IN ('repasse','aluguel','laboratorio','fornecedor','imposto','outros')),
  descricao text NOT NULL CHECK (char_length(descricao) <= 120),
  favorecido text,                                           -- razão social; repasse usa profissional_id (sem CPF em texto)
  profissional_id uuid, repasse_id uuid, recorrencia_id uuid,
  competencia char(7) NOT NULL, vencimento date NOT NULL,
  valor_centavos bigint NOT NULL CHECK (valor_centavos > 0),
  status text NOT NULL CHECK (status IN ('aberta','paga','cancelada')),
  pago_em date, motivo_cancelamento text,
  criado_por uuid, criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (recorrencia_id, competencia)                       -- job mensal idempotente
);
CREATE INDEX ix_pagar_venc ON contas_pagar (unidade_id, status, vencimento);

CREATE TABLE recorrencias_pagar (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  categoria text NOT NULL, descricao text NOT NULL, favorecido text,
  valor_centavos bigint NOT NULL CHECK (valor_centavos > 0),
  dia_vencimento smallint NOT NULL CHECK (dia_vencimento BETWEEN 1 AND 28),
  inicio char(7) NOT NULL, fim char(7), ativo boolean NOT NULL DEFAULT true,
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE contas_bancarias (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  banco_codigo text NOT NULL, agencia text NOT NULL, conta_final char(4) NOT NULL, apelido text NOT NULL,
  ativo boolean NOT NULL DEFAULT true
);

CREATE TABLE extratos_importados (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  conta_bancaria_id uuid NOT NULL REFERENCES contas_bancarias(id),
  formato text NOT NULL CHECK (formato IN ('ofx')),          -- 'cnab240' se a Q1 pedir
  arquivo_hash bytea NOT NULL, periodo_inicio date, periodo_fim date,
  lancamentos_novos integer NOT NULL, lancamentos_repetidos integer NOT NULL,
  importado_por uuid NOT NULL, importado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (conta_bancaria_id, arquivo_hash)
);

CREATE TABLE lancamentos_bancarios (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  conta_bancaria_id uuid NOT NULL REFERENCES contas_bancarias(id),
  extrato_id uuid NOT NULL REFERENCES extratos_importados(id),
  identificador text NOT NULL,                               -- FITID do OFX; sem FITID: hash(data, valor, memo, ordem)
  data date NOT NULL,
  valor_centavos bigint NOT NULL CHECK (valor_centavos <> 0),   -- + crédito, − débito
  descricao_cifrada bytea,                                   -- MEMO/NAME do OFX: pode ter nome/CPF do pagador (cifrado por tenant, KMS)
  status text NOT NULL DEFAULT 'pendente' CHECK (status IN ('pendente','conciliado','ignorado')),
  motivo_ignorado text,
  UNIQUE (conta_bancaria_id, identificador)                  -- extratos sobrepostos não duplicam
);
CREATE INDEX ix_lanc_pendente ON lancamentos_bancarios (conta_bancaria_id, status, data);

CREATE TABLE conciliacoes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  lancamento_id uuid NOT NULL REFERENCES lancamentos_bancarios(id),
  diferenca_centavos bigint NOT NULL DEFAULT 0,             -- lançamento − soma dos alvos
  tipo_diferenca text CHECK (tipo_diferenca IN ('retencao_tributaria','tarifa','outro')),
  motivo text,
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now(),
  desfeita_em timestamptz, desfeita_por uuid, motivo_desfazer text,   -- única mudança permitida
  CHECK (diferenca_centavos = 0 OR tipo_diferenca IS NOT NULL)
);
CREATE UNIQUE INDEX ux_concil_ativa ON conciliacoes (lancamento_id) WHERE desfeita_em IS NULL;

CREATE TABLE conciliacao_alvos (
  tenant_id uuid NOT NULL, conciliacao_id uuid NOT NULL REFERENCES conciliacoes(id),
  alvo_tipo text NOT NULL CHECK (alvo_tipo IN ('retorno','lote','recebivel','recebimento_particular','conta_pagar')),
  alvo_id uuid NOT NULL, valor_centavos bigint NOT NULL,
  PRIMARY KEY (conciliacao_id, alvo_tipo, alvo_id)
);

CREATE TABLE regras_repasse (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL, profissional_id uuid NOT NULL,
  tipo text NOT NULL DEFAULT 'percentual_sobre_recebido' CHECK (tipo IN ('percentual_sobre_recebido')),
  percentual_bp smallint NOT NULL CHECK (percentual_bp BETWEEN 0 AND 10000),
  deducao_bp smallint NOT NULL DEFAULT 0 CHECK (deducao_bp BETWEEN 0 AND 10000),   -- impostos e taxa (Q3)
  vigencia_inicio char(7) NOT NULL, vigencia_fim char(7),    -- nova regra encerra a anterior; nunca sobrescreve
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE repasses (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  profissional_id uuid NOT NULL, competencia char(7) NOT NULL,
  regra_id uuid NOT NULL REFERENCES regras_repasse(id),
  percentual_bp smallint NOT NULL, deducao_bp smallint NOT NULL,   -- cópia da regra no cálculo
  recebido_centavos bigint NOT NULL, deducao_centavos bigint NOT NULL, base_centavos bigint NOT NULL,
  ajustes_centavos bigint NOT NULL DEFAULT 0, valor_centavos bigint NOT NULL,
  termos jsonb NOT NULL,         -- { faturado, glosado, nao_pago, recebido_producao, recebido_anterior } em centavos
  status text NOT NULL CHECK (status IN ('calculado','aprovado','pago')),
  calculado_em timestamptz NOT NULL, calculado_por uuid NOT NULL,
  aprovado_em timestamptz, aprovado_por uuid, conta_pagar_id uuid,
  UNIQUE (unidade_id, profissional_id, competencia)
);

CREATE TABLE itens_repasse (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  repasse_id uuid NOT NULL REFERENCES repasses(id),
  tipo text NOT NULL CHECK (tipo IN ('recebimento','estorno_recebimento','ajuste')),
  movimento_id uuid REFERENCES movimentos_recebivel(id),     -- NULL só em ajuste
  recebivel_id uuid, data_efetiva date,
  recebido_centavos bigint NOT NULL,                        -- com sinal (estorno negativo)
  repasse_centavos bigint NOT NULL,                         -- rateio do valor do repasse
  motivo text, criado_por uuid,
  CHECK ((tipo = 'ajuste') = (movimento_id IS NULL)),
  CHECK (tipo <> 'ajuste' OR motivo IS NOT NULL)
);
CREATE UNIQUE INDEX ux_item_movimento ON itens_repasse (movimento_id) WHERE movimento_id IS NOT NULL;  -- um movimento repassa uma vez
```

Triggers: `movimentos_recebivel` sem `UPDATE`/`DELETE`; `itens_repasse` só aceita `INSERT`/`DELETE` enquanto o repasse está `calculado`; `repasses` aprovado só muda para `pago`. `recebimentos_particulares` e `conciliacoes` só aceitam a transição de estorno/desfazer (`NULL → valor`). Toda escrita grava `auditoria_eventos` (RFC-PEP-014) com ator, recurso e motivo.

### Máquinas de estados

```
Recebível (espelho, derivado dos movimentos):
aberto ──recebimento < saldo──▶ pago_parcial ──recebimento = saldo──▶ pago
aberto | pago_parcial ──glosa = saldo──▶ glosado_total ──estorno_glosa──▶ pago_parcial | aberto
qualquer ──baixa do saldo──▶ baixado

Conta a pagar:   aberta ──pagamento (manual ou conciliação)──▶ paga · aberta ──cancelar (motivo)──▶ cancelada
Repasse:         calculado ──(recalcular: substitui itens)──▶ calculado ──aprovar──▶ aprovado ──conta paga──▶ pago
```

Repasse aprovado não volta. Erro descoberto depois vira item `ajuste` (positivo ou negativo, com motivo) no repasse da competência seguinte; estorno de recebimento já repassado entra sozinho na próxima competência (o movimento `estorno_recebimento` ainda não foi repassado).

### Origem dos recebíveis e movimentos

| Fato | Gatilho | Efeito |
| :--- | :--- | :--- |
| Atendimento particular fechado | `AtendimentoFechado` (atendimento sem convênio) | Recebível `particular`, previsto pela `tabela_particular` vigente (chave = TUSS do procedimento, senão tipo do agendamento); sem preço → `valor_previsto_centavos = NULL` até o recebimento |
| Recebimento particular | `POST /financeiro/particular/recebimentos` | Linha em `recebimentos_particulares`, movimento `recebimento` (e `baixa` "desconto" se houver), recibo numerado, `RecebimentoRegistrado` |
| Lote enviado | `LoteEnviado` | Um recebível `guia` por guia do lote: previsto = valor da revisão enviada, profissional executante, `data_emissao` = envio, vencimento = envio + prazo de pagamento do contrato (RFC-PEP-008) ou `prazo_pagamento_padrao_dias` |
| Glosa / reversão | `DesfechoRegistrado` | Delta entre o glosado do desfecho e o já lançado: positivo → `glosa`, negativo → `estorno_glosa` (`origem_ref = desfecho_evento:<evento_id>`). Delta torna o consumidor idempotente e imune a ordem |
| Crédito conciliado | `POST /financeiro/conciliacoes` | `recebimento` por guia do retorno/lote, no valor pago informado (RFC-PEP-007) ou no saldo (lote sem demonstrativo), `data_efetiva` = data do crédito; `RecebimentoRegistrado` |
| Conciliação desfeita | `POST /conciliacoes/{id}/desfazer` | `estorno_recebimento` para cada recebimento criado por ela |
| Perda / inadimplência | `POST /financeiro/receber/{id}/baixas` | `baixa` com motivo |

`LoteEnviado` e não `LoteGerado` (como dizia o rascunho): lote gerado pode ser descartado e regerado antes do envio (RFC-PEP-006); a dívida da operadora nasce no envio.

### Conciliação bancária

1. Upload OFX → parser no worker; hash do arquivo repetido → `200` com "arquivo já importado"; lançamento com `identificador` já existente é contado como repetido e ignorado. Descrição cifrada antes de gravar.
2. **Sugestão** (calculada na leitura, sem tabela): para crédito pendente, candidatos = retornos importados ainda não conciliados da unidade (valor líquido informado pelo demonstrativo, RFC-PEP-007) e lotes enviados sem retorno (soma dos saldos), com valor **exato** e data do crédito em `[data prevista − 5, data prevista + 10]` dias; para débito, contas a pagar abertas com valor exato e vencimento em ± 5 dias. Um candidato → "Sugestão: lote L-0455 · Operadora A"; vários → lista; nenhum → "Sem correspondência automática" e vínculo manual.
3. **Conciliar** exige clique: o usuário confirma os alvos e, se o crédito difere da soma, declara a diferença (`retencao_tributaria`, `tarifa`, `outro` + motivo). Retenção na fonte pela operadora não reduz o recebido da guia; o recebido é o valor pago da guia e a retenção fica registrada na conciliação para a contabilidade **[validar com contador]**.
4. **Desfazer** com motivo gera estornos; se algum recebimento já está em repasse aprovado, o estorno entra no próximo repasse (ver acima).

`# ponytail:` sugestão só por valor exato e janela de data, sem fuzzy nem leitura do texto do extrato. Se o piloto mostrar muitos créditos sem sugestão (ex.: operadora que soma lotes num crédito), adicionar combinação de 2–3 alvos.

### Cálculo do repasse

Para a unidade `U`, o profissional `P` e a competência `C` (data de corte = último dia de `C`):

```
itens        = movimentos 'recebimento' e 'estorno_recebimento' dos recebíveis de P em U,
               com data_efetiva ≤ corte e ainda não repassados (ux_item_movimento)
recebido     = Σ recebimentos − Σ estornos                   (centavos)
deducao      = arred_par(recebido × deducao_bp / 10000)
base         = recebido − deducao
repasse      = arred_par(base × percentual_bp / 10000) + Σ ajustes
```

- `arred_par` = arredondamento bancário (meio para o par) sobre inteiros, uma vez por repasse; o valor é rateado entre os itens pelo maior resto, de modo que **Σ `itens.repasse_centavos` = `repasse.valor_centavos`** sempre. Função pura `calcularRepasse(itens, regra, ajustes)` em `@pep/contracts` (aritmética em `bigint`), usada pela API e pelo teste de propriedade.
- Regra aplicada = regra vigente no último dia de `C`. Sem regra vigente → profissional aparece como "Sem regra de repasse" e não é calculado.
- O que não foi pago fica no saldo e entra na competência em que o crédito ocorrer. Glosa revertida (`estorno_glosa`) volta para o saldo; quando o dinheiro entra, vira recebimento e base.
- Repasse negativo (estornos > recebimentos) não gera conta a pagar: na aprovação vira item `ajuste` "saldo devedor AAAA-MM" na competência seguinte.
- **Decomposição da tela** (`termos`), sobre a produção de `C` (recebíveis com `competencia_producao = C`) na data de corte: `faturado` = Σ previsto; `glosado` = glosas − reversões + baixas; `nao_pago` = Σ saldo; `recebido_producao` = recebido desses recebíveis neste repasse. Vale `faturado − glosado − nao_pago = recebido_producao`. `recebido_anterior` = recebido neste repasse de produção anterior; quando > 0 a tela soma um termo "+ recebido de competências anteriores" antes de "Recebido".
- Exemplo do design (Dr. Exemplo B, regra 55% com dedução de 15%): recebido 3.165.000 → dedução 474.750 → base 2.690.250 → 2.690.250 × 0,55 = 1.479.637,5 → `arred_par` = 1.479.638 (R$ 14.796,38).
- **Aprovar** recalcula na mesma transação e compara com o `total_esperado_centavos` enviado pela tela; diferente → `409` "Os valores mudaram desde o cálculo" (conciliação feita no meio). Igual → `aprovado`, conta a pagar `repasse` com vencimento no `dia_pagamento_repasse` do mês seguinte à competência, `RepasseAprovado`.
- Recalcular é idempotente: mesma entrada, mesmos itens e valores.

### API (módulo `financeiro`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/financeiro/resumo?competencia=2026-10` | Painel lateral: aging (4 faixas), a pagar em 7 dias, conciliação pendente (2 primeiras com sugestão + total), conciliados no mês |
| `GET` | `/api/v1/financeiro/receber?origem=&status=&operadora=&profissional=&faixa=&cursor=` | Contas a receber (cursor) + totais |
| `GET` | `/api/v1/financeiro/receber/{id}` | Recebível com movimentos |
| `POST` | `/api/v1/financeiro/receber/{id}/baixas` | `{ "valor_centavos": 35000, "motivo": "Inadimplência após 3 cobranças" }` |
| `GET/POST` | `/api/v1/financeiro/tabela-particular` | Lista vigente / nova vigência de um item |
| `POST` | `/api/v1/financeiro/particular/recebimentos` | `{ "atendimento_id": "…", "valor_centavos": 35000, "desconto_centavos": 0, "forma": "pix" }` → `201 { "numero_recibo": "R-000123" }` |
| `POST` | `/api/v1/financeiro/particular/recebimentos/{id}/estorno` | `{ "motivo": "Valor lançado em duplicidade" }` |
| `GET` | `/api/v1/financeiro/particular/recebimentos/{id}/recibo.pdf` | Recibo gerado na hora (download auditado) |
| `GET/POST` | `/api/v1/financeiro/pagar` | Lista (`?status=&vencimento_ate=`) / lança conta |
| `POST` | `/api/v1/financeiro/pagar/{id}/pagamento` | `{ "pago_em": "2026-11-10" }` |
| `POST` | `/api/v1/financeiro/pagar/{id}/cancelamento` | `{ "motivo": "…" }` |
| `GET/POST/PATCH` | `/api/v1/financeiro/recorrencias` | Recorrências (PATCH só `valor`, `fim`, `ativo`; vale para meses ainda não gerados) |
| `GET/POST` | `/api/v1/financeiro/contas-bancarias` | Contas da unidade |
| `POST` | `/api/v1/financeiro/extratos` | `multipart` (`conta_bancaria_id`, arquivo OFX ≤ 5 MB) → `201 { "novos": 42, "repetidos": 3 }` |
| `GET` | `/api/v1/financeiro/conciliacao?status=pendente` | Lançamentos com sugestões |
| `POST` | `/api/v1/financeiro/conciliacoes` | `{ "lancamento_id": "…", "alvos": [{ "tipo": "retorno", "id": "…", "valor_centavos": 3148660 }], "diferenca": { "tipo": "retencao_tributaria", "valor_centavos": 0, "motivo": null } }` |
| `POST` | `/api/v1/financeiro/conciliacoes/{id}/desfazer` | `{ "motivo": "…" }` |
| `POST` | `/api/v1/financeiro/lancamentos/{id}/ignorar` | `{ "motivo": "Transferência entre contas" }` |
| `GET/POST` | `/api/v1/financeiro/regras-repasse` | Regras por profissional (POST abre nova vigência) |
| `POST` | `/api/v1/financeiro/repasses/calcular` | `{ "competencia": "2026-10" }` → recalcula os `calculado` da unidade e cria os que faltam |
| `GET` | `/api/v1/financeiro/repasses?competencia=2026-10` | Linhas por profissional (faturado, glosado, recebido, base, %, repasse, a receber) + `termos` |
| `GET` | `/api/v1/financeiro/repasses/{id}` | Itens (guia/atendimento, data, id curto do paciente, TUSS, recebido, repasse) |
| `POST` | `/api/v1/financeiro/repasses/{id}/ajustes` | `{ "valor_centavos": -12000, "motivo": "Estorno de repasse a maior em 09/2026" }` (só `calculado`) |
| `POST` | `/api/v1/financeiro/repasses/aprovacao` | `{ "competencia": "2026-10", "itens": [{ "repasse_id": "…", "total_esperado_centavos": 2106300 }] }` |
| `GET` | `/api/v1/financeiro/repasses/{id}/demonstrativo.pdf` · `/api/v1/financeiro/repasses/demonstrativos.csv?competencia=` | Demonstrativo individual e planilha da competência |

Permissões (RFC-PEP-001): leitura com `financeiro:ler` (faturista, gestor, admin); escrita, conciliação, regras e cálculo com `financeiro:escrever` (gestor). **Novas, a incluir na matriz de `@pep/contracts` e na RFC-PEP-001:** `financeiro:receber_particular` (recepção e gestor: tabela particular só leitura, registrar e estornar recebimento, recibo); `financeiro:aprovar_repasse` (gestor; separado de `escrever` para a clínica poder exigir outra pessoa); `repasse:ler_proprio` (médico: só os próprios repasses e demonstrativos).

### Eventos

- **Consome:** `AtendimentoFechado` (particular → recebível), `LoteEnviado` (recebíveis de convênio), `DesfechoRegistrado` (glosa e reversão), `RetornoImportado` (habilita o retorno como candidato da conciliação). Consumidores idempotentes por `eventos_processados` e pela chave `(recebivel_id, tipo, origem_ref)`.
- **Publica:** `RecebimentoRegistrado { recebivel_id, origem, atendimento_id, guia_id, movimento_id }` (consumido pela RFC-PEP-009, para `recibo_emitido`, e por `bi`); `RepasseAprovado { repasse_id, profissional_id, competencia }` (consumido por `notificacoes`: aviso ao profissional, sem valores no texto). Os dois entram na lista de auxiliares do README.
- **Contrato esperado da RFC-PEP-007** (em detalhamento): por guia, `valor_cobrado`, `valor_glosado` e `valor_pago` em centavos, com o status final (pago após recurso = glosado menor); por retorno, valor líquido total e data de pagamento informada. Sem isso, a fatia 5 não começa.

### Regras

- **Aging** por `data_emissao` (hoje − emissão), só recebíveis com saldo > 0: 0–30, 31–60, 61–90, > 90 dias.
- Particular sem preço na tabela: a recepção informa o valor no recebimento e o previsto passa a ser esse valor (única transição `NULL → valor`). Desconto vira `baixa` com motivo "desconto", nunca recebimento menor sem explicação.
- Recebimento particular maior que o saldo → `422`; dois recebimentos parciais são permitidos (paciente paga em duas vezes no balcão).
- Recibo: número sequencial por unidade, sem buraco tolerado (sequência na mesma transação); recibo estornado continua existindo com a marca "ESTORNADO". Conteúdo do recibo (emitente PF ou PJ, CPF do pagador para IRPF, descrição do serviço sem diagnóstico) **[validar com contador]**.
- Conta a pagar de repasse não é cancelada pela rota comum (`409`): repasse aprovado é imutável, e a correção é um ajuste na competência seguinte.
- Recorrência gera a conta do mês no dia 1 (job diário idempotente pela `UNIQUE (recorrencia_id, competencia)`).
- Financeiro não lê evolução nem CID: itens de repasse mostram TUSS e termo, nunca texto clínico.

## UI (tela 23, rota `/financeiro`)

Hoje estática em `apps/ui/src/modulos/financeiro/Financeiro.tsx`. Aba ativa e competência em `nuqs` (`?aba=repasse&competencia=2026-10`); unidade vem da sessão.

| Bloco do design | Fonte | Observação |
| :--- | :--- | :--- |
| Chips "Competência: 10/2026" e "Unidade Centro" | `competencia` (URL), sessão | Troca de unidade é a do shell |
| Abas Contas a receber · Contas a pagar · Conciliação bancária · Repasse médico | rotas acima | O design só detalha o Repasse; as outras três abas usam tabela + filtros do DS com as colunas de `GET /receber`, `GET /pagar` e `GET /conciliacao` |
| "Como o repasse é calculado · Dra. Exemplo A" (equação) e selo "Regra: sobre o recebido" | `GET /repasses` (`termos`, regra) | Profissional selecionado na tabela; termo extra "+ recebido de competências anteriores" quando > 0; "Impostos e taxa (N%)" = `deducao_bp` |
| Texto "Valores ainda não pagos entram…" | constante | — |
| Repasse por profissional (Faturado, Glosado, Recebido, Base, %, Repasse, A receber) | `GET /repasses` | "A receber" = `nao_pago`; clique abre itens (`GET /repasses/{id}`); status por linha |
| Exportar demonstrativos · Aprovar repasse | CSV/PDF, `POST /repasses/aprovacao` | Aprovar confirma em diálogo com o total; desabilitado sem `financeiro:aprovar_repasse` |
| Contas a receber por idade | `GET /resumo` | Barras proporcionais ao maior valor |
| Conciliação bancária (2 pendentes, sugestão, Conciliar / Vincular, "N conciliados") | `GET /resumo`, `POST /conciliacoes` | "Vincular" abre a aba Conciliação com o lançamento |
| A pagar nos próximos 7 dias | `GET /resumo` | Inclui a conta de repasse aprovada |

**Prévia** até a RFC-PEP-007: coluna/termo "Glosado" e sugestões por retorno aparecem com o selo "Prévia" (sem desfecho, o glosado é 0 e a conciliação de convênio é por lote, no saldo). O selo "Dados ilustrativos" e o "Prévia" do menu saem quando a fatia 2 entrar.

## Segurança e LGPD

- Nada deste módulo vai à Spike.
- O financeiro não guarda nem mostra texto clínico. Paciente aparece pelo id curto (`P-XXXX`); o nome só no recibo, para quem tem `financeiro:receber_particular`.
- Descrição do extrato bancário (pode conter nome e CPF de pagador) cifrada por tenant (KMS, mesmo mecanismo da RFC-PEP-002), decifrada só na tela de conciliação para `financeiro:escrever`; nunca em log nem em evento. Arquivo OFX não é guardado depois do parse (só o hash).
- Recibo gerado sob demanda a partir dos dados imutáveis; não fica PDF com PII no armazenamento. Download do recibo e do demonstrativo grava auditoria.
- Toda escrita grava `auditoria_eventos`; logs com ids, contagens e latência, nunca corpo de requisição, valores por paciente ou descrição de extrato.
- Retenção: dados financeiros 5 anos após o pagamento **[validar]** (modelo de dados).

## Critérios de aceite

- [ ] RLS: tenant A não lê recebível, extrato, repasse ou conta do tenant B (teste com dois tenants).
- [ ] `UPDATE`/`DELETE` em `movimentos_recebivel` falha (inclusive para o papel admin da aplicação); itens de repasse `aprovado` não mudam.
- [ ] Teste de propriedade de `calcularRepasse` (laço com entradas aleatórias de semente fixa no Vitest, sem dependência nova): para qualquer conjunto de itens e regra, Σ itens = total, resultado não depende da ordem dos itens e recalcular dá o mesmo valor; os 3 profissionais do design reproduzem exatamente os valores da tela 23.
- [ ] `DesfechoRegistrado` reentregue ou fora de ordem não duplica glosa; glosa revertida volta ao saldo e, após o crédito conciliado, aparece no repasse da competência do crédito.
- [ ] Recebimento não pago na competência fica no "A receber" e entra no repasse da competência em que o crédito for conciliado.
- [ ] Importar o mesmo OFX duas vezes (ou extratos sobrepostos) não duplica lançamentos.
- [ ] Nenhuma conciliação acontece sem `POST /conciliacoes`; diferença sem tipo → `422`; desfazer gera estornos e o próximo repasse os desconta.
- [ ] Aprovar com total diferente do recalculado → `409`; aprovar cria exatamente uma conta a pagar por repasse positivo e nenhuma para repasse negativo (vira ajuste na competência seguinte).
- [ ] Recibos com numeração única e sem buraco sob 20 recebimentos concorrentes na mesma unidade.
- [ ] Matriz de permissões: recepção registra recebimento e não vê repasse; médico vê só o próprio repasse; faturista lê e não escreve.
- [ ] Tela 23 ligada à API (4 abas, painel lateral, aprovação), com estados de carregando, vazio e erro.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | Formatos de extrato dos bancos das clínicas piloto: OFX basta, ou algum só entrega CNAB 240? **[validar layout por banco]** | Produto | Fatia 4 (formato) |
| Q2 | Modelos de repasse praticados nas clínicas piloto: percentual sobre recebido cobre todos? Há fixo por procedimento, faixa, produção, ou percentuais diferentes para particular e convênio? | Gestor piloto | Fatia 6 |
| Q3 | "Impostos e taxa" é um percentual único por profissional, ou varia por origem (convênio × particular, taxa de cartão)? | Gestor piloto + contador | Fatia 6 (só o desenho da regra) |
| Q4 | Base do repasse = crédito conciliado (adotado) ou pagamento informado no demonstrativo? Clínicas que não conciliam o extrato ficariam sem repasse | Gestor piloto | Fatia 6 |
| Q5 | Repasse por unidade (adotado) ou consolidado por clínica para profissional que atende em várias unidades? | Produto | Nada (default por unidade) |
| Q6 | Particular: recibo simples basta ou alguma clínica piloto exige NFS-e já na entrada? Emitente é a clínica (PJ) ou o profissional (PF)? Recibo para IRPF exige CPF do pagador? **[validar com contador]** — mesma condição de aceite da ADR-PEP-0014 | Pedro/Thiago + contador | Fatia 1 |
| Q7 | Operadoras retêm tributos na fonte (IR, PIS/COFINS/CSLL) no pagamento à clínica? Se sim, o tratamento como diferença de conciliação basta para a contabilidade? **[validar com contador]** | Contador | Fatia 4 (só o rótulo da diferença) |
| Q8 | Prazo de pagamento por operadora: incluir "prazo de pagamento (dias)" no cadastro de contrato da RFC-PEP-008 (hoje o rascunho lista envio, autorização, senha e recurso) | RFC-PEP-008 | Nada (fallback `prazo_pagamento_padrao_dias`) |
| Q9 | Contrato da RFC-PEP-007 para o financeiro (valor pago por guia em centavos, valor líquido e data de pagamento do retorno) confirmado no detalhamento dela | RFC-PEP-007 | Fatia 5 |

Nenhuma lacuna no contrato da Spike: o financeiro não fala com ela.

## Fatias de entrega

1. **Particular simples** (antecipável para a P1) — `config_financeiro`, `tabela_particular`, `recebiveis`/`movimentos_recebivel` só com origem `particular`, consumidor de `AtendimentoFechado`, recebimento, desconto, estorno, recibo PDF, `RecebimentoRegistrado`, permissão `financeiro:receber_particular`; ação "Registrar recebimento" a partir do atendimento particular. Depende da ADR-PEP-0014 aceita e da Q6.
2. **Contas a receber de convênio** — consumidor de `LoteEnviado`, baixas manuais, aging, aba Contas a receber, `GET /resumo` (aging). Depende do envio de lote da RFC-PEP-006.
3. **Contas a pagar** — `contas_pagar`, `recorrencias_pagar` e job mensal, pagamento e cancelamento, aba Contas a pagar, "A pagar nos próximos 7 dias".
4. **Conciliação bancária** — contas bancárias, importação OFX, sugestão por lote (saldo) e por conta a pagar, conciliar/desfazer/ignorar, movimentos de recebimento de convênio, aba e cartão de Conciliação. Depende da Q1.
5. **Glosa, reversão e retorno** — consumidores de `DesfechoRegistrado` e `RetornoImportado`, sugestão por retorno (valor líquido), conciliação no valor pago por guia, "Glosado" sem selo de prévia. Depende da RFC-PEP-007 e da Q9.
6. **Repasse** — `regras_repasse`, `calcularRepasse` em `@pep/contracts` com teste de propriedade, calcular/ajustar/aprovar, conta de repasse, demonstrativos, `RepasseAprovado`, permissões `financeiro:aprovar_repasse` e `repasse:ler_proprio`, aba Repasse médico fiel ao design. Depende das fatias 1, 2 e 4 e das Q2–Q4; com a fatia 5 ausente, o glosado fica 0 com "Prévia".

## Notas de implementação

- **Fatia 1 (15/11/2026 — Particular simples com recibo em PDF, ADR-PEP-0014):**
  - Tabelas `config_financeiro`, `tabela_particular`, `recebiveis`, `movimentos_recebivel` e `recebimentos_particulares` criadas com RLS forçada por tenant (`0039_financeiro_particular_simples.sql`) e triggers de imutabilidade (`0040_financeiro_particular_triggers.sql` + `REVOKE UPDATE, DELETE ON movimentos_recebivel FROM app_rw`).
  - Contratos Zod estritos em `@pep/contracts` (`financeiro.ts`) e permissão `financeiro:receber_particular` atribuída a `recepcao` e `gestor` em `identidade.ts`.
  - Consumidor `financeiro.atendimento_fechado` registrado no worker BullMQ; adicionalmente, `garantirRecebivelParticular` é executado de forma idempotente na leitura/registro para evitar corrida entre o fechamento do atendimento e o recebimento imediato no balcão.
  - Recibo em PDF gerado sob demanda em memória (`recibo-pdf.ts`, PDF 1.4 sem dependência externa e sem persistir PII em armazenamento), com marca `ESTORNADO` após estorno e auditoria em `auditoria_eventos`.
  - UI: ação "Registrar recebimento", recebimento parcial/desconto, download do recibo PDF e estorno integrados em `Atendimento.tsx` e `Revisao.tsx` (`RecebimentoParticular.tsx`) para quem tem `financeiro:receber_particular`. NFS-e fora do MVP conforme ADR-PEP-0014.
