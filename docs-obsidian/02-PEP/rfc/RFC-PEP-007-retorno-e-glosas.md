---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-007-retorno-e-glosas.md
---

# RFC-PEP-007 — Retorno da operadora, conciliação e central de glosas

**Fase:** P2 · **Status:** Em implementação (fatias 0–5 e infraestrutura da fatia 6 entregues; ADRs 0006, 0012 e 0014 aceitas; pendente integração de recurso assistido via [[SPK-02-recurso-de-glosa|SPK-02]], [[SPK-12-desfecho-recurso-manual|SPK-12]] e [[SPK-13-ingestao-mapa-guia-ref|SPK-13]], ver Notas de implementação) · **ADRs:** ADR-PEP-0003, ADR-PEP-0005, ADR-PEP-0006, ADR-PEP-0008, ADR-PEP-0012, ADR-PEP-0014, ADR-PEP-0017
**Telas:** 19 (Retorno e demonstrativo), 20 (Central de glosas e recurso)
**Requisitos:** RF-RET-01 a RF-RET-05
**Depende de:** RFC-PEP-006 completa (lotes, `numero_prestador`, guia `enviada`, `LoteEnviado`); RFC-PEP-008 (preço contratado vigente, prazo e limite de texto do recurso; sem ela, fallback abaixo); RFC-PEP-005 (SpikeAdapter, fila); RFC-PEP-000 (relay do outbox com fan-out por consumidor, fatia 0); RFC-PEP-010 (consome a baixa; não bloqueia esta RFC); RFC-PEP-011 (evolução em PDF assinado como anexo; sem ela, evolução fechada com hash); Spike [[SPK-02-recurso-de-glosa|SPK-02]] (só a fatia 6) e [[SPK-03-processamento-assincrono|SPK-03]] (desejável)

## Objetivo

Fechar o ciclo da guia: importar o demonstrativo da operadora, conciliar guia a guia contra o que foi faturado e contratado, transformar cada glosa em item de trabalho com prazo e montar o recurso com evidências literais da evolução. Cada desfecho volta para a Spike, que aprende quais guias são glosadas em cada operadora. Nada disso depende da Spike para funcionar: importação, conciliação, central e recurso manual rodam só com o PEP.

## Escopo

**Dentro (P2)**

- [RF-RET-01] Importação do demonstrativo XML (análise de conta e pagamento) na versão TISS 4.03.00 (e nas anteriores que o piloto exigir **[validar]**, [[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]]) por upload (tela 19). Validação XSD, parser próprio (ADR-PEP-0012), vínculo item ↔ guia pelo número da guia no prestador.
- [RF-RET-01] Conciliação guia a guia: pago, glosa total, glosa parcial, pago abaixo/acima do contrato sem código de glosa (divergência), item não vinculado.
- [RF-RET-01] Detecção de padrão em divergências (mesma diferença unitária em N itens; diferença igual a um reajuste do contrato → "provável tabela anterior na operadora") e ações em lote "Contestar" e "Aceitar diferença e baixar".
- [RF-RET-01] Conciliação (baixa) dos itens pagos conforme contrato, com evento para o financeiro (RFC-PEP-010).
- [RF-RET-02] Envio à Spike, por fila: lote de envio (`LoteEnviado`) e demonstrativo para `POST /v1/tiss/lotes/ingerir` sem as tags de identificação do beneficiário; desfechos por guia para `POST /v1/desfechos`.
- [RF-RET-03] Central de glosas: glosa criada automaticamente na importação, com prazo de recurso do contrato; lista ordenada por prazo; filtros por operadora, motivo (Tabela 38 **[validar]**), médico, status e período; KPIs glosado, em recurso, recuperado, prazo em até 5 dias, aceito sem recurso.
- [RF-RET-04] Recurso: texto com limite por contrato (padrão 500 **[validar]**), evidências literais da evolução, anexos, aceite humano registrado, lote de recurso por operadora (pacote para o portal + protocolo), resultado. Recurso assistido por IA só com SPK-02 (fatia 6).
- [RF-RET-04] Taxa de reversão da clínica por operadora × motivo ("4 de 6 recursos revertidos"), calculada no PEP.
- [RF-RET-05] Baixa manual de glosa (informada no portal, fora do XML) com desfecho individual na Spike.

**Fora**

- Envio do recurso e recebimento do demonstrativo por webservice: depende do parceiro TISS (ADR-PEP-0012). Fica a porta `OperadoraGateway.buscarDemonstrativos` e a fatia 7, opcional.
- Demonstrativo em PDF ou planilha: importador alternativo só se o piloto medir fração relevante (ver Riscos).
- Contas a receber, conciliação bancária e repasse: RFC-PEP-010. Esta RFC só publica o fato.
- Painel do gestor (taxa de glosa, glosa evitada): RFC-PEP-012 consome os eventos daqui.
- Recurso em segunda instância **[validar se existe por contrato]**: fora até o piloto pedir.

## Design

### Visão geral

```
upload XML ──▶ objeto cifrado + hash ──▶ job retorno-importar (worker)
                                         ├─ XSD + parse (libxml2-wasm, igual à RFC-PEP-006)
                                         ├─ vínculo item ↔ guia (numero_prestador)
                                         ├─ classificação por item (preço contratado na data de execução)
                                         ├─ desfecho por guia + glosa por item glosado
                                         └─ outbox: RetornoImportado, DesfechoRegistrado (1 por guia)
consumidores: spike.ingerir · spike.desfechos · faturamento.status-guia · (financeiro, bi: RFC-PEP-010/012)
```

O PEP é dono do vínculo guia ↔ `guia_ref`. A Spike nunca recebe dado do beneficiário.

### Modelo de dados (módulo `retorno`)

Todas as tabelas têm `tenant_id uuid NOT NULL` com RLS por convenção em `drizzle/rls` (ADR-PEP-0019) e são acessadas só por `Db.comTenant`. FKs só dentro do módulo; `guia_id`, `lote_id`, `profissional_id` e `contrato_id` são `uuid` validados no serviço (ADR-PEP-0017).

Dinheiro em **centavos inteiros** (`*_centavos bigint`), como `guias.valor_total_centavos` já implementado; o SpikeAdapter converte para reais com 2 casas. Desvio consciente do `numeric(12,2)` do [[modelo-de-dados-pep|modelo de dados]] (registrar lá ao implementar).

```sql
-- Referência global (sem tenant_id, só leitura para a aplicação, como TUSS/CID): carga da tabela oficial
-- Tabela 38: `ref_dominios_tiss` com tabela = '38' (RFC-PEP-021), por versão da terminologia (202609: 470 termos); grupo = prefixo de 2 dígitos
-- (10 elegibilidade, 14 autorização, 17 cobrança, 18 auditoria clínica, 50 estrutura…), agrupamento nosso, não da ANS. agrupamento nosso, não da ANS
  vigencia_inicio date NOT NULL, vigencia_fim date,
  PRIMARY KEY (codigo, versao_ref)
);
-- Orientação de recurso por grupo (quais documentos anexar): texto procedural da Tissia, sem afirmação clínica, versionado em git
-- e exibido como "Orientação Tissia", nunca como norma da ANS. Referência: orientações do projeto `tiss` (Apache-2.0), reescritas.
CREATE TABLE ref_orientacao_recurso (
  grupo     text PRIMARY KEY,
  documentos text[] NOT NULL,              -- ex.: ["evolução clínica", "laudo do exame", "autorização prévia"]
  texto     text NOT NULL
);

CREATE TABLE retornos (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  unidade_id       uuid NOT NULL,
  registro_ans     text NOT NULL,          -- do cabeçalho do XML; precisa casar com um contrato/operadora da unidade
  tipo             text NOT NULL,          -- analise_conta | pagamento
  versao_tiss      text NOT NULL,          -- 4.03.00 (anteriores só se o piloto exigir)
  numero_demonstrativo text,               -- protocolo/número do demonstrativo da operadora
  data_demonstrativo date NOT NULL,        -- base do prazo de recurso
  competencia      char(7),
  origem           text NOT NULL DEFAULT 'upload',   -- upload | webservice (fatia 7)
  arquivo_objeto   text NOT NULL,          -- XML original, cifrado no armazenamento de objetos
  arquivo_hash     text NOT NULL,          -- SHA-256 do arquivo
  xsd_valido       boolean NOT NULL,
  xsd_erros        jsonb,                  -- primeiros 50 erros (linha, mensagem), sem conteúdo do XML
  estado           text NOT NULL,          -- importando | importado | falhou
  qtd_itens int, qtd_vinculados int,
  faturado_centavos bigint, pago_centavos bigint, glosado_centavos bigint,
  qtd_divergencias int,
  tissia_ai_ingestao   text NOT NULL DEFAULT 'pendente',  -- pendente | enviado | incerto | falhou | nao_aplicavel
  tissia_ai_ingerido_em timestamptz,
  importado_por    uuid NOT NULL,
  criado_em        timestamptz NOT NULL DEFAULT now(),
  UNIQUE (unidade_id, arquivo_hash)        -- reimportar o mesmo arquivo devolve o retorno existente
);
CREATE INDEX ix_retornos_unidade ON retornos (unidade_id, criado_em DESC);

CREATE TABLE retorno_itens (
  id                  uuid PRIMARY KEY,
  tenant_id           uuid NOT NULL,
  retorno_id          uuid NOT NULL REFERENCES retornos(id),
  ordem               int NOT NULL,
  numero_guia_prestador text NOT NULL,
  numero_guia_operadora text,
  guia_id             uuid,                -- NULL = não vinculado
  lote_id             uuid,                -- lote da guia vinculada
  codigo_tuss         text NOT NULL,
  data_execucao       date,
  quantidade          numeric(8,2),
  informado_centavos  bigint NOT NULL,     -- valor que a operadora diz ter recebido
  pago_centavos       bigint NOT NULL,
  glosa_centavos      bigint NOT NULL DEFAULT 0,
  codigo_glosa        text,                -- Tabela 38, validado contra a Tabela 38 (`ref_dominios_tiss`) vigente na data do demonstrativo
  motivo_texto        text,                -- texto da operadora, se houver; nunca logado
  contrato_centavos   bigint,              -- preço contratado × quantidade na data de execução (snapshot); NULL sem contrato
  tabela_versao_id    uuid,                -- versão da tabela negociada usada (RFC-PEP-008)
  situacao            text NOT NULL,       -- pago | glosa_total | glosa_parcial | divergente_abaixo | divergente_acima | nao_vinculado
  tratamento          text NOT NULL DEFAULT 'pendente',  -- pendente | conciliado | contestado | diferenca_aceita
  tratado_por uuid, tratado_em timestamptz,
  UNIQUE (retorno_id, ordem)
);
CREATE INDEX ix_retorno_itens_guia ON retorno_itens (guia_id);
CREATE INDEX ix_retorno_itens_situacao ON retorno_itens (retorno_id, situacao);

-- append-only: o desfecho final da guia é o mais recente
CREATE TABLE desfechos (
  id                 uuid PRIMARY KEY,
  tenant_id          uuid NOT NULL,
  guia_id            uuid NOT NULL,
  retorno_id         uuid REFERENCES retornos(id),   -- NULL na baixa manual e no resultado de recurso manual
  recurso_id         uuid,                           -- preenchido quando origem = recurso
  origem             text NOT NULL,                  -- demonstrativo | manual | recurso
  status             text NOT NULL,                  -- pago | glosado_parcial | glosado_total (vocabulário da Spike)
  cobrado_centavos   bigint NOT NULL,
  glosado_centavos   bigint NOT NULL,
  codigo_glosa_principal text,                       -- do item de maior valor glosado
  tissia_ai_estado       text NOT NULL DEFAULT 'pendente',  -- pendente | enviado | nao_encontrado | nao_aplicavel
  tissia_ai_enviado_em   timestamptz,
  registrado_por     uuid NOT NULL,
  criado_em          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_desfechos_guia ON desfechos (guia_id, criado_em DESC);

-- item de trabalho: uma glosa por item glosado (ou por divergência contestada, ou por baixa manual)
CREATE TABLE glosas (
  id                 uuid PRIMARY KEY,
  tenant_id          uuid NOT NULL,
  unidade_id         uuid NOT NULL,
  guia_id            uuid NOT NULL,
  retorno_item_id    uuid UNIQUE REFERENCES retorno_itens(id),  -- NULL na baixa manual
  desfecho_id        uuid NOT NULL REFERENCES desfechos(id),
  registro_ans       text NOT NULL,
  profissional_id    uuid,                 -- espelho da guia, para o filtro por médico
  tipo               text NOT NULL,        -- glosa | divergencia
  codigo_tuss        text NOT NULL,
  codigo_glosa       text,                 -- NULL em divergência e quando a operadora não informa
  motivo_texto       text,
  glosado_centavos   bigint NOT NULL CHECK (glosado_centavos > 0),
  recuperado_centavos bigint NOT NULL DEFAULT 0,
  prazo_recurso      date NOT NULL,
  prazo_origem       text NOT NULL,        -- contrato | padrao
  status             text NOT NULL DEFAULT 'aberta',
    -- aberta | em_recurso | recurso_enviado | revertida | revertida_parcial | mantida | aceita
  aceite_justificativa text,
  responsavel_id     uuid,
  criado_em          timestamptz NOT NULL DEFAULT now(),
  atualizado_em      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_glosas_fila ON glosas (unidade_id, status, prazo_recurso);
CREATE INDEX ix_glosas_reversao ON glosas (registro_ans, codigo_glosa, status);

-- trilha append-only de toda mudança de status de glosa e recurso
CREATE TABLE glosa_eventos (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  glosa_id uuid NOT NULL REFERENCES glosas(id),
  de text, para text NOT NULL, motivo text, usuario_id uuid,
  criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE recursos (
  id                 uuid PRIMARY KEY,
  tenant_id          uuid NOT NULL,
  unidade_id         uuid NOT NULL,
  guia_id            uuid NOT NULL,
  registro_ans       text NOT NULL,
  status             text NOT NULL DEFAULT 'rascunho',
    -- rascunho | pronto | em_lote | enviado | respondido | descartado
  texto              text,                 -- CHECK no serviço: char_length <= limite_caracteres
  limite_caracteres  int NOT NULL,         -- do contrato no momento da criação
  texto_origem       text,                 -- manual | ia_aceito | ia_editado
  evidencias         jsonb NOT NULL DEFAULT '[]',
    -- [{ "trecho": "…", "evolucao_versao_id": "…", "origem": "analise_dut" | "campo23" | "manual" | "tissia_ai_recurso" }]
  anexos             jsonb NOT NULL DEFAULT '[]',   -- [{ "tipo": "evolucao_pdf" | "documento" | "anexo", "ref_id": "…", "rotulo": "…" }]
  tissia_ai_recurso_id   text,                 -- SPK-02
  sugestao           jsonb,                -- resposta da SPK-02 como veio, nunca editada
  lote_recurso_id    uuid,
  resultado          text,                 -- revertido_total | revertido_parcial | mantido
  recuperado_centavos bigint,
  criado_por         uuid NOT NULL,
  criado_em          timestamptz NOT NULL DEFAULT now(),
  atualizado_em      timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE recurso_glosas (
  tenant_id uuid NOT NULL,
  recurso_id uuid NOT NULL REFERENCES recursos(id),
  glosa_id uuid NOT NULL REFERENCES glosas(id),
  PRIMARY KEY (recurso_id, glosa_id)
);
-- uma glosa tem no máximo um recurso não descartado: checado no serviço, na mesma transação (glosa travada com FOR UPDATE)

-- append-only: aceite humano do texto (regra invariante "nada da IA sem aceite humano registrado")
CREATE TABLE recurso_decisoes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  recurso_id uuid NOT NULL REFERENCES recursos(id),
  acao text NOT NULL,                      -- aceito | editado | descartado
  texto_final text,
  usuario_id uuid NOT NULL,
  criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE lotes_recurso (
  id              uuid PRIMARY KEY,
  tenant_id       uuid NOT NULL,
  unidade_id      uuid NOT NULL,
  registro_ans    text NOT NULL,
  numero          text NOT NULL,           -- R-0012, tabela `sequencias` da RFC-PEP-006 (nome = 'lote_recurso')
  status          text NOT NULL DEFAULT 'aberto',   -- aberto | gerado | enviado
  formato         text NOT NULL DEFAULT 'pacote_portal',  -- pacote_portal | xml_tiss (Q2)
  pacote_objeto   text, pacote_hash text,
  protocolo       text, enviado_em timestamptz,
  criado_por      uuid NOT NULL,
  criado_em       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (unidade_id, numero)
);
```

`# ponytail:` `glosas` e `recursos` são as únicas tabelas mutáveis do módulo (item de trabalho); toda transição vai para `glosa_eventos` e `recurso_decisoes`. Retorno, itens e desfechos não mudam depois de gravados, exceto `tratamento`, `tissia_ai_*` e `guia_id` (vínculo manual), auditados.

**Desvio do Marco D a absorver:** sem `guia_revisoes` (RFC-PEP-006, notas), o valor faturado da guia vem de `guias.itens` + preço; o item do demonstrativo traz `informado_centavos`, que é o que a conciliação compara. Quando a RFC-PEP-006 entregar `guia_procedimentos`, a conciliação também confere item a item o enviado contra o informado (aviso "valor informado difere do enviado").

### Importação e conciliação

1. `POST /api/v1/retornos` (multipart) grava o arquivo cifrado, calcula SHA-256 e responde `202` com o `id`. Mesmo hash na mesma unidade → `200` com o retorno existente (idempotente).
2. Job `retorno-importar` (BullMQ, worker): valida no XSD da versão declarada (`libxml2-wasm`, mesmo componente da RFC-PEP-006; imune a XXE). XSD inválido não descarta: grava `xsd_valido = false` e os erros, e a importação só segue se o parser extraiu todos os campos obrigatórios (número da guia, TUSS, valores). O parser **não lê** `nomeBeneficiario`, `numeroCarteira`, `numeroCNS`, `cpf` nem outras tags de identificação: nada do beneficiário entra no banco.
3. `registro_ans` do cabeçalho precisa casar com uma operadora com contrato na unidade (RFC-PEP-008) ou com o `registro_ans` dos beneficiários das guias vinculadas; senão `422 operadora_desconhecida`.
4. Vínculo: `numero_guia_prestador` = `guias.numero_prestador` na unidade (único por unidade, RFC-PEP-006). Não achou → `nao_vinculado`, com ação "Vincular" manual. Meta da tela: "88 de 88 guias vinculadas".
5. Classificação por item (função pura `classificarItemRetorno` em `@pep/contracts`, testada por tabela):

| Condição (centavos) | `situacao` |
| :--- | :--- |
| sem guia | `nao_vinculado` |
| `glosa > 0` e `pago = 0` | `glosa_total` |
| `glosa > 0` e `pago > 0` | `glosa_parcial` |
| `glosa = 0` e `pago < contrato − tolerância` | `divergente_abaixo` |
| `glosa = 0` e `pago > contrato + tolerância` | `divergente_acima` |
| demais | `pago` |

   `contrato` = preço contratado vigente na `data_execucao` × quantidade (`ContratosService.precoContratado`, RFC-PEP-008). Tolerância = 1 centavo por unidade (arredondamento). Sem contrato cadastrado, `contrato_centavos = NULL`, `contrato := informado` e a tela mostra "sem contrato cadastrado; divergência não calculada".
6. Desfecho por guia vinculada (mesma transação): todos os itens `glosa_total` → `glosado_total`; algum item com glosa → `glosado_parcial`; senão → `pago` (divergência sem código **não** vira glosa para a Spike; é questão de tabela, não de documentação clínica — ver Q3). `codigo_glosa_principal` = código do item de maior glosa.
7. Glosa por item `glosa_total`/`glosa_parcial` (`tipo = glosa`), com `prazo_recurso = data_demonstrativo + prazo_recurso_dias` do contrato; sem contrato, padrão do tenant (`retorno.prazo_recurso_padrao_dias`, default 15 dias corridos **[validar]**) e `prazo_origem = padrao` (badge "prazo padrão" na central).
8. Outbox na mesma transação: `RetornoImportado` e um `DesfechoRegistrado` por guia.

Uma guia que aparece em dois demonstrativos (ex.: análise de conta e depois pagamento) gera dois desfechos; vale o mais recente. Glosa de item já glosado no retorno anterior não é duplicada (chave: `guia_id` + `codigo_tuss` + `data_execucao` com glosa aberta → atualiza valor via `glosa_eventos`).

### Divergências e padrão

- Padrão = ≥ 3 itens `divergente_*` do mesmo retorno com mesmos `codigo_tuss` e diferença unitária (`(pago − contrato) / quantidade`). Se a diferença unitária bate exatamente com `preço da versão vigente − preço da versão anterior` da tabela negociada (RFC-PEP-008) e a execução é posterior ao início da versão vigente, a dica é "Provável uso da tabela anterior pela operadora. O reajuste vale desde DD/MM (Contratos)".
- "Contestar as N guias" → cria `glosas` `tipo = divergencia`, `codigo_glosa = NULL`, `glosado_centavos = contrato − pago`, prazo do contrato; itens `tratamento = contestado`. Entram na central como qualquer glosa, sem recurso assistido por IA (não há argumento clínico).
- "Aceitar diferença e baixar" → itens `tratamento = diferenca_aceita`, entram na baixa pelo valor pago. Exige justificativa curta (registrada).
- `divergente_acima` não tem ação de contestação: a tela sugere "Conferir tabela no contrato" (pode ser pagamento a maior a devolver **[validar tratamento]**).

### Conciliação (baixa)

"Conciliar N guias" marca `tratamento = conciliado` nos itens `pago` (e `diferenca_aceita`) e publica `RetornoConciliado` com `retorno_id` e `item_ids`. A RFC-PEP-010 consome o evento e cria/atualiza contas a receber; até lá o evento fica publicado sem consumidor (o estado da conciliação já vale na tela 19). Itens glosados não entram na baixa: o valor pago de uma glosa parcial entra quando o item é conciliado; o glosado volta à base só se o recurso reverter (desfecho `origem = recurso`).

### Máquinas de estados

Glosa:

```
aberta ──(criar recurso)──▶ em_recurso ──(lote de recurso enviado)──▶ recurso_enviado
em_recurso ──(recurso descartado)──▶ aberta
aberta ──(aceitar, justificativa obrigatória)──▶ aceita
recurso_enviado ──(resultado)──▶ revertida | revertida_parcial | mantida
```

Prazo vencido não muda o status: a linha ganha o selo "Prazo vencido" e recorrer fora do prazo exige justificativa (registrada em `glosa_eventos`).

Recurso:

```
rascunho ──(aceite humano: aceito/editado, texto ≤ limite, ≥ 1 evidência)──▶ pronto
pronto ──(adicionar ao lote)──▶ em_lote ──(lote enviado + protocolo)──▶ enviado ──(resultado)──▶ respondido
pronto | em_lote ──(editar texto)──▶ rascunho (sai do lote aberto)
rascunho | pronto ──(descartar)──▶ descartado (glosas voltam a aberta)
```

Lote de recurso: `aberto ──(gerar pacote)──▶ gerado ──(registrar envio: protocolo, data)──▶ enviado`. Recurso só entra em lote `aberto` da mesma unidade e operadora.

### Recurso

1. "Criar recurso" numa glosa agrupa as glosas abertas da mesma guia e mesmo retorno (um recurso por guia, como o pedido da SPK-02).
2. Evidências pré-preenchidas, sem IA nova: os trechos literais de critérios DUT da última análise da guia (`analises_tissia_ai.resposta`) e o Campo 23 aceito da guia. O faturista remove ou acrescenta trechos; trecho manual só é aceito se for **substring literal** da versão da evolução usada na guia (validado no servidor). O faturista lê só o trecho vinculado à guia (`prontuario:ler_trecho_guia`, RFC-PEP-001).
3. Texto: manual na P2 até a SPK-02 (fatia 6). Com SPK-02 e unidade em modo `assistido` (ADR-PEP-0006), "Sugerir texto" chama `POST /v1/glosas/recurso` pelo SpikeAdapter com `clinica_ref`, `analise_id`, `guia_ref`, `registro_ans`, `itens_glosados` (TUSS, valor, `codigo_glosa_tuss`; `motivo_glosa_texto` só se Q4 liberar), `evolucao_clinica` (mesma versão e truncamento da análise, ≤ 20.000), `documentos_anexos` (rótulos) e `limite_caracteres` do contrato. A sugestão fica em `sugestao` e aparece marcada como IA; `criterios_nao_documentados` aparecem como aviso ("recurso provavelmente não reverte sem complemento"), nunca no texto. Timeout de 6 s, `degradado` ou falha → o recurso segue manual com as evidências lexicais, sem bloqueio.
4. Aceite: Aceitar / Editar / Descartar gravam `recurso_decisoes` (usuário, horário, texto final). Só recurso `pronto` entra em lote. Com sugestão da Spike, a decisão também vai como feedback (`POST /v1/glosas/recurso/{id}/feedback`, proposto na SPK-02), pela fila.
5. Anexos: evolução da guia em PDF (assinado quando a RFC-PEP-011 existir; antes, PDF da versão fechada com o hash da versão impresso) e documentos/anexos do atendimento (laudos, RFC-PEP-004) escolhidos da lista.
6. Lote de recurso: "Gerar pacote" monta um ZIP com um PDF por recurso (guia, itens, motivo, texto, evidências citadas com data da evolução) e os anexos, salvo cifrado (`pacote_objeto`). Formato XML TISS de recurso de glosa só se a Q2 confirmar a mensagem e a operadora exigir (`formato = xml_tiss`). Envio é manual (portal) com registro de protocolo.
7. Resultado: informado manualmente (`POST /recursos/{id}/resultado`) ou detectado num demonstrativo posterior que traga pagamento para o mesmo item (regra exata a definir com a Q6). Gera `desfechos` `origem = recurso`, atualiza glosas (`recuperado_centavos`, status) e publica `DesfechoRegistrado`.

**Desfecho do recurso na Spike:** **não** usar `POST /v1/analises/{id}/desfecho` para o resultado do recurso. Na Spike o desfecho é um upsert por análise (`_upsert_desfecho`, verificado no código): reenviar com `pago` apaga o registro da glosa original, que é o sinal que o preditivo precisa. O resultado vai para `POST /v1/glosas/recurso/{id}/desfecho` (SPK-02) quando houver `tissia_ai_recurso_id`; recurso manual fica sem envio até a lacuna da Q8 ser resolvida.

### Taxa de reversão

`GET /api/v1/glosas/{id}` traz `historicoReversao = { recursos, revertidos }` contando recursos `respondido` da mesma clínica, mesmo `registro_ans` e mesmo `codigo_glosa` (revertido total ou parcial conta como revertido). Mostra "Histórico da clínica com este motivo nesta operadora: X de Y recursos revertidos." com Y ≥ 1; Y = 0 → "Primeiro recurso deste motivo nesta operadora." É contagem, não previsão; a previsão (`chance_reversao`) é da SPK-02, com o mínimo de 10 similares dela.

### Integração com a Spike

Tudo pelo SpikeAdapter (ADR-PEP-0005), por fila, nunca na requisição do usuário. Unidade com modo `desligado` → `tissia_ai_estado`/`tissia_ai_ingestao = nao_aplicavel`. Modo `silencioso` **envia** desfechos e XML (a linha de base precisa deles); só o recurso assistido exige `assistido`.

| Fila (consumidor) | Gatilho | Chamada | Idempotência e falha |
| :--- | :--- | :--- | :--- |
| `spike.ingerir` | `LoteEnviado` (XML do lote, RFC-PEP-006) e `RetornoImportado` | `POST /v1/tiss/lotes/ingerir` `{ clinica_ref, xml_conteudo }` | Ver abaixo |
| `spike.desfechos` | `DesfechoRegistrado` com `origem ≠ recurso` | `POST /v1/desfechos` em lotes de até 1.000 itens (agrupados por retorno, janela de 5 s) | Upsert por análise na Spike: reenvio é seguro. Retry com backoff (8 tentativas/24 h, depois DLQ) |
| `spike.desfecho-individual` | Baixa manual (RF-RET-05) | `POST /v1/analises/{analise_id}/desfecho` | Idem |

Mapeamento do desfecho ([[mapeamento-pep-tissia-ia|spike/README §3.4]]): `analise_id` = `analises_tissia_ai.tissia_ai_analise_id` da última análise da guia (pelo `atendimento_id`); sem análise → par `clinica_ref = "un_" + unidade_id`, `guia_ref = "gu_" + guia_id`; `status`; `valor_cobrado`/`valor_glosado` em reais; `codigo_glosa_tuss = codigo_glosa_principal`; `motivo_glosa_texto` = `null` até a Q4; `lote_ref` = `numero_demonstrativo` (ou `"rt_" + retorno.id`, até 128 caracteres). Itens em `nao_encontrados` → `tissia_ai_estado = nao_encontrado` ("sem análise Spike" na tela; esperado para guias de antes da integração).

Ingestão de XML:

- Antes do envio, o worker remove do XML as tags de identificação direta do beneficiário (`nomeBeneficiario`, `nomeSocialBeneficiario`, `numeroCarteira`, `numeroCNS`, `cpf`, `identificadorBeneficiario` e equivalentes por versão **[validar lista contra os XSD 3.05/4.00/4.01]**). Remoção por denylist **e** teste de propriedade: nenhum nome ou número de carteirinha das guias do lote aparece na string enviada.
- Limite da Spike: 5.000.000 caracteres, síncrono. O worker usa timeout de 120 s e fatia arquivos acima de 2 MB por guias, repetindo o cabeçalho (`# ponytail:` fatiamento por guia até a SPK-03; o parser da Spike não confere o hash do lote).
- **A Spike não deduplica a ingestão** (cada chamada insere de novo em `historico_guias_tiss`, verificado no código). Por isso: erro com resposta (4xx/5xx) → retry; **timeout → `tissia_ai_ingestao = incerto`, sem retry automático**; reenvio só por ação de gestor/faturista (`POST /retornos/{id}/spike/reenviar`), auditada. Com a SPK-03 (`Idempotency-Key = un_<unidade>:retorno:<arquivo_hash>`), o timeout passa a ter retry normal.
- A Spike vincula desfechos na ingestão por `numeroGuiaPrestador`, que no PEP não é o `guia_ref` (`gu_<uuid>`): a ingestão alimenta só o histórico; os desfechos vão sempre por `/v1/desfechos` (ver Q7).

### API (módulo `retorno`)

Corpos em camelCase, como a API já implementada. Permissões novas em `@pep/contracts`: `retornos:ler`, `retornos:escrever`, `glosas:ler`, `glosas:escrever` (faturista: todas; gestor: leitura + `glosas:escrever` para aceitar; médico, recepção e admin: nenhuma na P2).

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `POST` | `/api/v1/retornos` | Upload multipart (`arquivo`, ≤ 20 MB) → `202 { "id": "…", "estado": "importando" }`; mesmo hash → `200` com o existente |
| `GET` | `/api/v1/retornos?operadora=&competencia=&cursor=` | Lista com totais |
| `GET` | `/api/v1/retornos/{id}` | Cabeçalho, leitura (`versaoTiss`, `xsdValido`, `duracaoMs`, `qtdVinculados`/`qtdItens`), KPIs (faturado, pago, glosado e %, divergências), lote(s) e `spikeIngestao` |
| `GET` | `/api/v1/retornos/{id}/itens?filtro=todas\|com_diferenca\|glosadas\|divergentes\|nao_vinculadas&cursor=` | Tabela de conciliação + contagem por filtro |
| `GET` | `/api/v1/retornos/{id}/itens/{itemId}` | Painel lateral: contrato (valor, versão), pago, diferença, padrão detectado e itens do padrão |
| `POST` | `/api/v1/retornos/{id}/itens/{itemId}/vinculo` | `{ "guiaId": "…" }` (só item `nao_vinculado`; reclassifica e cria desfecho/glosa) |
| `POST` | `/api/v1/retornos/{id}/divergencias` | `{ "acao": "contestar" \| "aceitar_diferenca", "itemIds": ["…"], "justificativa": "…" }` |
| `POST` | `/api/v1/retornos/{id}/conciliacao` | `{ "itemIds": ["…"] }` ou `{ "todosPagos": true }` → `RetornoConciliado` |
| `POST` | `/api/v1/retornos/{id}/spike/reenviar` | Reenvia XML em estado `incerto`/`falhou` (auditado) |
| `GET` | `/api/v1/glosas?status=&operadora=&motivo=&profissional=&prazoAte=&de=&ate=&retorno=&cursor=` | Central ordenada por `prazo_recurso` asc + KPIs do período (glosado, em recurso, recuperado, prazo ≤ 5 dias, aceito sem recurso) |
| `GET` | `/api/v1/glosas/{id}` | Glosa + guia (número, procedimento, operadora) + recurso + `historicoReversao` |
| `POST` | `/api/v1/glosas` | Baixa manual: `{ "guiaId": "…", "codigoTuss": "<TUSS_…>", "glosadoCentavos": 32000, "codigoGlosa": "<T38_…>", "motivoTexto": "…", "dataDemonstrativo": "2026-11-05" }` → desfecho `manual` + glosa + desfecho individual na Spike |
| `POST` | `/api/v1/glosas/{id}/aceite` | `{ "justificativa": "…" }` → `aceita` |
| `POST` | `/api/v1/glosas/{id}/recurso` | Cria recurso `rascunho` com evidências pré-preenchidas (409 se a glosa já tem recurso ativo) |
| `GET` | `/api/v1/recursos/{id}` | Texto, contador, evidências, anexos, sugestão, decisões |
| `PATCH` | `/api/v1/recursos/{id}` | `{ "texto": "…", "evidencias": [...], "anexos": [...] }` (volta a `rascunho` se estava `pronto`/`em_lote`) |
| `POST` | `/api/v1/recursos/{id}/sugestao` | Pede texto à SPK-02 (fatia 6; `409 modo_nao_assistido`; `503` vira "sugestão indisponível" sem bloquear) |
| `POST` | `/api/v1/recursos/{id}/decisoes` | `{ "acao": "aceito" \| "editado" \| "descartado", "textoFinal": "…" }` |
| `POST` | `/api/v1/recursos/{id}/resultado` | `{ "resultado": "revertido_total" \| "revertido_parcial" \| "mantido", "recuperadoCentavos": 32000 }` |
| `POST` | `/api/v1/lotes-recurso` | `{ "registroAns": "…" }` → lote `aberto` (ou o aberto existente) |
| `POST` | `/api/v1/lotes-recurso/{id}/recursos` | `{ "recursoIds": ["…"] }` (só `pronto`, mesma operadora) |
| `POST` | `/api/v1/lotes-recurso/{id}/pacote` | Gera o pacote (job) |
| `GET` | `/api/v1/lotes-recurso/{id}/pacote` | Download (auditado) |
| `POST` | `/api/v1/lotes-recurso/{id}/envio` | `{ "protocolo": "…", "enviadoEm": "…" }` |

Erros RFC 9457 com `code` estável (`operadora_desconhecida`, `arquivo_nao_tiss`, `campos_obrigatorios_ausentes`, `texto_acima_do_limite`, `evidencia_nao_literal`, `recurso_sem_aceite`, `modo_nao_assistido`). Schemas Zod `.strict()` em `@pep/contracts/retorno.ts`, com `classificarItemRetorno`, `statusDesfechoDaGuia` e `prazoRecurso` como funções puras.

### Eventos

- **Publica:** `RetornoImportado` (`retorno_id`, `unidade_id`, `registro_ans`, `lote_ids`, contagens), `DesfechoRegistrado` (`desfecho_id`, `guia_id`, `status`, `origem`, valores), auxiliares do módulo `RetornoConciliado` (`retorno_id`, `item_ids`) e `RecursoEnviado` (`lote_recurso_id`, `recurso_ids`). Payload só com ids, códigos e valores (ADR-PEP-0008).
- **Consome:** `LoteEnviado` (RFC-PEP-006) → `spike.ingerir` do XML do lote; `RetornoImportado` → `spike.ingerir`; `DesfechoRegistrado` → `spike.desfechos`.
- **Consumidores externos:** `faturamento` (`DesfechoRegistrado` → status da guia `paga | glosada_parcial | glosada_total`; desfecho de recurso pode levar `glosada_*` → `paga`, complemento à máquina da RFC-PEP-006), `financeiro` (`RetornoConciliado`, `DesfechoRegistrado`, RFC-PEP-010), `bi` (RFC-PEP-012).
- **Pré-requisito (fatia 0):** o relay atual publica uma fila por **tipo** de evento (`evento.<tipo>`, `apps/api/src/worker.ts`); com dois ou mais consumidores do mesmo evento, só um processaria. O relay passa a publicar uma fila por **consumidor** (ADR-PEP-0008), com dedupe em `eventos_processados (consumidor, evento_id)`.

### Regras

- Falha, timeout ou `degradado` da Spike nunca bloqueia importação, conciliação, central nem recurso.
- Nenhum texto sugerido por IA entra em lote de recurso sem `recurso_decisoes` com `aceito` ou `editado`. Aviso legal fixo no painel.
- Argumento do recurso só com fatos da evolução (Spike ADR-0008); evidência manual precisa ser trecho literal da versão da evolução da guia.
- Retorno, itens, desfechos e decisões não são apagados (modelo de dados: faturamento nunca tem delete).
- Prazo de recurso é calculado na criação da glosa; mudança posterior no contrato não altera glosas existentes (`# ponytail:` recalcular só se o piloto pedir).
- Não inventar códigos da Tabela 38: só valores presentes na Tabela 38 carregada (`ref_dominios_tiss`); código desconhecido no XML é guardado e marcado "código não reconhecido na versão X".
- Módulo atrás da flag `modulos.retorno` por clínica (release plan).

## UI

Rotas canônicas da RFC-PEP-015: `/retornos` (lista), `/retornos/:id` (tela 19) e `/glosas` (tela 20, `?glosa=` abre o painel). Filtros e seleção em query string (`nuqs`). Ao ligar na API, sai o selo "Dados ilustrativos".

**Tela 19 (`apps/ui/src/modulos/faturamento/Retorno.tsx`)**

| Bloco | Fonte | Observação |
| :--- | :--- | :--- |
| Título "Retorno do lote L-…", badge de status, operadora · competência · N guias | `GET /retornos/{id}` (lote(s) vinculados, `competencia`, `qtdItens`) | Badge: "Pago", "Pago parcial" (há glosa), "Glosado" (todo glosado) |
| Zona "Importar demonstrativo" + resultado da leitura ("TISS 4.01 · lido em 1,8 s · 88 de 88 guias vinculadas") | `POST /retornos`, depois `GET /retornos/{id}` até `importado` (consulta a cada 1 s) | XSD inválido → aviso amarelo com os erros; `falhou` → mensagem e "Tentar outro arquivo" |
| KPIs Faturado, Pago, Glosado (% do faturado), Divergências sem glosa | `GET /retornos/{id}` | |
| Chips Todas / Com diferença / Glosadas / Divergentes | `GET /retornos/{id}/itens?filtro=` (contagens) | Mais o chip "Não vinculadas" quando > 0 |
| Tabela guia a guia (Guia, Procedimento, Faturado, Pago, Diferença, Tabela 38, Situação) | `GET /retornos/{id}/itens` | Rodapé: "N guias pagas conforme contrato estão ocultas pelo filtro" |
| Painel lateral da divergência (contrato, pago, diferença, padrão, dica de reajuste) | `GET /retornos/{id}/itens/{itemId}` | Dica de tabela anterior depende da RFC-PEP-008; sem ela o painel mostra só os valores |
| "Contestar as N guias", "Aceitar diferença e baixar" | `POST /retornos/{id}/divergencias` | Diálogo de justificativa no aceitar |
| Baixa no financeiro: "Conciliar N guias" | `POST /retornos/{id}/conciliacao` | Sem RFC-PEP-010 o efeito é só local; o texto não promete lançamento financeiro |
| "Enviar N glosas para a central →" | Link para `/glosas?retorno=<id>` | As glosas já foram criadas na importação |

**Tela 20 (`apps/ui/src/modulos/faturamento/Glosas.tsx`)**

| Bloco | Fonte | Observação |
| :--- | :--- | :--- |
| KPIs (glosado no período, em recurso, recuperado, prazo em até 5 dias, sem recurso) | `GET /glosas` (bloco `kpis`) | Chip "Período" → `de`/`ate` |
| Chips Operadora, Motivo, Status; "Ordenado por prazo" | Query de `GET /glosas` | Motivo = código + descrição da Tabela 38 (`ref_dominios_tiss`); filtro por médico entra como chip extra |
| Tabela (Guia, Operadora, Tab. 38, Motivo, Valor, Prazo, Status) | `GET /glosas` | Prazo: vermelho ≤ 2 dias, amarelo ≤ 5, neutro acima; "prazo padrão" quando sem contrato; status do recurso vira o rótulo ("Recurso em rascunho") |
| Painel "Recurso · G-…", prazo, resumo da glosa | `GET /glosas/{id}` | |
| "Evidências da evolução assinada (data)" | `recursos.evidencias` | Sem RFC-PEP-011 o título vira "Evidências da evolução (fechada em DD/MM)" |
| Texto do recurso com contador `N / limite` | `PATCH /recursos/{id}`; limite do contrato | Bloqueia digitação acima do limite, como o Campo 23 |
| Trecho de IA (componente `Ai`) | `recursos.sugestao` | Só na fatia 6, unidade `assistido`; antes o bloco é um editor de texto manual |
| Anexos | `recursos.anexos` | Escolha a partir dos documentos do atendimento |
| Aceitar / Editar / Descartar | `POST /recursos/{id}/decisoes` | Sem sugestão de IA, "Aceitar" vira "Marcar como pronto" |
| "Adicionar ao lote de recurso da Operadora X" | `POST /lotes-recurso` + `/recursos` | Desabilitado até `pronto` |
| "Histórico … X de Y recursos revertidos" | `historicoReversao` | |
| Aviso legal fixo | constante da UI | Sempre visível |
| Exportar | `GET /glosas?…&formato=csv` | CSV sem texto clínico (só ids, códigos, valores, status) |

Tela 02 (Início, aba Faturamento): contador de glosas com prazo ≤ 5 dias pela mesma rota (`prazoAte`). O contador de Glosas no menu deixa de ser fixo (nota da RFC-PEP-015).

## Segurança e LGPD

- O XML do demonstrativo tem nome e carteirinha: fica só no armazenamento de objetos, cifrado, acesso por `retornos:ler`; o banco não guarda nenhuma tag de identificação. Download do original auditado.
- A Spike recebe o XML sem as tags de identificação direta e desfechos só com refs (`un_`, `gu_`, `analise_id`). `motivo_glosa_texto` não sai até a Q4.
- Evidências e pacote de recurso contêm texto clínico: leitura grava evento de acesso ("quem viu", RFC-PEP-014); download do pacote auditado; faturista acessa só o trecho vinculado à guia.
- Logs: ids, códigos, contagens, estado e latência. Nunca o XML, `motivo_texto`, evidência ou texto de recurso.
- Retenção: 5 anos após o pagamento **[validar]** (modelo de dados), inclusive o XML original.

## Critérios de aceite

- [ ] RLS: usuário do tenant A não lê retorno, glosa ou recurso do tenant B, mesmo com id válido (teste com dois tenants).
- [ ] Demonstrativos de exemplo (sintéticos) em 4.03.00 (e nas versões anteriores que o piloto exigir), de análise de conta e de pagamento, importam com todas as guias vinculadas; reimportar o mesmo arquivo devolve o mesmo retorno sem duplicar itens.
- [ ] Nenhuma coluna do módulo contém nome, carteirinha, CNS ou CPF depois de importar um XML que os contém (teste varre as linhas gravadas).
- [ ] `classificarItemRetorno`: teste de tabela cobrindo as 6 situações e a tolerância de 1 centavo; `statusDesfechoDaGuia` para guia com itens mistos.
- [ ] Conciliação usa o preço contratado vigente na data de execução: item executado antes e depois de um reajuste classifica diferente (com RFC-PEP-008).
- [ ] Padrão detectado com 3 itens iguais e dica de "tabela anterior" quando a diferença bate com o reajuste.
- [ ] Glosa criada com prazo do contrato; sem contrato, prazo padrão e `prazo_origem = padrao`.
- [ ] Payload enviado a `/v1/tiss/lotes/ingerir` não contém nenhum nome nem carteirinha das guias (teste de propriedade); timeout deixa `incerto` sem retry.
- [ ] `/v1/desfechos` recebe até 1.000 itens por chamada; reprocessar o evento não cria envio duplicado (`eventos_processados`); `nao_encontrados` vira `nao_encontrado`.
- [ ] Spike fora (stub com timeout e 5xx): importação, central e recurso manual funcionam; nenhum 5xx para o usuário.
- [ ] Recurso sem decisão `aceito`/`editado` não entra em lote (409); texto acima do limite → 422; evidência manual que não é substring da evolução → 422.
- [ ] Resultado de recurso não chama `/v1/analises/{id}/desfecho`.
- [ ] Relay com dois consumidores do mesmo evento entrega a ambos (fatia 0).
- [ ] Importação de demonstrativo com 2.000 itens em ≤ 30 s no worker; `GET /glosas` com 5.000 glosas em p95 ≤ 400 ms.
- [ ] Telas 19 e 20 ligadas à API; selo "Dados ilustrativos" removido.

## Riscos

- Demonstrativos fora do padrão (PDF, planilha ou XML que não valida no XSD): medir a fração no piloto antes de construir importador alternativo.
- Códigos da Tabela 38 mudam entre versões da terminologia: `ref_dominios_tiss` guarda todas as versões (`versao_ref`) e a conciliação usa a vigente na data do demonstrativo (RFC-PEP-021); a verificação semanal avisa quando sai versão nova.
- Prazo de recurso depende do contrato; sem contrato o padrão conservador pode estar errado para a operadora, por isso o selo "prazo padrão".
- Recurso gerado por IA que afirme fato ausente é risco ético-jurídico: só fatos da evolução, aceite humano obrigatório, aviso fixo (Spike ADR-0008).
- Ingestão sem idempotência na Spike: estado `incerto` exige atenção humana até a SPK-03.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | Quais operadoras do piloto entregam demonstrativo por webservice e quais só por portal/PDF? | Produto + consultor de faturamento | Fatia 7 |
| Q2 | Limite de caracteres do recurso por operadora e formato do lote de recurso. O 4.03.00 **tem** mensagem de recurso de glosa (`recursoGlosa`, guia de recurso de glosa e `tissRecursoGlosaV4_03_00.wsdl` no pacote da ANS); falta saber quais operadoras do piloto exigem o XML em vez do portal **[validar]** | Consultor de faturamento | Formato do pacote na fatia 5 (padrão: pacote para portal, limite 500) |
| Q3 | Divergência sem código de glosa entra no fluxo de recurso ou de cobrança administrativa? Pagamento acima do contrato precisa de devolução? | Gestor financeiro piloto | Não (padrão: central como `divergencia`, recurso manual) |
| Q4 | A Spike deve receber `motivo_glosa_texto`? Passa por scrubber lá, mas não é necessário para o preditivo | Pedro | Não (padrão: não enviar) |
| Q5 | Prazo padrão de recurso quando não há contrato (15 dias corridos?) e contagem a partir do demonstrativo ou do pagamento **[validar]** | Consultor de faturamento | Não (padrão configurável) |
| Q6 | Como a operadora sinaliza, no demonstrativo seguinte, o pagamento de um recurso? Dá para casar automaticamente? **[validar]** | Consultor de faturamento | Não (padrão: resultado manual) |
| Q7 | Proposta de SPK a abrir: `ingerir` vincula desfechos por `numeroGuiaPrestador`, que não é o `guia_ref` do PEP; aceitar um mapa número ↔ `guia_ref` (ou desligar o vínculo) para o histórico casar com as análises e com `guias/esquecidas` (RFC-PEP-009) | Pedro | Não |
| Q8 | Proposta de SPK a abrir (ou ampliar a SPK-02): registrar o desfecho de recurso **manual** (sem `recurso_id`) sem sobrescrever o desfecho original da análise | Pedro | Não (resultado de recurso manual fica só no PEP) |
| Q9 | O recurso precisa ser documento assinado (ICP, RFC-PEP-011) ou basta o texto no pacote com a evolução assinada anexa? | Consultor de faturamento + jurídico | Não (padrão: não assinado) |
| Q10 | ~~Fonte oficial da Tabela 38~~ **Resolvida em 13/11/2026:** Representação de Conceitos em Saúde da ANS, versão 202609 (`TUSS - Demais terminologias`), carregada pela [[RFC-PEP-021-dados-de-referencia-ans\|RFC-PEP-021]] | — | Fatia 2 da 021 antes da fatia 1 desta |

## Fatias de entrega

0. **Pré-requisitos:** relay do outbox com fila por consumidor e dedupe por consumidor (ADR-PEP-0008); Tabela 38 carregada em `ref_dominios_tiss` (RFC-PEP-021, fatia 2); porta de armazenamento de objetos cifrado (a mesma do XML de lote da RFC-PEP-006); permissões novas na matriz.
1. **Importação e conciliação (RF-RET-01):** upload, job, XSD, parser 3 versões, vínculo, classificação com preço contratado (ou fallback), desfecho por guia, `RetornoImportado`/`DesfechoRegistrado`; tela 19 com KPIs, filtros e tabela.
2. **Divergências e baixa (RF-RET-01):** padrão, painel lateral, contestar/aceitar, vínculo manual, conciliação com `RetornoConciliado`.
3. **Spike (RF-RET-02):** `spike.desfechos`, `spike.ingerir` (demonstrativo e `LoteEnviado`) com remoção de tags, fatiamento e estado `incerto`.
4. **Central de glosas (RF-RET-03, RF-RET-05):** glosas com prazo, lista, filtros, KPIs, aceite, baixa manual com desfecho individual; tela 20 (lista e KPIs); contador do menu e da tela 02.
5. **Recurso manual (RF-RET-04):** evidências pré-preenchidas, texto com limite, anexos, decisões, lote de recurso, pacote, protocolo, resultado, taxa de reversão; painel da tela 20.
6. **Recurso assistido (RF-RET-04):** só depois da SPK-02 aceita e implantada na Spike; sugestão, feedback e desfecho do recurso na Spike.
7. **Demonstrativo por webservice (opcional):** depois da ADR-PEP-0012 aceita e do parceiro escolhido (`OperadoraGateway.buscarDemonstrativos`), mesmo pipeline com `origem = webservice`.

## Notas de implementação

Implementação das fatias 0–5 (mais infraestrutura dos consumidores da fatia 6) concluída com os seguintes desvios e observações documentados:

- **`contrato_centavos` sem RFC-PEP-008:** Como a RFC-PEP-008 (contratos com operadoras) ainda não está implementada, `retorno_itens.contrato_centavos` é persistido como `NULL` e o fallback usa `cobrado_centavos` (conforme previsto na seção *Classificação de itens* da RFC).
- **Sugestão assistida de recurso (`POST /api/v1/recursos/:id/sugestao`):** Mantida como endpoint funcional que retorna `503 recurso_sugestao_indisponivel` enquanto a proposta **SPK-02** não for aceita e implantada no repositório `tissia-ai`.
- **Propostas SPK abertas:** Criadas em [[SPK-13-ingestao-mapa-guia-ref|SPK-13]] (antiga `SPK-11` de ingestão, Q7: `mapa_guias`, `vincular_desfechos_automaticamente` e `lote_ref` em `POST /v1/tiss/lotes/ingerir`) e [[SPK-12-desfecho-recurso-manual|SPK-12]] (Q8: `POST /v1/glosas/recurso-manual/desfecho` sem sobrescrever o desfecho da análise original), sem editar o repositório `tissia-ai`.
- **Parser XML e cobertura de versões TISS:** O parser em `apps/api/src/modulos/retorno/retorno-xml.ts` utiliza `libxml2-wasm` navegando pelos nomes de elementos locais (`localName`) para suportar `demonstrativoAnaliseConta` e `demonstrativoPagamento` independentemente de prefixo/versão de schema, ignorando integralmente tags de identificação do beneficiário (`nomeBeneficiario`, `numeroCarteira`, `numeroCNS`, `cpf`, etc.).
- **Upload multipart sem dependência extra:** O endpoint `POST /api/v1/retornos` lê o stream da requisição via `Request` / `FormData` nativo do Node/Undici com limite de 50 MB, evitando adicionar `@fastify/multipart` ou `multer`.
- **Compactação ZIP do lote de recurso:** A dependência `fflate` foi movida de `devDependencies` para `dependencies` em `apps/api/package.json` para gerar o pacote `.zip` (XML de recurso + PDF resumo + anexos) em `POST /api/v1/lotes-recurso/:id/pacote`.
- **Central de Notificações (`Home.tsx`):** Mantida como prévia (`<Sample />`); o contador real de glosas com prazo ≤ 5 dias foi conectado ao badge do menu lateral em `apps/ui/src/Layout.tsx`.
