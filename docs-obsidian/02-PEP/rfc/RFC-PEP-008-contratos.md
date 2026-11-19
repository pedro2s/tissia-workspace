---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-008-contratos.md
---

# RFC-PEP-008 — Contratos, tabelas negociadas e autorização prévia

**Fase:** P2 (fatia 0 pode ser antecipada para a P1, ver abaixo) · **Status:** Em implementação (fatias 1–6 entregues em 15/11/2026; ADRs 0012 e 0014 aceitas; fora XLSX, E2E dedicado e `provisionar_regras_autorizacao` via [[SPK-11-contratos-auditar-substituicao|SPK-11]], ver Notas de implementação) · **ADRs:** ADR-PEP-0003, ADR-PEP-0005, ADR-PEP-0008, ADR-PEP-0012, ADR-PEP-0014, ADR-PEP-0017
**Telas:** 21 (Contratos e tabelas negociadas)
**Requisitos:** RF-CON-01 a RF-CON-03; alimenta RF-AGE-03/RF-AGE-06 (agenda) e as validações de valor, prazo e autorização da RFC-PEP-006
**Depende de:** RFC-PEP-000 (RLS, outbox com fila por consumidor, worker com agendamento), RFC-PEP-005 (SpikeAdapter, só a fatia 6), RFC-PEP-006 (consome preço, limite e prazo; fornece o volume de guias para o impacto); consumida por RFC-PEP-003, RFC-PEP-006, RFC-PEP-007 e RFC-PEP-009

## Objetivo

Ter, por unidade × operadora, a fonte de verdade do que foi **negociado** (preço, pacote, limite, exigência de autorização prévia, prazos) e do que o **faturamento usa** (preço cobrado e regras de autorização em vigor), e auditar continuamente um contra o outro para pegar reajuste não aplicado, valor cadastrado errado e autorização não parametrizada. A auditoria roda no PEP sempre; a Spike recebe o contrato para aprender e cruzar, sem nunca ser condição para a tela funcionar.

## Escopo

**Dentro (P2)**

- [RF-CON-01] Catálogo de operadoras (registro ANS) e contrato por operadora, abrangendo uma ou mais unidades da clínica: número, vigência, versão TISS, canal de envio (portal/webservice), prazo de envio, prazo de resposta de autorização, validade de senha, prazo de recurso, limite do texto de recurso, dias de retorno sem cobrança, PDF do contrato.
- [RF-CON-01] Tabela negociada por TUSS, **versionada por vigência** (nunca sobrescrita): valor, limite por guia, exigência de autorização segundo o contrato, observações. Reajuste gera nova versão.
- [RF-CON-01] Pacotes (código do pacote, itens inclusos, valor).
- [RF-CON-01] Importação da tabela por CSV com prévia do diff (novos, alterados, removidos, inválidos). XLSX quando a Q1 confirmar.
- [RF-CON-01] Preços de faturamento ("Sistema"), também com vigência, e "Aplicar valores do contrato" em lote. Substituem a `TABELA_PRECOS_SINTETICA_CENTAVOS` do Marco D.
- [RF-CON-02] Regras de autorização prévia em **tipos fechados** (`sempre`, `eletivo`, `acima_de_quantidade`) por lista de TUSS, usadas pela agenda, pela guia e pela mesa do faturista via função pura em `@pep/contracts`.
- [RF-CON-03] Auditoria contrato × sistema: ao publicar versão da tabela, ao mudar regras ou preços, sob demanda ("Reauditar") e semanal. Resultado local sempre; envio a `POST /v1/contratos/auditar` da Spike quando o conteúdo mudou.
- Exportação das divergências em CSV.

**Fora**

- Leitura automática do PDF do contrato (a Spike RFC-008 cita, não entregou como API).
- Tabela de preço do atendimento **particular**: fica com o financeiro (ADR-PEP-0014, RFC-PEP-010). O esquema de preços aceita `registro_ans` só de operadora.
- Webservice de autorização prévia na operadora (ADR-PEP-0012, P2 via parceiro): esta RFC só diz **quando** exigir.
- Reconciliação retroativa automática de retornos já importados quando um reajuste é cadastrado depois (ver Regras).

## Design

### Dois lados: contrato e sistema

| Lado | O que é | Tabelas | Quem usa |
| :--- | :--- | :--- | :--- |
| **Contrato** | O que foi negociado com a operadora, como está no contrato/aditivo | `tabelas_negociadas`, `tabela_negociada_itens`, `pacotes`, `pacote_itens` | Conciliação do retorno (RFC-PEP-007: quanto a operadora deveria pagar), auditoria |
| **Sistema** | O que o PEP efetivamente cobra e exige | `precos_faturamento`, `regras_autorizacao` | Guia (valor), agenda e guia (autorização), auditoria (`itens_pep` da Spike) |

Separar os dois é o que permite a tela 21 ("Contrato × Sistema") e o contrato da Spike (`itens_contrato` × `itens_pep`). "Aplicar valores do contrato" copia do lado contrato para o lado sistema, item a item, com autor.

### Modelo de dados (módulo `contratos`)

Toda tabela de tenant tem `tenant_id uuid NOT NULL` com RLS por convenção em `drizzle/rls` (ADR-PEP-0019), acesso só por `Db.comTenant`. FKs só dentro do módulo; `unidade_id`/`clinica_id` são `uuid` validados no serviço (ADR-PEP-0017). Dinheiro em centavos inteiros (`*_centavos bigint`), como `guias.valor_total_centavos`; o SpikeAdapter converte para reais.

```sql
-- Referência global (sem tenant_id; só leitura para a aplicação, ADR-PEP-0003): carga de dados abertos da ANS [validar fonte]
-- Carga do CADOP (dados abertos da ANS, "operadoras de planos de saúde ativas", CC BY), por script com manifesto (ADR-PEP-0024, RFC-PEP-021 fatia 3).
-- Operadora que some do CADOP fica `ativa = false` (nunca apagada: contratos e guias antigos apontam para ela).
CREATE TABLE operadoras (
  registro_ans   text PRIMARY KEY,
  razao_social   text NOT NULL,
  nome_fantasia  text,
  cnpj           text,
  modalidade     text,                     -- como publicada no CADOP
  uf             text,
  ativa          boolean NOT NULL DEFAULT true,
  versao_ref     text NOT NULL,            -- data da publicação do CADOP usada na carga
  atualizado_em  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE contratos (
  id                     uuid PRIMARY KEY,
  tenant_id              uuid NOT NULL,
  clinica_id             uuid NOT NULL,
  registro_ans           text NOT NULL,        -- operadoras.registro_ans (validado no serviço)
  numero                 text,                 -- "2025/0412"
  vigencia_inicio        date NOT NULL,
  vigencia_fim           date,                 -- NULL = indeterminado
  versao_tiss            text NOT NULL,        -- '4.03.00' (ADR-PEP-0012: configuração por operadora)
  canal_envio            text NOT NULL DEFAULT 'portal',   -- portal | webservice
  prazo_envio_dias       smallint NOT NULL,    -- após o atendimento
  prazo_resposta_autorizacao_dias_uteis smallint,
  validade_senha_dias    smallint,
  prazo_recurso_dias     smallint,             -- após o demonstrativo (RFC-PEP-007, Q5 de lá)
  limite_texto_recurso   smallint NOT NULL DEFAULT 500,   -- [validar por operadora]
  retorno_sem_cobranca_dias smallint,          -- consulta de retorno não cobrável (RFC-PEP-009 Q3) [validar]
  documento_objeto       text, documento_hash text,       -- PDF cifrado no armazenamento de objetos
  ativo                  boolean NOT NULL DEFAULT true,
  criado_por             uuid NOT NULL,
  criado_em              timestamptz NOT NULL DEFAULT now(),
  atualizado_em          timestamptz NOT NULL DEFAULT now()
);

-- unidades abrangidas pelo contrato (um contrato do CNPJ costuma valer para várias unidades)
CREATE TABLE contrato_unidades (
  tenant_id   uuid NOT NULL,
  contrato_id uuid NOT NULL REFERENCES contratos(id),
  unidade_id  uuid NOT NULL,
  PRIMARY KEY (contrato_id, unidade_id)
);
-- sem sobreposição de vigência para (unidade, registro_ans): checado no serviço sob pg_advisory_xact_lock(tenant, registro_ans)

CREATE TABLE contrato_reajustes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  contrato_id uuid NOT NULL REFERENCES contratos(id),
  percentual numeric(6,3) NOT NULL,           -- 6.500 = +6,5%
  inicio date NOT NULL,
  tabela_gerada_id uuid,                      -- versão criada a partir dele
  observacao text, criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);

-- lado CONTRATO: versão imutável depois de publicada
CREATE TABLE tabelas_negociadas (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  contrato_id      uuid NOT NULL REFERENCES contratos(id),
  versao           int NOT NULL,
  vigencia_inicio  date NOT NULL,             -- vigência_fim = início da próxima versão publicada
  origem           text NOT NULL,             -- importacao | manual | reajuste
  status           text NOT NULL DEFAULT 'rascunho',   -- rascunho | publicada
  arquivo_objeto   text,                      -- CSV importado, para auditoria
  qtd_itens        int,
  publicada_por uuid, publicada_em timestamptz,
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (contrato_id, versao)
);
CREATE UNIQUE INDEX ux_tabela_vigencia ON tabelas_negociadas (contrato_id, vigencia_inicio) WHERE status = 'publicada';

CREATE TABLE tabela_negociada_itens (
  tenant_id        uuid NOT NULL,
  tabela_id        uuid NOT NULL REFERENCES tabelas_negociadas(id),
  codigo_tuss      text NOT NULL,
  descricao        text,
  valor_centavos   bigint,                    -- NULL = coberto, sem valor próprio (ex.: dentro de pacote)
  limite_por_guia  smallint,
  exige_autorizacao boolean NOT NULL DEFAULT false,   -- o que o CONTRATO diz
  observacoes      text,
  PRIMARY KEY (tabela_id, codigo_tuss)
);

CREATE TABLE pacotes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  tabela_id uuid NOT NULL REFERENCES tabelas_negociadas(id),   -- pacote acompanha a versão da tabela
  codigo text NOT NULL,                        -- código do pacote na operadora [validar se é TUSS ou próprio]
  descricao text NOT NULL,
  valor_centavos bigint NOT NULL,
  UNIQUE (tabela_id, codigo)
);
CREATE TABLE pacote_itens (
  tenant_id uuid NOT NULL,
  pacote_id uuid NOT NULL REFERENCES pacotes(id),
  codigo_tuss text NOT NULL,
  quantidade_max smallint,
  PRIMARY KEY (pacote_id, codigo_tuss)
);

-- lado SISTEMA: append-only por vigência (novo preço fecha o anterior)
CREATE TABLE precos_faturamento (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  unidade_id       uuid NOT NULL,
  registro_ans     text NOT NULL,
  codigo_tuss      text NOT NULL,
  valor_centavos   bigint NOT NULL CHECK (valor_centavos >= 0),
  vigencia_inicio  date NOT NULL,
  vigencia_fim     date,                      -- preenchido quando outro preço entra; única atualização permitida
  origem           text NOT NULL,             -- manual | aplicado_do_contrato | importacao
  tabela_origem_id uuid,                      -- quando aplicado do contrato
  criado_por       uuid NOT NULL,
  criado_em        timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_preco_aberto ON precos_faturamento (unidade_id, registro_ans, codigo_tuss) WHERE vigencia_fim IS NULL;
CREATE INDEX ix_preco_busca ON precos_faturamento (unidade_id, registro_ans, codigo_tuss, vigencia_inicio DESC);

-- lado SISTEMA: tipos fechados; editar = fechar a vigência e inserir nova linha
CREATE TABLE regras_autorizacao (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  contrato_id      uuid NOT NULL REFERENCES contratos(id),
  nome             text NOT NULL,             -- rótulo da tela: "Ressonância magnética"
  codigos_tuss     text[] NOT NULL CHECK (cardinality(codigos_tuss) > 0),
  tipo             text NOT NULL,             -- sempre | eletivo | acima_de_quantidade | nao_exige
  parametros       jsonb NOT NULL DEFAULT '{}',   -- acima_de_quantidade: { "quantidade": 10, "escopo": "guia" }
  vigencia_inicio  date NOT NULL,
  vigencia_fim     date,
  criado_por       uuid NOT NULL,
  criado_em        timestamptz NOT NULL DEFAULT now(),
  CHECK (tipo IN ('sempre','eletivo','acima_de_quantidade','nao_exige'))
);
CREATE INDEX ix_regras_tuss ON regras_autorizacao USING gin (codigos_tuss);

CREATE TABLE auditorias_contrato (
  id                     uuid PRIMARY KEY,
  tenant_id              uuid NOT NULL,
  contrato_id            uuid NOT NULL REFERENCES contratos(id),
  unidade_id             uuid NOT NULL,       -- uma auditoria por unidade abrangida (clinica_ref da Spike = unidade)
  tabela_id              uuid NOT NULL,       -- versão vigente auditada
  gatilho                text NOT NULL,       -- publicacao | regras | precos | manual | semanal
  estado                 text NOT NULL,       -- na_fila | concluida | falhou
  tissia_ai_estado           text NOT NULL DEFAULT 'pendente',  -- pendente | enviado | sem_mudanca | falhou | nao_aplicavel
  tissia_ai_payload_hash     text,                -- SHA-256 do conteúdo enviado; igual ao último enviado → sem_mudanca
  total_itens            int,
  qtd_divergencias       int,
  impacto_mensal_centavos bigint,
  regras_criadas_tissia_ai   int,
  iniciada_em timestamptz NOT NULL DEFAULT now(), concluida_em timestamptz
);
CREATE INDEX ix_auditoria_ultima ON auditorias_contrato (contrato_id, unidade_id, iniciada_em DESC);

CREATE TABLE divergencias_contrato (
  id                 uuid PRIMARY KEY,
  tenant_id          uuid NOT NULL,
  auditoria_id       uuid NOT NULL REFERENCES auditorias_contrato(id),
  codigo_tuss        text NOT NULL,
  tipo               text NOT NULL,           -- ver tabela de tipos
  fonte              text NOT NULL,           -- local | spike | ambas
  contrato_centavos  bigint, sistema_centavos bigint,
  volume_90d         int,                     -- quantidade executada nas guias da unidade × operadora
  impacto_mensal_centavos bigint,             -- NULL quando não é perda certa (autorização, limite)
  detalhe            jsonb,                   -- ex.: { "guias": 4, "limite": 10 } ou { "pacote": "…" }
  UNIQUE (auditoria_id, codigo_tuss, tipo)
);
```

`# ponytail:` divergência é refeita a cada auditoria (não há estado "resolvida" a manter); a tela mostra sempre a última auditoria e o histórico guarda as anteriores.

**Antecipação para a P1 (fatia 0):** a RFC-PEP-006 já depende de "cadastro mínimo de operadoras/contratos (versão TISS, prazo de envio)". `operadoras`, `contratos` e `contrato_unidades` com o cabeçalho podem entrar na P1 sem o resto. Isso também resolve o desvio do Marco D em que `beneficiarios` e `guias` guardam só `operadora_nome`: a RFC-PEP-002/006 passa a guardar `registro_ans` validado contra `operadoras` (migração aditiva; o seed da demo usa registros sintéticos marcados como tais).

### Tipos de divergência

Os quatro primeiros são o vocabulário da Spike ([[RFC-IA-008-preditivo-historico-e-contratos|Spike RFC-008 §4]]); os locais complementam o que a Spike não conhece (limite e pacote).

| `tipo` | Regra (para a versão vigente da tabela e o preço/regra vigente hoje) | Rótulo na tela 21 | Impacto mensal |
| :--- | :--- | :--- | :--- |
| `valor_abaixo_do_contrato` | sistema < contrato | "Reajuste não aplicado" se sistema = valor da versão **anterior** da tabela; senão "Sistema abaixo do contrato" | (contrato − sistema) × volume_90d / 3 |
| `valor_acima_do_contrato` | sistema > contrato | "Sistema acima do contrato" | (sistema − contrato) × volume_90d / 3 (glosa provável) |
| `procedimento_ausente_no_pep` | item com valor no contrato e sem preço de faturamento | "Sem preço no sistema" | — |
| `autorizacao_previa_nao_configurada` | `exige_autorizacao` no contrato e nenhuma regra (≠ `nao_exige`) cobre o TUSS | "Autorização não parametrizada" | — (risco, não perda certa) |
| `cobrado_fora_do_pacote` (local) | nos últimos 90 dias, guia com o código do pacote cobrou também item incluso | "Cobrado fora do pacote" | valor do item × ocorrências / 3 |
| `limite_excedido` (local) | nos últimos 90 dias, guia com quantidade > `limite_por_guia` | "Limite: N por guia" | — |

`volume_90d` vem do faturamento (`FaturamentoService.volumePorTuss(unidade, registro_ans, desde)`, soma das quantidades das guias criadas). Impacto da auditoria = soma dos impactos não nulos.

### Auditoria

Job `contrato-auditar` (BullMQ), por contrato × unidade abrangida:

1. **Local, sempre:** carrega a versão vigente da tabela, os preços e regras vigentes e compara com as funções puras `compararContratoSistema` e `exigeAutorizacao` de `@pep/contracts`. Grava a auditoria `concluida` e as divergências `fonte = local`. Não depende de rede.
2. **Spike, quando aplicável** (fatia 6; unidade com modo ≠ `desligado`): monta `itens_contrato` (TUSS, `valor_contratado` em reais, `exige_autorizacao_previa` do contrato, `observacoes`) e `itens_pep` (TUSS, `valor_configurado`, `exige_autorizacao_previa` = existe regra ≠ `nao_exige`). Se o hash do conteúdo é igual ao do último envio bem-sucedido → `tissia_ai_estado = sem_mudanca` e não chama. Senão chama `POST /v1/contratos/auditar` pelo SpikeAdapter, em fatias de até 2.000 itens ordenadas por TUSS, com `clinica_ref = "un_" + unidade_id` e `registro_ans`. Divergências da Spike são mescladas por (`codigo_tuss`, `tipo`): presente nas duas → `fonte = ambas`; só na Spike → `fonte = spike` e métrica `contratos.divergencia_pep_tissia_ai` (sinal de bug de um dos lados).
3. Falha, timeout ou 422 da Spike → `tissia_ai_estado = falhou`, retry com backoff (8 tentativas/24 h); a tela já mostra o resultado local com "Auditoria da Spike indisponível".

`provisionar_regras_autorizacao` vai **`false`** no PEP. Motivo, verificado no código da Spike: a provisão só **acrescenta** TUSS à regra `CONTRATO-AUT-<registro_ans>` (nunca remove) e grava novas linhas de `contratos_operadora` a cada chamada. Uma autorização que deixou de ser exigida continuaria alertando na análise para sempre, e as regras condicionais do PEP (`eletivo`, `acima_de_quantidade`) não cabem num booleano. As regras locais do PEP já geram o alerta de autorização na guia (RFC-PEP-006). Revisitar com a proposta de SPK da Q3.

Gatilhos: evento `ContratoAtualizado` (publicação de tabela, mudança de regras ou preços), "Reauditar" e agendamento semanal (BullMQ repeatable, segunda-feira 06:00 `America/Sao_Paulo`, por tenant). O semanal existe porque o volume de 90 dias e os pacotes/limites mudam com as guias, não porque a Spike perca a tabela.

### Importação de tabela

1. `POST /contratos/{id}/tabelas/importacoes` recebe CSV UTF-8 (`;` ou `,`), colunas `codigo_tuss;descricao;valor;limite_por_guia;exige_autorizacao;observacoes`. Parser próprio, sem dependência nova (`# ponytail:` XLSX só quando a Q1 confirmar o formato; aí entra uma biblioteca de leitura de planilha).
2. Valida cada linha: TUSS no formato da tabela de referência (contra a tabela TUSS quando existir no PEP; até lá, 8 dígitos **[validar]**), valor ≥ 0 com vírgula ou ponto, duplicados. Até 20.000 linhas.
3. Resposta: rascunho de nova versão + diff contra a vigente: `novos`, `alterados` (antes/depois), `removidos`, `invalidos` (linha, motivo). Nada é publicado.
4. `POST …/importacoes/{id}/confirmar { "vigenciaInicio": "…" }` publica a versão (imutável), fecha a anterior e publica `ContratoAtualizado`. Vigência no passado é permitida (reajuste retroativo) com aviso de que retornos já importados **não** são reconciliados sozinhos.

Reajuste (`POST /contratos/{id}/reajustes`) gera rascunho de versão com todos os valores × (1 + percentual), arredondamento meio-par para o centavo, a revisar e publicar pelo mesmo fluxo.

### Aplicar valores do contrato

`POST /contratos/{id}/aplicar-valores { "unidadeId": "…", "codigosTuss": [...] }`: para cada TUSS com divergência de valor ou sem preço, fecha o preço aberto em `precos_faturamento` e insere o valor da versão vigente da tabela com `origem = aplicado_do_contrato`, vigência a partir de hoje. Guias ainda não enviadas com o preço antigo não são alteradas: a validação "Valores × contrato" da RFC-PEP-006 passa a apontá-las na mesa ("revalidar"). Publica `ContratoAtualizado` (gatilho `precos`) → nova auditoria.

### Autorização prévia

Função pura em `@pep/contracts`:

```ts
exigeAutorizacao(regras: RegraAutorizacao[], c: { codigoTuss: string; carater: 'eletivo' | 'urgencia'; quantidade: number; data: string })
  → { exige: boolean; regraId: string | null; motivo: string | null }
```

- Considera só regras vigentes na `data` cujo `codigos_tuss` contém o TUSS. `nao_exige` é exceção explícita e vence as demais; entre as outras, basta uma exigir.
- `eletivo` exige só com `carater = eletivo`; `acima_de_quantidade` exige quando `quantidade > parametros.quantidade` (escopo `guia` na P2; por período é a Q7).
- Consumidores: agenda (RFC-PEP-003: ao agendar com `procedimento_tuss`, preenche `exige_autorizacao`/`autorizacao_status = pendente`; carater padrão `eletivo`), guia (RFC-PEP-006: validação "Autorização" sem senha), mesa do faturista. Pelo serviço público `ContratosService.regrasAutorizacao(unidadeId, registroAns, data)`, nunca por HTTP entre módulos.
- `ContratoAtualizado` com gatilho `regras` → a agenda recalcula o selo dos agendamentos futuros da unidade × operadora.

### Serviço público para outros módulos

| Método | Retorno | Usado por |
| :--- | :--- | :--- |
| `precoFaturamento(unidadeId, registroAns, codigoTuss, data)` | centavos ou `null` | RFC-PEP-006 (valor da guia; substitui a tabela sintética), RFC-PEP-009 (`valor_estimado`) |
| `precoContratado(unidadeId, registroAns, codigoTuss, data)` | `{ valorCentavos, tabelaId, versao, valorVersaoAnteriorCentavos, limitePorGuia }` ou `null` | RFC-PEP-007 (conciliação e dica de reajuste), RFC-PEP-006 (limite) |
| `parametros(unidadeId, registroAns, data)` | `{ contratoId, versaoTiss, canalEnvio, prazoEnvioDias, prazoRecursoDias, limiteTextoRecurso, validadeSenhaDias, prazoRespostaAutorizacaoDiasUteis, retornoSemCobrancaDias }` ou `null` | RFC-PEP-006 (prazo, versão do lote), RFC-PEP-007 (prazo e limite do recurso), RFC-PEP-009 |
| `regrasAutorizacao(unidadeId, registroAns, data)` | `RegraAutorizacao[]` | RFC-PEP-003, RFC-PEP-006 |

Sem contrato vigente, cada método devolve `null` e o consumidor usa o seu padrão sinalizado (ex.: "sem contrato cadastrado").

### API (módulo `contratos`)

Corpos em camelCase, como a API implementada. Permissões novas: `contratos:ler` (faturista, gestor, admin, recepção para ver regras de autorização) e `contratos:escrever` (gestor e admin; faturista importa e aplica valores, mas não cria nem edita contrato — Q5 confirma).

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/operadoras?q=` | Busca no catálogo global por nome ou registro ANS |
| `GET` | `/api/v1/contratos?unidade=` | Lista da tela (operadora, vigência, "renovação em N dias", divergências abertas da última auditoria) |
| `POST` | `/api/v1/contratos` | `{ "registroAns": "<REGISTRO_ANS>", "numero": "2025/0412", "unidadeIds": ["…"], "vigenciaInicio": "2025-09-01", "vigenciaFim": "2027-08-31", "versaoTiss": "4.01.00", "canalEnvio": "portal", "prazoEnvioDias": 30, "prazoRespostaAutorizacaoDiasUteis": 5, "validadeSenhaDias": 30, "prazoRecursoDias": 30, "limiteTextoRecurso": 500 }` → 409 se sobrepõe vigência na mesma unidade × operadora |
| `GET` | `/api/v1/contratos/{id}` | Cabeçalho, unidades, último reajuste, versão vigente da tabela (`qtdItens`) |
| `PATCH` | `/api/v1/contratos/{id}` | Cabeçalho e prazos (diff na trilha de auditoria; vale para o que for calculado depois) |
| `PUT` / `GET` | `/api/v1/contratos/{id}/documento` | Upload do PDF (≤ 20 MB) / download (auditado) |
| `GET` | `/api/v1/contratos/{id}/tabelas` | Versões (aba Histórico) |
| `GET` | `/api/v1/contratos/{id}/tabelas/{versao}/itens?q=&cursor=` | Aba Tabela negociada |
| `POST` | `/api/v1/contratos/{id}/tabelas/importacoes` | Multipart CSV → `{ "importacaoId": "…", "novos": 12, "alterados": [{ "codigoTuss": "<TUSS_…>", "antes": 13615, "depois": 14500 }], "removidos": 0, "invalidos": [{ "linha": 41, "motivo": "valor inválido" }] }` |
| `POST` | `/api/v1/contratos/{id}/tabelas/importacoes/{importacaoId}/confirmar` | `{ "vigenciaInicio": "2026-09-01" }` → publica |
| `POST` | `/api/v1/contratos/{id}/reajustes` | `{ "percentual": 6.5, "inicio": "2026-09-01", "observacao": "…" }` → rascunho de versão |
| `GET` / `PUT` | `/api/v1/contratos/{id}/pacotes` | Pacotes da versão vigente / substitui o conjunto do rascunho |
| `GET` / `PUT` | `/api/v1/contratos/{id}/regras-autorizacao` | Lista vigente / substitui o conjunto: `[{ "nome": "Ressonância magnética", "codigosTuss": ["<TUSS_…>"], "tipo": "sempre" }, { "nome": "Fisioterapia acima de 10 sessões", "codigosTuss": ["<TUSS_…>"], "tipo": "acima_de_quantidade", "parametros": { "quantidade": 10, "escopo": "guia" } }]` |
| `POST` | `/api/v1/contratos/{id}/auditorias` | Reauditar → `202 { "auditoriaIds": [...] }` |
| `GET` | `/api/v1/contratos/{id}/auditorias/ultima?unidade=` | Resultado, impacto, `spikeEstado`, divergências |
| `GET` | `/api/v1/contratos/{id}/auditorias/{auditoriaId}/divergencias.csv` | Exportar divergências |
| `POST` | `/api/v1/contratos/{id}/aplicar-valores` | `{ "unidadeId": "…", "codigosTuss": ["…"] }` |
| `GET` | `/api/v1/precos-faturamento?unidade=&operadora=&tuss=` | Histórico do lado sistema |

Erros RFC 9457 com `code` estável (`vigencia_sobreposta`, `operadora_desconhecida`, `tabela_publicada_imutavel`, `csv_invalido`, `regra_invalida`). Schemas Zod `.strict()` em `@pep/contracts/contratos.ts`, com `parametros` de regra como união discriminada por `tipo`.

### Eventos

- **Publica:** `ContratoAtualizado` (auxiliar: `contrato_id`, `registro_ans`, `unidade_ids`, `gatilho` = `cabecalho | publicacao | regras | precos`) e `AuditoriaContratoConcluida` (`auditoria_id`, `contrato_id`, `unidade_id`, `qtd_divergencias`, `impacto_mensal_centavos`). Só ids e números.
- **Consome:** `ContratoAtualizado` (próprio) → `contrato-auditar`; agendamento semanal.
- **Consumidores externos:** agenda (`ContratoAtualizado` com `regras` → recalcula selos), faturamento (`ContratoAtualizado` com `precos`/`regras` → revalida guias não enviadas da unidade × operadora), bi (`AuditoriaContratoConcluida`, RFC-PEP-012).

### Regras

- Tabela publicada nunca muda; correção = nova versão. A conciliação da RFC-PEP-007 usa a versão vigente na data de execução.
- Preço de faturamento e regra de autorização nunca são sobrescritos: fecha a vigência e insere.
- Reajuste retroativo não reconcilia retornos já importados (`# ponytail:` reconciliar sob demanda numa RFC futura se o piloto pedir).
- Sem contrato vigente, guia, agenda e retorno seguem com padrão sinalizado; nada bloqueia por falta de contrato.
- Falha da Spike nunca impede salvar, importar, aplicar ou ver a auditoria local.
- Não inventar TUSS em exemplo, teste ou seed: `<TUSS_…>` e tabelas sintéticas marcadas.
- Módulo atrás da flag `modulos.contratos` por clínica (release plan), exceto a fatia 0 (necessária ao faturamento).

## UI

Rota canônica `/contratos` (RFC-PEP-015), `?contrato=` e `?aba=` em query string (`nuqs`). Tela `apps/ui/src/modulos/faturamento/Contratos.tsx`; ao ligar na API sai o selo "Dados ilustrativos" e o rodapé "Códigos e valores ilustrativos".

| Bloco | Fonte | Observação |
| :--- | :--- | :--- |
| "Importar tabela (CSV ou XLSX)" | `POST …/tabelas/importacoes` → diálogo de prévia → confirmar | Aceita só CSV até a Q1; o rótulo vira "Importar tabela (CSV)" |
| "Novo contrato" | `GET /operadoras`, `POST /contratos` | Diálogo com cabeçalho e prazos |
| Lista "Operadoras (N)" com vigência e badge de divergências | `GET /contratos?unidade=` | Badge = `qtdDivergencias` da última auditoria; "Renovação em N dias" quando `vigenciaFim` ≤ 60 dias |
| Cabeçalho "Operadora · contrato N" + "Ver PDF do contrato" | `GET /contratos/{id}`, `GET …/documento` | Sem PDF: botão "Anexar PDF" |
| Grade Vigência / Versão TISS · canal / Prazo de envio / Reajuste | `GET /contratos/{id}` | "webservice" só aparece com a ADR-PEP-0012 aceita e parceiro ativo; antes, "portal" |
| Abas Auditoria de valores · Tabela negociada (N itens) · Pacotes · Histórico | `…/auditorias/ultima`, `…/tabelas/{versao}/itens`, `…/pacotes`, `…/tabelas` + reajustes | Histórico lista versões, reajustes e auditorias |
| Aviso "Auditoria de DD/MM HH:MM: N divergências… Impacto estimado R$ X/mês" + "Reauditar" | `…/auditorias/ultima`, `POST …/auditorias` | Com `spikeEstado = falhou`: sufixo "auditoria da Spike indisponível; resultado local" |
| Tabela TUSS · Procedimento · Contrato · Sistema · Diferença · Situação | divergências da última auditoria | Rótulos da tabela de tipos; "Pacote" na coluna Contrato quando o item está em pacote |
| "Aplicar valores do contrato (N itens)" | `POST …/aplicar-valores` com os TUSS de divergência de valor ou sem preço | Confirmação com a lista; desabilitado com N = 0 |
| "Exportar divergências" | `GET …/divergencias.csv` | |
| Bloco "Autorização prévia" + "Editar regras" | `GET/PUT …/regras-autorizacao` | Badge: `sempre` → "Exige", `eletivo` → "Só eletivo", `acima_de_quantidade` → "Exige" com o nome descrevendo o limite, `nao_exige` → "Não exige" |
| Prazos (resposta, validade da senha, recurso) | `GET /contratos/{id}` | Texto montado a partir dos campos |

## Segurança e LGPD

- Contrato e tabelas não têm dado pessoal. O PDF do contrato é sigilo comercial: cifrado no armazenamento de objetos, download auditado, acesso por `contratos:ler`.
- Para a Spike vão só `clinica_ref` (unidade), `registro_ans`, TUSS, valores, flags e observações. Observações são texto livre digitado pela clínica: a tela avisa "não inclua dados de pacientes" e o envio passa pelo limite de tamanho do contrato da Spike.
- Toda criação/edição de contrato, publicação de tabela, aplicação de valores e mudança de regra vai para a trilha de auditoria (RFC-PEP-014) com autor e diff (valores não são PII).
- Logs: ids, contagens, estado e latência; nunca o CSV nem o corpo enviado à Spike.
- Retenção: vigência + 5 anos **[validar]** (modelo de dados).

## Critérios de aceite

- [ ] RLS: tenant A não lê contrato, tabela, preço, regra ou auditoria do tenant B (teste com dois tenants); `operadoras` é legível por todos e não gravável pela aplicação.
- [ ] `UPDATE` em item de tabela publicada falha (`tabela_publicada_imutavel`); novo preço fecha o anterior (índice parcial impede dois preços abertos).
- [ ] Contratos com vigências sobrepostas para a mesma unidade × operadora → 409, inclusive sob concorrência (10 transações paralelas).
- [ ] `precoContratado` devolve a versão certa para datas antes, no dia e depois do início de uma nova versão.
- [ ] Importação de CSV de 5.000 linhas com prévia em ≤ 5 s; linhas inválidas listadas; nada publicado sem confirmar.
- [ ] Reajuste de 6,5% gera versão com arredondamento meio-par testado por propriedade (nenhum item difere mais de 1 centavo do cálculo exato).
- [ ] `exigeAutorizacao`: teste de tabela para os 4 tipos, exceção `nao_exige`, vigência e carater.
- [ ] Auditoria local detecta os 6 tipos em dados sintéticos; "Reajuste não aplicado" só quando o sistema tem o valor da versão anterior; impacto mensal calculado sobre o volume de 90 dias.
- [ ] Spike fora (stub com timeout e 5xx): auditoria `concluida` com resultado local, `tissia_ai_estado = falhou`; nada bloqueado.
- [ ] Payload para `/v1/contratos/auditar` com 4.500 itens sai em 3 chamadas de até 2.000; `provisionar_regras_autorizacao = false`; mesmo conteúdo na semana seguinte → `sem_mudanca`, sem chamada.
- [ ] "Aplicar valores do contrato" cria preços com `origem = aplicado_do_contrato` e dispara reauditoria sem as divergências aplicadas.
- [ ] Agenda e guia passam a usar `regrasAutorizacao` (teste de integração: regra `sempre` → agendamento com `autorizacao_status = pendente`).
- [ ] RFC-PEP-006 usa `precoFaturamento` e a `TABELA_PRECOS_SINTETICA_CENTAVOS` sai do caminho de produção (fica só no seed da demo).
- [ ] Tela 21 ligada à API.

## Riscos

- Contratos em PDF heterogêneos: onboarding manual por planilha; medir horas por clínica no piloto.
- Tabelas com milhares de itens (materiais, medicamentos) aumentam o volume da auditoria; fatiamento já previsto, SPK-03 para auditoria assíncrona na Spike.
- Duas fontes de regra de autorização (PEP e, se a provisão for ligada, Spike) podem divergir; por isso a provisão fica desligada.
- Reajuste retroativo muda a conciliação de retornos já importados; sinalizado, não automático.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | As clínicas do piloto têm tabelas exportáveis do sistema atual? Formato (CSV, XLSX, PDF)? | Produto | Só o XLSX |
| Q2 | Quais tabelas TUSS entram (procedimentos; diárias e taxas; materiais; medicamentos) **[validar]** e a carga da tabela TUSS de referência no PEP | Consultor de faturamento | Validação de TUSS na importação (até lá, só formato) |
| Q3 | Proposta de SPK a abrir: `contratos/auditar` com semântica de **substituição** da tabela por `clinica_ref` × `registro_ans` (hoje acumula linhas a cada chamada) e remoção de TUSS da regra provisionada; só então ligar `provisionar_regras_autorizacao` | Pedro | Não |
| Q4 | Confirmar `provisionar_regras_autorizacao = false` no PEP e, com isso, o valor do envio à Spike (RF-CON-03) para o PEP: histórico de contrato na Spike e checagem cruzada | Pedro | Não (fatia 6 é a última) |
| Q5 | Contrato por CNPJ abrangendo várias unidades (desenho atual) ou sempre um por unidade? Faturista pode criar/editar contrato ou só importar e aplicar valores? | Produto + gestor piloto | Não (padrão: várias unidades; edição por gestor/admin) |
| Q6 | ~~Fonte do catálogo `operadoras`~~ **Resolvida em 13/11/2026:** CADOP, dados abertos da ANS (CC BY), carga por script com manifesto ([[ADR-PEP-0024-dados-de-referencia-ans\|ADR-PEP-0024]]). Falta a frequência de publicação (mensal?) **[validar]** | Produto + engenharia | **Sim, fatia 0** |
| Q7 | Regra "acima de N sessões": por guia, por mês ou por ano? Conta guias de outros profissionais? | Consultor de faturamento | Não (padrão: por guia) |
| Q8 | Código de pacote: TUSS ou código próprio da operadora? Como aparece no XML do lote **[validar]** | Consultor de faturamento | Fatia 5 |
| Q9 | Dias de retorno sem cobrança por contrato (RFC-PEP-009 Q3): valor típico e se varia por especialidade **[validar]** | Consultor de faturamento | Não (campo opcional) |

## Fatias de entrega

0. **Cadastro mínimo (pode entrar na P1, pré-requisito da RFC-PEP-006):** `operadoras` (Q6), `contratos` com prazos, versão TISS e canal, `contrato_unidades`, `parametros()`; `registro_ans` validado em beneficiário e guia; lista e cabeçalho da tela 21.
1. **Tabela negociada (RF-CON-01):** versões, importação CSV com prévia, reajuste, PDF do contrato, `precoContratado()`; abas Tabela negociada e Histórico.
2. **Preços de faturamento (RF-CON-01):** `precos_faturamento`, `precoFaturamento()`, "Aplicar valores do contrato"; faturamento troca a tabela sintética.
3. **Autorização prévia (RF-CON-02):** regras em tipos fechados, `exigeAutorizacao` em `@pep/contracts`, consumo pela agenda e pela guia; bloco da tela 21.
4. **Auditoria local (RF-CON-03):** job, gatilhos, semanal, 4 tipos de valor/autorização, impacto, exportação; aba Auditoria e aviso.
5. **Pacotes e limite (RF-CON-01):** pacotes por versão, `cobrado_fora_do_pacote` e `limite_excedido`; aba Pacotes.
6. **Spike (RF-CON-03):** `contratos/auditar` pelo SpikeAdapter, fatiamento, hash, mescla de divergências, `provisionar_regras_autorizacao = false`.

- **Telas do canvas de 13/11/2026 já na UI como prévia:** 21b (novo contrato, etapa 1: operadora escolhida no CADOP, versão TISS 4.03.00 com exceção) em `/contratos/novo`; tela 21 com a operadora "inativa na ANS", a exceção em 4.01.00 e os códigos TUSS como placeholders. O cadastro real é a fatia 0 desta RFC.

## Notas de implementação

Implementadas as fatias 1–6 em 15/11/2026 (`ContratosModule`, `@pep/contracts`, migrations `0043_contratos_completo.sql` e `0044_contratos_imutabilidade.sql`, integração com `AgendaModule`, `FaturamentoModule`, `ProntuarioModule`, `TissiaAiModule`, worker BullMQ semanal e telas 21 / 21b em `@pep/ui`), além da proposta [[SPK-11-contratos-auditar-substituicao|SPK-11]].

Desvios registrados:
- **Upload de CSV e PDF sem parser `multipart`:** `POST /v1/contratos/:id/tabelas/importacoes` e `PUT /v1/contratos/:id/documento` aceitam payload JSON (`{ conteudo }` e `{ documentoBase64 }` ou corpo binário raw), mantendo o comportamento funcional sem adicionar dependência de upload multipart.
- **Armazenamento do PDF do contrato:** o PDF é cifrado em repouso com AES-256-GCM via `CriptoService` na coluna `contratos.documento_cifrado` (PostgreSQL) em vez de bucket S3/GCS externo.
- **Execução da auditoria:** `POST /v1/contratos/:id/auditorias` executa a auditoria local + `TissiaAiService.auditarContratos` imediatamente (além de enfileirar evento no outbox e contar com o agendador semanal `0 6 * * 1` em `America/Sao_Paulo`), mantendo `provisionar_regras_autorizacao: false`, fatiamento de 2.000 itens, hash SHA-256 (`sem_mudanca`) e fallback local quando a Spike falha.
- **Permissões (Q5):** leitura com `contratos:ler`; edição de contrato e regras com `contratos:escrever` (gestor/admin); importação/reajuste de tabela, `aplicar-valores` e disparo de auditoria aceitam também `guias:escrever` (faturista).
- **Fora do escopo:** importação `.xlsx` (apenas CSV, conforme Q1), busca/criação de pacotes via modal específico na UI (CRUD de pacotes entregue na API e listagem na aba Pacotes) e suíte E2E dedicada (cobertura feita por testes unitários em `@pep/contracts` e de integração em `apps/api/test/contratos.spec.ts`).
- **Propagação de `registroAns`:** `ProntuarioService` passou a enviar `registroAns` em `NovaGuiaRascunho` ao fechar atendimento, permitindo a precificação real por `ContratosService.precoFaturamento` em `FaturamentoService` sem alterar `guias.dados`.

**Demo de dados fictícios (19/11/2026, PR #30 do `tissia`, em rascunho).** Ver [[demo-dados-ficticios]] e [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026]].

- **Preço na data do lote:** a regra de qual data precifica a guia (dataAtendimento na consulta, dataSolicitacao na SP/SADT, dia de criação em Brasília) está na nota da [[RFC-PEP-006-guias-e-lotes-tiss]]; ela casa com a lógica do lote e com "versão vigente na data de execução" desta RFC.
- **Seed `faturamento`:** o módulo é dono do domínio `contratos` do tenant da demo e o recria a cada execução: 4 contratos (operadoras A–D), 128 preços, 1 pacote por operadora e auditoria com divergências. Operadoras ausentes do cadastro são inseridas como "Operadora X (demonstração)" (`versao_ref = 'demo'`), sem tocar linhas reais do CADOP.
- **UI (`Contratos.tsx`):** diferença zero sem sinal (some o "−R$ 0,00"), "Vigência indeterminada" para vigência `null`, aba "Tabela negociada" com rolagem horizontal. Em `VITE_AMBIENTE_DEMO=true`, "Novo contrato" busca só as 4 operadoras da demo (por rótulo A–D ou registro), sem enumerar o CADOP real (`apps/ui/src/modulos/faturamento/operadoraDemo.ts`).
- **Catálogo TUSS da demo:** `CATALOGO_TUSS_DEMO` e `TABELA_PRECOS_SINTETICA_CENTAVOS` (`packages/contracts`, só aditivo) foram alinhados à Tabela 22 oficial (19 códigos). O mesmo código passou a ter outro termo para 11 entradas, e a tabela negociada da demo passou a 19 itens.
