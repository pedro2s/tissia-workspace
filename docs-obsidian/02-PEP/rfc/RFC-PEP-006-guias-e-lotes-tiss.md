---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-006-guias-e-lotes-tiss.md
---

# RFC-PEP-006 — Guias TISS, mesa do faturista e lotes XML

**Fase:** P1 · **Status:** Em implementação (Marco D e Onda 5 concluídas em 14/11/2026; pendências registradas em Notas de implementação) · **ADRs:** ADR-PEP-0003, ADR-PEP-0007, ADR-PEP-0008, ADR-PEP-0012, ADR-PEP-0014
**Telas:** 13 (Revisão da guia SP/SADT — dados e checagem local), 16 (Mesa do faturista), 17 (Editor completo de guia), 18 (Gestão de lotes)
**Requisitos:** RF-FAT-01 a RF-FAT-13
**Depende de:** RFC-PEP-002 (beneficiário), RFC-PEP-004 (`AtendimentoFechado`, pedidos), RFC-PEP-005 (análise Spike); RFC-PEP-021 (XSD 4.03.00, campos e domínios da guia); cadastro mínimo de operadoras/contratos (versão TISS, prazo de envio)

## Objetivo

Transformar o atendimento em guia TISS correta, com validações locais que sempre funcionam (mesmo sem Spike), uma fila de trabalho para o faturista ordenada por risco e prazo, e a geração do lote XML 4.03.00 validado contra o XSD. A Spike não gera XML nem fala com operadora: isso é do PEP.

## Escopo

**Dentro (P1)**

- [RF-FAT-01] Guia de consulta e SP/SADT criadas a partir do atendimento (`GuiaRascunhoCriada`) ou manualmente. Internação: P3.
- [RF-FAT-02] `GuiaRevisao` imutável: toda alteração gera nova revisão com autor e origem (`manual`, `sugestao_tissia_ai`, `sistema`).
- [RF-FAT-03] Máquina de estados da guia (abaixo).
- [RF-FAT-04] Validações locais determinísticas: campos obrigatórios TISS 4.03.00, sexo/idade × procedimento, acessórios/excludentes, quantidades máximas, regras cadastradas da operadora, valores × tabela, prazo de envio, autorização prévia. Tabelas de regras versionadas no PEP.
- [RF-FAT-05] Mesa do faturista: fila por risco e prazo, abas (por risco, por prazo, minhas, análise parcial), filtros (operadora, tipo, médico, pendência, competência), visões salvas, seleção e ações em lote (revalidar, atribuir, mover para lote), atalhos `J/K/X/R/L`.
- [RF-FAT-06] Editor completo de guia com numeração dos campos TISS, procedimentos, anexos, histórico de revisões e da análise.
- [RF-FAT-07] Revalidação individual e em massa (local + Spike, RFC-PEP-005).
- [RF-FAT-08] Lote por operadora e competência: checagem prévia (XSD, numeração sequencial por unidade, hash, prazos, exclusão de guias com bloqueante), geração do XML na versão exigida pela operadora (4.03.00 na P1; versão anterior só se o piloto exigir **[validar]**).
- [RF-FAT-09] Envio manual na P1: download do XML para upload no portal da operadora e registro do protocolo. Webservice TISS por operadora na P2 (ADR-PEP-0012).
- [RF-FAT-10] Status do lote: em preparação → validado → enviado → protocolado → em análise → pago/pago parcial/rejeitado.
- [RF-FAT-11] Autorização prévia na guia: senha, validade e status (manual na P1).
- [RF-FAT-12] Relatório da checagem prévia (PDF/CSV).
- [RF-FAT-13] Guia SP/SADT impressa (solicitação de exame/procedimento) fiel ao formulário oficial, para o paciente levar ao executante; gerada da guia/pedido já revisado, sem CID estruturado (ADR-PEP-0012). **Tela nova: precisa de design.**

**Fora**

- Ingestão do lote enviado e do demonstrativo na Spike, desfechos e glosas → RFC-PEP-007 (P2).
- Webservices de elegibilidade, autorização e envio de lote → P2 (parceiro ou próprio, ADR-PEP-0012).
- Guias de honorários, internação e resumo de internação → P3.

## Design

### Máquina de estados da guia

```
rascunho ──(médico aprova / "enviar ao faturista")──▶ em_revisao
em_revisao ──(faturista libera; sem bloqueante local)──▶ validada
validada ──(entra em lote)──▶ em_lote ──(lote enviado)──▶ enviada
enviada ──(retorno, P2)──▶ paga | glosada_parcial | glosada_total
qualquer estado antes de enviada ──▶ cancelada (motivo)
em_lote ──(removida do lote)──▶ validada
```

Alterar uma guia `validada` ou `em_lote` volta para `em_revisao` (nova revisão) e a tira do lote ainda não enviado. Guia `enviada` não muda: correção = guia nova/recurso (P2).

### Modelo de dados

```sql
CREATE TABLE guias (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  tipo text NOT NULL,                       -- consulta | sp_sadt
  atendimento_id uuid, paciente_id uuid NOT NULL, beneficiario_id uuid NOT NULL,
  operadora_id uuid NOT NULL,
  numero_prestador text NOT NULL,           -- G-004812, sequencial por unidade
  status text NOT NULL,
  revisao_atual_id uuid,
  risco text, score_risco numeric(4,3), analise_estado text,   -- espelho para a mesa
  prazo_envio date,                         -- data atendimento + prazo do contrato
  responsavel_id uuid,                      -- faturista atribuído
  lote_id uuid,
  criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (unidade_id, numero_prestador)
);

-- append-only
CREATE TABLE guia_revisoes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, guia_id uuid NOT NULL REFERENCES guias(id),
  numero int NOT NULL,
  dados jsonb NOT NULL,                     -- campos TISS (caráter, CBO, solicitante, executante, CNES, senha…)
  indicacao_clinica text,                   -- Campo 23, CHECK (char_length <= 500)
  evolucao_versao_id uuid,                  -- versão usada como base
  origem text NOT NULL,                     -- manual | sugestao_tissia_ai | sistema
  autor_id uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now(),
  validacao_local jsonb,                    -- resultado das regras locais desta revisão
  UNIQUE (guia_id, numero)
);
CREATE TABLE guia_procedimentos (
  tenant_id uuid NOT NULL, guia_revisao_id uuid NOT NULL REFERENCES guia_revisoes(id), ordem smallint NOT NULL, codigo_tuss text NOT NULL,
  descricao text, quantidade smallint NOT NULL, valor_unitario numeric(12,2),
  data_execucao date, PRIMARY KEY (guia_revisao_id, ordem)
);
CREATE TABLE guia_cids (tenant_id uuid NOT NULL, guia_revisao_id uuid NOT NULL REFERENCES guia_revisoes(id), codigo text NOT NULL, principal boolean NOT NULL, PRIMARY KEY (guia_revisao_id, codigo));
-- CID da guia serve à análise da Spike e à conferência interna; não existe tag de CID no XML de SP/SADT nem de consulta 4.03.00 (ADR-PEP-0012)

CREATE TABLE lotes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  operadora_id uuid NOT NULL, numero text NOT NULL,   -- L-0461, sequencial por unidade
  competencia char(7) NOT NULL, versao_tiss text NOT NULL,
  status text NOT NULL, qtd_guias int, valor_total numeric(14,2),
  xml_objeto text, xml_hash text, checagem jsonb,
  enviado_em timestamptz, canal_envio text,             -- portal | webservice (P2)
  protocolo text, criado_por uuid, criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (unidade_id, numero)
);

CREATE TABLE sequencias (tenant_id uuid NOT NULL, unidade_id uuid NOT NULL, nome text NOT NULL, valor bigint NOT NULL, PRIMARY KEY (tenant_id, unidade_id, nome));
-- numeração por UPDATE ... RETURNING dentro da transação; sem buracos aceitos? ver Perguntas
```

### Validações locais

Rodam em toda nova revisão, em menos de 100 ms, sem rede. Fonte das regras, versionadas e com vigência:

| Grupo | Fonte | Exemplo de saída |
| :--- | :--- | :--- |
| Obrigatórios TISS 4.03.00 | XSD + tabela de obrigatoriedade por tipo de guia | "CNES do executante (31)" |
| Sexo/idade | Tabela TUSS **[validar fonte oficial]** | "Compatíveis com masculino, 38 anos" |
| Acessórios/excludentes | Regras da operadora/contrato cadastradas | "Itens 1 e 2 não podem ser cobrados na mesma data" |
| Quantidades máximas | Regra da operadora | "20 informadas, máximo 10" + ação "Dividir em 2 guias" |
| Valores × contrato | Tabela de preços (P1 mínima, P2 RFC-PEP-008) | "Iguais à tabela negociada" |
| Autorização | Exigência por TUSS/operadora + senha na guia | "Sem senha de autorização" |
| Prazo | Prazo de envio do contrato | "Vence em 2 dias" |

Severidades `erro` (impede liberar para lote) e `aviso`. As mesmas checagens existem na Spike para quem não tem PEP; no PEP elas são a garantia de "nunca bloquear sem IA". `# ponytail:` duplicação consciente de regras simples PEP × Spike; unificar só se divergirem com frequência (o painel mostra a fonte de cada alerta).

### Lote XML

1. Faturista escolhe operadora e competência; o sistema propõe guias `validada` dentro do prazo.
2. Checagem prévia: reanálise Spike das guias necessárias (gatilho `pre_lote`, RFC-PEP-005), exclusão das com erro/bloqueante (listadas), XSD da versão escolhida, numeração, hash.
3. Geração do XML (`mensagemTISS` / `loteGuias`) com template por versão; hash conforme a especificação TISS **[validar algoritmo e campo do hash na versão 4.03.00; ver "Dados de referência da guia"]**; validação XSD com `libxml2-wasm` (WebAssembly isolado, sem bindings C++ nativos e imune a XXE) no worker.
4. XML salvo no armazenamento de objetos (imutável, versão por hash); `LoteGerado`.
5. Envio manual: download + "registrar envio" (data, canal `portal`, protocolo) → `LoteEnviado`. Guias do lote vão para `enviada`.

Geração roda em job BullMQ (`lote-gerar`), com progresso por SSE. Meta: 500 guias em ≤ 60 s.

### API (módulo `faturamento`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/guias?status=&operadora=&risco=&prazo_ate=&responsavel=&ordem=risco` | Mesa (cursor) + totais (na fila, risco alto, prazo 48 h, prontas, valor) |
| `POST` | `/api/v1/guias` | Guia manual |
| `GET` | `/api/v1/guias/{id}` | Guia + revisão atual + validação local + resumo da análise |
| `GET` | `/api/v1/guias/{id}/sugestao-dados` | Sugestão de autopreenchimento a partir dos cadastros de origem (`SugestaoDadosGuia`, [[ADR-PEP-0028-resolvedor-de-dados-da-guia\|ADR-PEP-0028]], *em implementação — a confirmar na integração*) |
| `POST` | `/api/v1/guias/{id}/revisoes` | Nova revisão `{ "base": 3, "dados": {...}, "procedimentos": [...], "cids": [...], "indicacao_clinica": "…" }` → 409 se `base` desatualizada |
| `POST` | `/api/v1/guias/{id}/transicoes` | `{ "para": "em_revisao" \| "validada" \| "cancelada", "motivo": "…" }` |
| `POST` | `/api/v1/guias/acoes-em-lote` | `{ "acao": "atribuir" \| "revalidar" \| "mover_para_lote", "guia_ids": [...] }` |
| `POST` | `/api/v1/lotes` | `{ "operadora_id": "…", "competencia": "2026-11", "versao_tiss": "4.03.00" }` → job |
| `GET` | `/api/v1/lotes/{id}` | Status, checagem, linha do tempo |
| `GET` | `/api/v1/lotes/{id}/xml` | Download (auditoria do download) |
| `POST` | `/api/v1/lotes/{id}/envio` | `{ "canal": "portal", "protocolo": "…", "enviado_em": "…" }` |

### Eventos

- Consome `AtendimentoFechado` → cria guia(s) e publica `GuiaRascunhoCriada`.
- Publica `GuiaRevisada` (toda nova revisão), `GuiaValidada`, `LoteEmPreparacao`, `LoteGerado`, `LoteEnviado`.
- Consome `AnaliseTissiaAiConcluida` → atualiza `risco`, `score_risco`, `analise_estado` da guia (ordenação da mesa).
- `LoteEnviado` é consumido na P2 pela RFC-PEP-007 (ingestão do lote na Spike e prazo de retorno).

### Dados de referência da guia (ADR-PEP-0024)

- **Campos e obrigatoriedade:** a tabela "Obrigatórios" das validações locais é carregada do Componente de Conteúdo e Estrutura 202511 (campos numerados da SP/SADT e da consulta, tamanho, `obrigatório`/`condicional`/`opcional`) e os domínios fechados da Representação de Conceitos 202609, em `ref_campos_guia` e `ref_dominios_tiss`. O `src/tiss/_data/sadt/fields.json` do projeto `tiss` serve só para conferir a carga (lista de 68 campos impressos da SP/SADT), não como fonte.
- **XSD:** `apps/api/ref/tiss/4.03.00/`, extraídos do pacote oficial do Componente de Comunicação 202511, com SHA-256 no manifesto.
- **Hash do lote:** seguir o Componente de Comunicação 4.03.00. Referência de implementação a conferir (não copiar): o `tiss` calcula MD5 sobre a concatenação do conteúdo das folhas da mensagem, em ISO-8859-1, na ordem do XSD **[validar contra o texto da ANS]**. Teste com lote de exemplo oficial cujo hash é conhecido.
- **Guia impressa (RF-FAT-13):** renderização HTML para impressão/PDF com os campos numerados do formulário; mesma fonte de campos acima. O template do `tiss` é só referência visual; o layout segue o canvas quando a tela for desenhada.
- **Validador na CI (em aberto, ADR-PEP-0012):** opcionalmente, um job de CI valida os lotes de exemplo também com o `tiss`, como segunda opinião ao XSD; nunca no runtime.

## UI

- Tela 16 (`apps/ui/src/modulos/faturamento/MesaFaturista.tsx`): KPIs, abas, filtros, tabela com risco/prazo/pendência principal, barra de seleção, gaveta "Gerar lote XML" com checagem prévia. Ordem padrão: risco desc, empate por prazo asc. Guias `parcial` com badge e aba própria.
- Tela 17 (`EditorGuia.tsx`): abas Consulta/SP-SADT, campos com número TISS, procedimentos com validação por linha, painel "Validação há N s" com ações corretivas ("Separar datas", "Dividir em 2 guias"), "Liberar para lote (N erros)" desabilitado com erro.
- Tela 13 (dados da guia e "Checagem local (sem IA)") compartilha componentes com a 17; o lado de IA é da RFC-PEP-005.
- Tela 18 (`Lotes.tsx`): resumo por operadora (canal, versão TISS, em aberto), tabela de lotes, detalhe com linha do tempo (criado → checagem → hash → envio → protocolo → aguardando demonstrativo). "Webservice ok" só na P2; na P1 todas as operadoras aparecem como "Envio pelo portal".
- Tablet (tela 26d) usa a mesma API da mesa com layout compacto.

## Segurança e LGPD

- XML contém PII (nome, carteirinha): armazenado cifrado, download auditado, acesso só a faturista/gestor da unidade.
- Guia e revisões sob RLS; faturista não edita evolução, só a guia.
- Indicação clínica alterada pelo faturista fica registrada como revisão com autor; aceitar justificativa da Spike exige papel `medico` quando a unidade configurar (a CFM atribui o conteúdo clínico ao médico **[validar]**).

## Critérios de aceite

- [x] `UPDATE` em `guia_revisoes` falha; nova revisão com `base` antiga → 409. *(fatia 6; teste com 5 revisões simultâneas)*
- [x] Transições inválidas rejeitadas (teste de tabela de estados). *(fatia 1, `@pep/contracts`)*
- [x] Validações locais: casos positivo/negativo para cada grupo; Spike desligada não altera o resultado local. *(fatias 3 e 8; E2E com a Spike fora do ar)* Falta: regras da operadora, sexo e idade, autorização e prazo (ver "Fica fora da onda 5").
- [x] Validações de obrigatoriedade vêm de `ref_campos_guia` carregada da publicação oficial; manifesto com a competência vigente. *(fatia 3)*
- [x] Nenhuma tag de CID no XML de SP/SADT ou consulta. *(testes do gerador)*
- [x] Lote gerado valida no XSD oficial 4.03.00 (arquivos XSD versionados no repo) para conjunto de guias de exemplo de consulta e SP/SADT. *(fatias 4 e 8)*
- [x] Numeração de guia e lote única por unidade sob concorrência (teste com 20 transações paralelas). *(fatias 4 e 6)*
- [x] Guia com bloqueante é excluída do lote e listada no relatório da checagem. *(fatia 4; a lista fica na tela de Lotes, o PDF/CSV da checagem fica para depois)*
- [x] Mesa com 5.000 guias: primeira página em p95 ≤ 400 ms. *(fatia 9: p95 de 32 ms sem filtro e 26 ms com filtros, `mesa-desempenho.spec.ts`)*
- [x] Telas 13, 16, 17 e 18 ligadas à API. *(13 na onda 4; 16 a 18 nas fatias 2, 5, 9)*

## Perguntas abertas

1. ~~Versão TISS da P1~~ **Resolvida em 13/11/2026:** gera 4.03.00, a vigente ([[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]]). **Confirmada em 14/11/2026 (Pedro):** é a versão exigida pelas operadoras para o faturamento eletrônico entre prestador e operadora. Versão anterior só entra se uma operadora do piloto exigir por escrito.
2. ~~Numeração do lote~~ **Resolvida em 14/11/2026:** buraco na sequência é aceito. O XSD 4.03.00 define `numeroLote` como `st_texto12` (1 a 12 caracteres) e não exige sequência. A única regra é a **unicidade por operadora** (não repetir número de lote já recepcionado). Implementação: restrição `UNIQUE (tenant_id, operadora_id, numero_lote)` e comprimento ≤ 12 validado no contrato Zod; sequência global ou por operadora é detalhe interno.
3. ~~Assinatura do XML~~ **Resolvida em 14/11/2026:** estruturar como **configurável por operadora**, já preparada para o padrão completo. No XSD, `ds:Signature` é opcional em `mensagemTISS` (após `epilogo`) e `assinaturaDigitalGuia` é opcional por guia; `epilogo/hash` é obrigatório. Regras: XML-DSig (W3C) conforme `tissAssinaturaDigital_v1.01.xsd`; certificado da cadeia ICP-Brasil (A1 `.pfx`/`.p12` para assinatura automática em servidor, A3 local; e-CNPJ da clínica ou e-CPF do profissional); o `hash` é calculado **antes** de inserir a assinatura e não a inclui. Flag `assinar_xml` por operadora (padrão `false`). A custódia do certificado A1 e o adaptador de assinatura reaproveitam a [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]] e a [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]]; até lá, o gerador deixa o ponto de extensão `assinarLote()` sem efeito.
4. ~~Fonte das regras da operadora~~ **Resolvida em 14/11/2026:** cadastradas no **backoffice** (equipe da plataforma), não pelo faturista nem por planilha. O PEP lê as regras; a edição fica na [[RFC-PEP-018-backoffice-clinicas-e-onboarding|RFC-PEP-018]] (B0). Consequência: as validações locais de quantidade e excludentes dependem de B0 e entram depois do núcleo da onda 5, com a lista de regras vazia até lá.
5. **Aberta:** a confirmação do item 1 e o limite do item 3 vieram de Pedro, não de uma operadora do piloto **[validar]** com a primeira clínica que escolher o convênio.

## Notas de implementação

**Marco D / D4 (09/11/2026): rascunho da guia e Mesa do faturista por risco (tela 16).**

- Tabela `guias` ampliada de forma aditiva com o espelho de ordenação da mesa (`profissional_id`, `numero_prestador`, `risco`, `score_risco`, `analise_estado`, `prazo_envio_dias`, `pendencia_principal`, `valor_total_centavos`) sob RLS por tenant.
- `ProntuarioService.fechar` cria a guia já espelhando o risco, o score e a pendência principal da última análise da Spike; `ProntuarioService.analisar` sincroniza a guia existente quando uma reanálise conclui.
- `GET /api/v1/guias?risco=` (`guias:ler`) devolve os 5 KPIs do topo (`naFila`, `riscoAlto`, `prazo48h`, `prontas`, `valorTotalCentavos`) e a lista ordenada por `scoreRisco DESC` (com guias `parcial` posicionadas entre `medio` e `baixo`, e desempate por `prazoEnvioDias ASC`).
- Tela `/faturamento` (`apps/ui/src/modulos/faturamento/MesaFaturista.tsx`), fiel ao design F16: KPIs reais, abas "Fila por risco" e "Análise parcial", seleção de linhas e atalhos `J/K/X/L`. Os chips de filtro (operadora, tipo, médico, pendência, competência), as abas "Por prazo"/"Minhas" e as ações em massa aparecem como no design, mas desabilitados (P1). O painel "Gerar lote XML" calcula operadora, guias prontas (`guiaProntaParaLote`, `@pep/contracts`), prazo e total; XSD, numeração e hash aparecem como `P1`, sem fingir checagem.
- **Desvios:** sem tabela `guia_revisoes` separada nem máquina completa de estados (`em_revisao → validada → em_lote → enviada`); valores calculados pela `TABELA_PRECOS_SINTETICA_CENTAVOS` (`@pep/contracts`); validações locais determinísticas completas, editor completo de guia (tela 17) e geração/validação XSD de lote XML 4.01 (tela 18) ficam para a P1.

- **Telas do canvas de 13/11/2026 já na UI como prévia (dados ilustrativos):** 17b (guia SP/SADT impressa, A4 paisagem, sem CID) em `/guia-impressa`, 17c (impressão com dado obrigatório faltando) em `/imprimir-guia`, botão "Imprimir guia" no editor, versão TISS do contrato (4.03.00) na geração de lote e nas validações, e "Fontes" com TUSS 202609. A checagem real e o PDF entram com esta RFC e a RFC-PEP-021.

**Onda 5, fatia 1 (14/11/2026): máquina de estados e revisões da guia (só backend).**

- Estados `rascunho → em_revisao → validada → em_lote → enviada` e `cancelada` (função pura `transicaoGuiaPermitida` em `@pep/contracts`). Esta fatia expõe só `em_revisao`, `validada` e `cancelada`; `em_lote` e `enviada` vêm com o lote.
- Tabela `guia_revisoes` (RLS por tenant, trigger append-only no banco, `UNIQUE (guia_id, numero)`, Campo 23 ≤ 500). Cada revisão guarda o conteúdo inteiro (itens, CIDs, indicação clínica), não diff. `guias.revisao_atual` e `guias.motivo_cancelamento` adicionados; a restrição de `status` foi **ampliada** (compatível com o app anterior).
- `GET /api/v1/guias/:id`, `POST /api/v1/guias/:id/revisoes` (`guias:escrever`) e `POST /api/v1/guias/:id/transicoes`. Editar `validada`/`em_lote` volta a `em_revisao`; `enviada` e `cancelada` recusam revisão (409). Cancelar exige motivo (≥ 10 caracteres). Só o perfil faturista libera (`em_revisao → validada`). Eventos `GuiaRevisada` e `GuiaStatusAlterado`, só com ids e estados.
- `fechar` agora cria a revisão 1 (origem `sistema`). Guias anteriores ganham a revisão 1 na primeira leitura ou escrita (sem backfill em migração: a RLS forçada impede o dono do schema de ler as linhas).
- **Desvios:** a revisão ainda não guarda os campos TISS (`dados jsonb`: caráter, CBO, CNES, senha…) nem `validacao_local`; chegam com as validações locais e o editor. Liberar não consulta validação local: o ponto de ligação está marcado no `transicionar`. A transição não grava em `auditoria_eventos`, só no outbox. `GET` trava a linha (`FOR UPDATE`) porque pode materializar a revisão base.
- **Fica para as próximas fatias:** UI da guia (`/faturamento/guias/:id`, tela 17), validações locais, guia de consulta, lote XML, ações em lote e filtros `nuqs`.

**Onda 5, fatia 2 (14/11/2026): UI da guia (tela 17) em `/faturamento/guias/:id`.**

- `EditorGuia` deixou de ser tela estática: carrega `GET /api/v1/guias/:id`, edita a quantidade dos procedimentos e o Campo 23 (contador 0/500) e grava como nova revisão. A rota `/editor` foi removida (caminho canônico da RFC-PEP-015); o número da guia na Mesa agora abre a tela, e o atendimento fica no botão "Abrir atendimento".
- Ações por estado: `rascunho` → "Enviar ao faturista"; `em_revisao` → "Liberar para lote" (desabilitado fora do perfil faturista e enquanto houver alteração não salva); "Cancelar guia" pede motivo (≥ 10 caracteres). Guia `enviada`/`cancelada` abre só leitura. O painel lateral lista as revisões.
- Visíveis e inertes (`aria-disabled` + `title` com a RFC): campos TISS (carteirinha, CBO, CNES, senha…), "Revalidar", "+ Procedimento", abas Consulta/Internação/Anexos e o painel "Validação". Liberar ainda não consulta regras locais.
- Descoberta: o `seed:demo` apagava `guias` e quebrou com a FK e o trigger append-only de `guia_revisoes`. O seed agora desabilita esse trigger só ali, como já faz com a evolução.
- E2E: `specs-demo/guia.spec.ts` (faturista percorre enviar, revisar, liberar, reeditar e cancelar; médico só envia) e ajuste de `canvas-tiss.spec.ts`. Na demo a `dra.exemplo` acumula médico, faturista e gestor; o teste de "só médico" usa `dr.exemplo.b`.

**Onda 5, fatia 3 (14/11/2026): validações locais de obrigatoriedade, formato e domínios.**

- `validarGuiaLocal` (função pura em `@pep/contracts`, sem rede e sem IA): grupos "Campos obrigatórios TISS", "Formato e domínios" e "Procedimentos". Obrigatoriedade vem de `ref_campos_guia` (`condicao = obrigatorio`, SP/SADT 202511) e os domínios fechados de `ref_dominios_tiss` (tabelas 23, 24, 26, 36, 50, 59 e 76 da versão vigente); nenhuma lista de campos obrigatórios fica no código.
- A guia guarda 16 campos TISS por revisão (`guia_revisoes.dados`, chaves por campo do formulário: 5, 8, 9, 12, 15–19, 21, 22, 29, 31–33, 91). Só esses são avaliados: campos só do formulário impresso (assinaturas, totais) e os que vêm de outros módulos ficam de fora. Nova revisão sem `dados` herda os da atual.
- A validação é calculada **na leitura** da guia (`GET /api/v1/guias/:id` traz `validacao`), não gravada: assim acompanha a base de referência vigente e não esbarra no append-only. **Liberar** (`em_revisao → validada`) devolve 422 com os campos em erro; aviso não bloqueia. Itens: código 8–10 dígitos, quantidade inteira 1–999; código repetido é aviso.
- **Sem a base da ANS carregada** a validação não bloqueia: devolve `referenciaCarregada: false` e um aviso (ausência da base nunca trava o faturamento). `ReferenciasService` (módulo `referencias`, cache em memória por 5 min) lê as tabelas; `GET /api/v1/referencias/dominios/:tabela` alimenta os selects, só para as tabelas dos campos da guia.
- **Decisão: a vigência da planilha não é usada.** Na carga 202609 os 28 códigos da tabela 59 (UF) trazem fim de vigência em 2012, mas o XSD 4.03.00 ainda exige exatamente esses códigos; avisar "fora de vigência" apareceria em toda guia. O XSD é a referência de aceitação do XML.
- **Fica de fora:** regras da operadora (quantidade máxima, excludentes, valores × contrato), cadastradas no backoffice (RFC-PEP-018); sexo e idade por procedimento (fonte oficial a validar); autorização por TUSS/operadora; prazo de envio; guia de consulta (a validação usa sempre a SP/SADT).
- UI: campos editáveis por grupo (selects com as tabelas da ANS), erro abaixo do campo, painel "Validação" com x/y por grupo e "Liberar para lote (N erros)" desabilitado, como no canvas. O erro de um campo some enquanto o valor difere do salvo; "Revalidar" recalcula com a base vigente.
- Testes: 3 em `@pep/contracts`, 2 novos na API (domínio e formato, opções de domínio; os de estado agora liberam com dados válidos) e o E2E da guia percorrendo obrigatório → dado inválido → corrigido → liberada. O teste da API semeia o mínimo de referência só se as tabelas estiverem vazias.

**Onda 5, fatia 4 (14/11/2026): lote XML TISS 4.03.00 validado no XSD (só backend).**

- `POST /api/v1/lotes` `{ registroAns, competencia }` (só o perfil faturista; `guias:escrever`), `GET /api/v1/lotes`, `GET /api/v1/lotes/:id` e `GET /api/v1/lotes/:id/xml` (download, auditado como `lote.xml_baixado`). Gera `mensagemTISS` `ENVIO_LOTE_GUIAS` com `guiaSP-SADT`, em ISO-8859-1.
- **Checagem prévia:** pega as guias `validada` da operadora (campo 1) e da competência (mês da data da solicitação ou do fechamento, em Brasília), **revalida com as regras de hoje** e valida cada guia e o lote inteiro no XSD. A guia que não passa é excluída e listada com o motivo (`checagem`); sem nenhuma elegível, 422 com os motivos. Limite de 100 guias por lote (`maxOccurs` do XSD): o excedente é listado e vai no próximo. As incluídas vão para `em_lote` (`guias.lote_id`).
- **Motivos sem dado do beneficiário:** a mensagem do libxml2 cita o valor recusado; a checagem guarda só "elemento (restrição)" para erro de valor e a mensagem estrutural sem namespace.
- **Número do lote:** contador atômico por unidade e operadora (`sequencias`, `INSERT … ON CONFLICT DO UPDATE … RETURNING`), com `UNIQUE (tenant, unidade, registro_ans, numero)` e no máximo 12 caracteres; buraco é aceito (resposta 2 da RFC). Teste com 20 geradores em paralelo.
- **Arquivo imutável:** `lotes.xml` (`bytea`), `xml_sha256`, `hash_tiss`, `numero`, `registro_ans` e `checagem` não mudam (trigger); o lote não se apaga; só `status` muda (envio, na fatia seguinte). O `seed:demo` desabilita esse trigger como faz com os demais.
- **Campos novos na guia, exigidos pelo XSD:** registro ANS (1), código (13) e nome (14) do contratado solicitante. Nome do beneficiário é opcional no XSD e **não vai ao XML**; CID também não (ADR-PEP-0012).
- **Hash do `epilogo`:** MD5 do texto das folhas do cabeçalho e do corpo, na ordem do documento, em ISO-8859-1, calculado antes de qualquer assinatura. **Continua [validar]:** não foi conferido contra um lote de exemplo oficial com hash conhecido; o teste só prova a regra implementada.
- **Simplificações assumidas (`ponytail:` no código):** geração síncrona (o job BullMQ com SSE entra quando 500 guias em ≤ 60 s deixar de caber); XML em `bytea` na linha (armazenamento de objetos depois); `codigoTabela` fixo em `22` (procedimentos), com descrição do catálogo da demo ou o próprio código; **valores unitários da tabela de preços sintética do Marco D**, não do contrato (RFC-PEP-008); a **senha de autorização (campo 5) e a data de execução** reais ainda não vão ao XML (execução = data da solicitação ou do fechamento); assinatura do XML segue configurável por operadora e desligada.
- **Armadilha de execução:** `libxml2-wasm` é só ESM com top-level await, e o app é CommonJS. A validação usa `import()` nativo e preguiçoso (`ReferenciasService.validarXmlTiss`, `referencias/tiss-xsd.ts`); `require()` quebraria a subida da API e o `pnpm openapi`. Sob o Jest o carregamento é por `import()` comum (limitação do runtime ESM dele). Exige Node ≥ 22.
- **Fica para a próxima fatia:** UI de Lotes (tela 18) e o registro de envio (`/envio`, protocolo, guias `enviada`), remover guia do lote, guia de consulta, ações em lote e filtros `nuqs`.

**Onda 5, fatia 5 (14/11/2026): tela 18 (Lotes) ligada à API e registro do envio manual.**

- `POST /api/v1/lotes/:id/envio` `{ canal: "portal", protocolo, enviadoEm }` (só o perfil faturista): o lote vai de `validado` para `enviado` e suas guias de `em_lote` para `enviada` (imutáveis). Uma vez só (409 depois); data não pode estar no futuro nem antes da geração do lote. O trigger passa a impedir mudar status, data, canal e protocolo depois do envio. Eventos `LoteEnviado` (consumido na P2, RFC-PEP-007) e `GuiaStatusAlterado`, só com ids. `canal` aceita só `portal`: webservice é da P2.
- **Defeito achado pelo E2E:** o campo de data e hora da tela não tem segundos, então o envio registrado logo após gerar o lote vinha "anterior" à criação e era recusado. A comparação é feita na precisão de minuto.
- Tela `/faturamento/lotes` (caminho canônico da RFC-PEP-015; a rota `/lotes` foi removida): resumo por operadora, tabela, linha do tempo do lote selecionado (criado, checagem prévia com incluídas e exclusões, hash, envio, protocolo), "Novo lote", "Baixar XML" (download autenticado por blob), "Registrar envio" e "Ver guias" com as incluídas e as excluídas com o motivo. O protocolo aparece mascarado (`••••7781`), como no design.
- **Desvio do canvas:** "Novo lote" abre um formulário na própria tela (registro ANS e competência) em vez de levar à Mesa, porque a Mesa lista guias por nome de operadora e o lote precisa do registro ANS. O painel "Gerar lote XML" da Mesa agora leva a Lotes e deixou de ser inerte; "Baixar relatório da checagem" continua inerte (a checagem está na lista de guias do lote).
- Visíveis e inertes (`aria-disabled` + `title`): "Importar lote enviado" e "Baixar protocolo" (RFC-PEP-007, P2) e os filtros da lista (próxima fatia). "Aguardando demonstrativo" é a etapa final, sempre pendente até a P2.
- Os valores dos lotes vêm da tabela de preços sintética (aviso no `title` da célula); o contrato entra com a RFC-PEP-008.
- E2E: `lote.spec.ts` percorre guia liberada → operadora sem guia (422 com motivo) → lote → download do XML (nome, `ENVIO_LOTE_GUIAS`, `4.03.00`, número da guia) → envio registrado → guia enviada e somente leitura; o médico só vê. Passa também com a Spike fora do ar.
- **Fica para a próxima fatia:** guia de consulta, ações em lote e filtros `nuqs` da Mesa e dos Lotes, remover guia do lote, reanálise `pre_lote`, relatório da checagem em PDF/CSV e a conferência do hash TISS com um lote oficial.

**Onda 5, fechamento (14/11/2026): fatias 6 a 9 e o que ficou de fora.**

- **Fatia 6, concorrência e numeração:** `NovaRevisaoGuia` exige `base` (revisão que a pessoa editava); revisão sobre base antiga → 409 sem gravar, e a tela recarrega a atual. O número da guia vem do contador atômico da unidade (`G-000001`…, helper `proximaSequencia` compartilhado com o lote) com `UNIQUE (tenant, unidade, numero)`. A demo reserva a sequência acima dos números fixos da seed.
- **Fatia 7, catálogo de operadoras (RFC-PEP-008, fatia 0 antecipada, só o catálogo):** `GET /api/v1/operadoras?q=` no módulo `contratos` (CADOP, só leitura; `%` e `_` digitados são texto). A validação da guia confere o campo 1: fora do CADOP é erro, inativa é aviso, sem catálogo carregado não confere. O lote recusa registro desconhecido e usa o nome oficial da operadora. `contratos`, `contrato_unidades` e `parametros()` da fatia 0 seguem para a RFC-PEP-008.
- **Fatia 8, guia de consulta:** campos e numeração próprios (`guia_de_consulta`: carteirinha 4, CNES 11, tipo de consulta 19…), um único procedimento, sem solicitante nem indicação clínica. O XML usa `ctm_consultaGuia`: o XSD apontou que o `CNES` fica **dentro** de `contratadoExecutante` (na SP/SADT é irmão). Lote por tipo (o XSD aceita um tipo de guia por lote; `CriarLote.tipo`, padrão `sp_sadt`). Fechar atendimento de convênio **sem pedidos** passa a gerar guia de consulta (TUSS 10101012, da tabela sintética) com a data do atendimento já preenchida; com pedidos segue SP/SADT; particular não gera guia. O editor segue o tipo da guia (o tipo é definido na criação).
- **Fatia 9, Mesa:** filtros (operadora, tipo, médico, pendência, competência), abas (risco, prazo, minhas, parcial) e limite saem do SQL; só a página resolve nomes. Os totais cobrem a fila inteira e `filtradas` conta a visão. **Correção:** a fila passou a contar só `rascunho`, `em_revisao` e `validada` (guia em lote, enviada ou cancelada saía contada como "na fila"). `POST /api/v1/guias/acoes-em-lote`: `atribuir` (a quem executa, `guias.responsavel_id`), `revalidar` (local, resultado por guia com o primeiro erro) e `mover_para_lote` (um lote por tipo, operadora e competência; `CriarLote.guiaIds` restringe as guias). Filtros e aba ficam na URL (nuqs). **Correção de usabilidade:** os atalhos `J/K/X/R` eram ignorados com o foco num checkbox (depois de marcar uma linha).
- **Fica fora da onda 5 (registrado, não esquecido):**
  - Regras da operadora (quantidade máxima, excludentes, valores × contrato): dependem do backoffice (RFC-PEP-018, B0) e do contrato (RFC-PEP-008, fatias 1 e 2). Os valores do lote seguem da tabela sintética.
  - Sexo e idade × procedimento (fonte oficial a validar), autorização por TUSS/operadora e prazo de envio.
  - Hash do `epilogo` conferido contra um lote oficial com hash conhecido (**continua [validar]**) e assinatura do XML (configurável por operadora, desligada).
  - Senha de autorização e data de execução reais no XML; guia manual (`POST /guias`); reanálise `pre_lote` na Spike; relatório da checagem em PDF/CSV; remover guia do lote; visões salvas e "atribuir" a outra pessoa; job BullMQ com SSE na geração (hoje síncrona; o p95 da Mesa não depende dele).
- **Ambiente:** o teste do worker (Redis e BullMQ) de `spike.spec.ts` falha quando outra API consome as mesmas filas do Redis (por exemplo `pnpm demo:dev` em paralelo): passou 123/123 sem ela. Não é regressão da onda.

**Revisão do PR da onda 5 (14/11/2026): correções antes do merge.** Uma revisão independente do diff (agente de revisão de hardening, só leitura) apontou o que segue; cada item foi confirmado no código e corrigido.

- **Conflito de número engolido:** `criarRascunho` usava `onConflictDoNothing()` sem alvo, que também engolia a violação do índice único do número e estourava `TypeError` no fechamento do atendimento. Agora o alvo é o atendimento, há guarda explícita, e `proximoNumeroGuia` pula números já ocupados por guia antiga (a numeração anterior derivava do id). Refazer o fechamento só atualiza itens, CIDs e valor enquanto a guia é o rascunho original; depois há revisões e elas são a fonte da verdade.
- **Migrações com dados reais:** a 0029 deduplica `numero_prestador` (a mais antiga mantém o número, as demais ganham `-2`, `-3`…) antes de criar o índice único; a 0033 faz o backfill da revisão 1 das guias anteriores à tabela. A RLS é forçada até para o dono do schema, então as duas desligam o `FORCE` só dentro da transação. Provado com um dono **sem** superusuário (a RLS o barra, o dedupe e o backfill funcionam, o `FORCE` volta); com o `app_owner` do Docker, que é superusuário, a RLS nunca o barra.
- **Lote e guia sem revisão:** o lote usa `leftJoin`; quem escolhe uma guia sem revisão vê "Sem revisão registrada: abra a guia" em vez de ela sumir. `GET /guias/:id` virou leitura pura: não grava a revisão base (que saía com o leitor como autor), não trava a linha e funciona para perfil só de leitura; a revisão base nasce na primeira escrita.
- **Guia em lote e lote enviado:** a guia `em_lote` fica travada (revisão e transição dão 409): o XML do lote já a contém e é imutável. Para alterá-la, **descartar o lote** antes do envio (`POST /api/v1/lotes/:id/descartar`, só faturista): o lote vira `descartado` (arquivo e número ficam guardados) e as guias voltam a `validada`. Depois do envio não se descarta. `guiaEditavel` do contrato passou a excluir `em_lote`, e a tela mostra o aviso com o atalho para Lotes.
- **Competência única:** o filtro da Mesa passou a usar a mesma definição do lote (mês da data do atendimento ou da solicitação na revisão atual; sem ela, o mês do fechamento). Antes a Mesa filtrava pelo mês do fechamento e o lote pela data do atendimento.
- **Mudança transversal declarada:** o pipe global de validação rejeita byte nulo (`\u0000`) em qualquer corpo ou querystring com DTO (422), porque o Postgres o recusa e virava 500. Vale para todos os módulos, não só o faturamento.
- **Constante:** o TUSS da consulta gerada no fechamento saiu de literal solto para `TUSS_CONSULTA_CONSULTORIO` em `@pep/contracts`.

**Demo de dados fictícios e coerência de guia (19/11/2026, PR #30 do `tissia`, em rascunho).** Contexto em [[demo-dados-ficticios]] e [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026]].

- **Data do preço = a do lote.** `FaturamentoService.dataDoPreco` (`faturamento.service.ts`): consulta usa `dataAtendimento`; SP/SADT usa `dataSolicitacao`; sem data válida, o dia de criação da guia em fuso de Brasília (`America/Sao_Paulo`, antes UTC). Só vale `AAAA-MM-DD` real do calendário (`dataIsoValida`: rejeita `2026-02-30`, `30/11/2026`, ano `0000`); texto livre malformado não chega mais ao SQL `date`. Vale em `detalhe`, `transicionar`, registrar revisão, criar rascunho e no consumidor de reajuste `faturamento.contrato_atualizado`. Antes, SP/SADT era precificada por `dataAtendimento` ou por "hoje" em UTC. Valores já gravados só mudam na próxima revisão ou reajuste. Testes: `apps/api/test/faturamento-preco-data.spec.ts` (22).
- **Mesma proteção em `validarRevisao`:** a data de execução usada nas regras de autorização também só aceita `AAAA-MM-DD` válido (antes, data ruim dava 500 em `GET /guias/:id` de consulta); o fallback segue o dia de hoje em UTC.
- **`GuiaDetalhe` aditivo** (`packages/contracts/src/faturamento.ts`, todos `.optional()`, `.strict()` mantido): `pacienteNome` (só com `pacientes:ler`), `profissionalNome`, `precosUnitariosCentavos` (preço do contrato por TUSS, na data acima) e `atendimentoDisponivel` (`false` quando o `atendimento_id` não existe: `guias.atendimento_id` não tem FK, guia importada ou de demonstração pode apontar para atendimento inexistente; a UI então desabilita "Abrir atendimento"). Os nomes são resolvidos fora da transação, em `try/catch` (falha vira `null`, sem 500) e com log `faturamento.nomes_da_guia_indisponiveis` (só id e tipo do erro). Snapshot `apps/api/openapi/openapi.json` regenerado (`pnpm --filter @pep/api openapi`). **Nenhum desses campos vai ao `SpikeAdapter`:** o diff não toca `modulos/tissia-ai` e nada lá lê `GuiaDetalhe`.
- **Guia impressa lê a guia real** (`/guia-impressa?guia=` e `/imprimir-guia?guia=`; antes eram mockups fixos). O preço impresso vem **só do contrato** (`apps/ui/src/modulos/faturamento/precosGuia.ts`): item sem preço imprime "—", sem total parcial, com a legenda de preencher à mão. O fallback para a tabela sintética existe somente com `VITE_AMBIENTE_DEMO=true`. **Desvio:** `EditorGuia.tsx` ainda usa o fallback sintético no total da tela (avisa no `title`) e a impressão da guia de consulta continua indisponível.
- **UI:** a Mesa deixou de exibir o selo "Dados ilustrativos" (a fila vem da API); Lotes avisa "Mostrando os 50 lotes mais recentes" quando a lista vem cheia (limite fixo em `LotesService.listar`, sem cursor); a Mesa e o editor não citam mais RFC/fase nos textos visíveis.
- **Risco residual:** `lotes.service.ts` (cerca das linhas 100 e 200) ainda leva data de texto livre ao SQL (erro 500 ao gerar lote com guia de data malformada). Não alterado.
- **Seed:** as guias "Prontas" da Mesa agora gravam a revisão completa (campos 1 a 33, CRM, UF, CBO, CNES). A G-004818 (consulta do seed base) fica incompleta de propósito (`consulta.spec.ts`).

**Indicação única no Campo 23 e autopreenchimento da guia (19/11/2026, status: em implementação).** Ver [[plano-guia-indicacao-unica-e-autopreenchimento|plano]], [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]] e [[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028]].

- **Já entregue na fundação (`feat/guia-fundacao`, commit `dc4ae7b`):** schemas `OrigemDadoGuiaSchema` e `SugestaoDadosGuiaSchema` em `@pep/contracts/validacao-guia.ts` (`registroAns`, `contratoId`, `dados`, `origens`, `pendencias`).
- **Intenção do plano em andamento (fatias G1, G3 e G4 — *a confirmar na integração*):**
  - **Campo 23 (G1):** a revisão 1 da guia SP/SADT criada em `ProntuarioService.fechar` → `FaturamentoService.criarRascunho` passa a nascer com `indicacaoClinica` preenchida a partir de `atendimentos.indicacao_pedidos` (com *fallback* para o primeiro pedido).
  - **Resolvedor no servidor (G3):** `GET /api/v1/guias/{id}/sugestao-dados` resolve dados de beneficiário (`pacientes`), profissional (`identidade`, incluindo `cbo`), unidade (`organizacao`, incluindo `clinicas.registro_cnes`) e contrato vigente na data (`contratos`, incluindo `codigoContratado`), sem inventar códigos de domínio ANS não confirmados nas bases carregadas em `ref_*`, e devolve `pendencias` acionáveis indicando onde cadastrar o que faltar. O mesmo resolvedor preenche `dados` da revisão 1 no fechamento do atendimento.
  - **Editor de guia — tela 17 (G4):** consome `GET /api/v1/guias/{id}/sugestao-dados` ao abrir e ao trocar a operadora, aplica sugestões **sem nunca sobrescrever** campos já editados pelo faturista, exibe a origem de cada dado preenchido e oferece atalhos para resolver as pendências cadastrais.

**Consulta + exame, preço do contrato e estados (20/11/2026, status: em implementação).** Ver [[ADR-PEP-0029-consulta-tiss-e-preco-de-contrato-na-guia|ADR-PEP-0029]] e [[fluxo-guia-consulta-exame-e-estados|fluxo]].

- **Guias do fechamento:** `FaturamentoService.gerarGuiasDoAtendimento`. Sem pedido: Guia de Consulta. Com exames: SP/SADT única com a consulta como linha (padrão) ou Guia de Consulta + SP/SADT quando o contrato tem `consulta_em_guia_separada`. Índice único `ux_guia_atendimento_tipo` (`atendimento_id`, `tipo`) no lugar de `ux_guia_atendimento` (migração 0048). O **desvio** acima ("o editor ainda usa o fallback sintético") foi corrigido: editor, Mesa, histórico, lote e BI usam só o preço do contrato.
- **Sem preço no contrato:** `guias.itens_sem_preco` (0048). Mesa, histórico e editor mostram "Sem preço no contrato"; a guia não é "pronta para lote"; o lote a exclui com o motivo (sem `valorProcedimentoCentavos` no XML). `GuiaMesaItem.itensSemPreco`, `GuiaDoPaciente.itensSemPreco`, `GuiaDetalhe.outrasGuiasDoAtendimento` e `Atendimento.guias` são aditivos e opcionais.
- **Tipo de atendimento (Tabela 50):** o resolvedor ([[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028]]) usa o código de "Consulta" (achado pela descrição na base da ANS) quando a guia inclui a consulta; sem achar, pendência. `ehProcedimentoDeConsulta` (`10101xxx`) em `@pep/contracts`.
- **Análise anti-glosa:** `sincronizarAnalise` atualiza a SP/SADT (ou a guia única); a Guia de Consulta separada nasce sem risco. BI recalcula todas as guias do atendimento.
- **Estados:** documentados em [[fluxo-guia-consulta-exame-e-estados|fluxo]] (onde cada perfil enxerga cada estado; Salvar não muda o estado).
- **Testes:** `faturamento.spec.ts` (cenário padrão, cenário de guia separada e idempotência, item sem preço), `lotes*.spec.ts`/`faturamento-robustez.spec.ts` (preço de contrato semeado), `guia-sugestao-dados.spec.ts` (Tipo de atendimento de consulta), `contratos.spec.ts`, UI (`EditorGuia.test.tsx`, `Contratos.test.tsx`, `precosGuia.test.ts`) e E2E `fluxo-medico-faturista.spec.ts`.
