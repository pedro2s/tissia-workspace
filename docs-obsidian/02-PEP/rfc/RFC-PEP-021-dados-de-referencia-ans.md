---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-021-dados-de-referencia-ans.md
---

# RFC-PEP-021 — Dados de referência da ANS: carga, versões e alerta de defasagem

**Fase:** P1, pré-requisito da onda 5 (RFC-PEP-006) e da fatia 0 da RFC-PEP-008 · **Status:** Implementada (fatias 1–3 entregues em 14/11/2026; fatias 4–6 planejadas, ver Notas de implementação) · **ADRs:** ADR-PEP-0003, ADR-PEP-0008, ADR-PEP-0012, ADR-PEP-0017, ADR-PEP-0019, ADR-PEP-0024
**Telas:** Início (card "Sistema"), barra superior (versão TISS), backoffice "Bases" (via RFC-PEP-020)
**Requisitos:** RF-REF-01 a RF-REF-04; RNF-11
**Depende de:** RFC-PEP-000 (worker, migrações, grants); consumida por RFC-PEP-006 (XSD, campos, domínios), RFC-PEP-007 (Tabela 38, campos do demonstrativo e do recurso), RFC-PEP-008 (operadoras); Spike: competência por base (prompt de refinamento da Spike, ADR-PEP-0024 §Consequências)

## Objetivo

Ter no PEP, antes de gerar a primeira guia real, as bases da ANS que o faturamento usa sem rede e sem IA (XSD, campos e obrigatoriedade das guias, domínios TISS, Tabela 38 e operadoras ativas), carregadas só da publicação oficial, com versão e proveniência rastreáveis, e saber **antes do release** quando a ANS publicou algo mais novo (alerta, sem bloqueio).

## Escopo

- [RF-REF-01] Manifesto versionado de cada base (fonte, URL, versão/competência, SHA-256, licença, data de carga) e XSD do Componente de Comunicação 04.03.00 no repo.
- [RF-REF-02] Carga idempotente por script, a partir do arquivo oficial, das tabelas `ref_campos_guia`, `ref_dominios_tiss` (inclui a Tabela 38) e `operadoras` (CADOP), mantendo todas as versões e o diff entre elas.
- [RF-REF-03] Verificação semanal de defasagem contra as publicações da ANS, com alerta operacional no Backoffice e relatório informativo no release (não bloqueia).
- [RF-REF-04] Status das bases (PEP e Spike) por API, no Início, na barra superior e no backoffice.

**Fora:** TUSS, CID-10, DUT e Rol (dono: Spike, SPK-09); Monitoramento TISS 01.06.00 (operadora → ANS, não se aplica ao prestador); atualização automática de base (proibida pela ADR-PEP-0024); preço de referência D-TISS.

## Design

### Bases e fontes

| Base | Arquivo oficial (pacote) | Destino no PEP | Usada por |
| :--- | :--- | :--- | :--- |
| `tiss_xsd` | `PadroTISSComunicao_202511.zip` → `tiss*V4_03_00.xsd`, `tissAssinaturaDigital_v1.01.xsd`, `xmldsig-core-schema.xsd` (sem os `*Monitoramento*` e sem WSDL) | `apps/api/ref/tiss/4.03.00/` (versionado no git) | 006 (validação do lote), 007 (demonstrativo) |
| `campos_guia` | `PadroTISS_ComponentedeContedoeEstrutura_202511.zip` → `.xlsx`, uma aba por documento (`Guia de Consulta`, guia SP/SADT, `Demonstrativo Analise Conta`, `Demonstrativo Pagamento`, `Guia de Recurso de Glosa`) | `ref_campos_guia` | 006 (obrigatórios, guia impressa), 007 |
| `dominios_tiss` | `Padrao_TISS_Representacao_de_Conceitos_em_Saude_202609.zip` → `TUSS - Demais terminologias - VERSÃO 202609.xlsx`, abas `Tab NN` | `ref_dominios_tiss` | 006 (domínios fechados), 007 (Tabela 38) |
| `operadoras` | CADOP, dados abertos da ANS ("operadoras de planos de saúde ativas", `Relatorio_cadop.csv`) | `operadoras` (RFC-PEP-008) | 002, 006, 008 |

Só se carregam as abas `Tab NN` referenciadas por algum campo de `ref_campos_guia` mais a Tabela 38; a lista sai da própria carga de campos, não de constante no código. O nome exato da aba da SP/SADT e as colunas de cada planilha se conferem na primeira carga **[validar]**; o script falha se o cabeçalho esperado não estiver lá.

### Manifesto (`apps/api/ref/MANIFEST.yaml`)

```yaml
esquema: 1
bases:
  tiss_xsd:
    versao: "4.03.00"
    competencia: "202511"
    fonte: "ANS — Padrão TISS, Componente de Comunicação"
    url_pagina: "<página oficial do Padrão TISS na gov.br>"
    pacote: { arquivo: "PadroTISSComunicao_202511.zip", sha256: "<sha256 do pacote>" }
    arquivos: { "tissV4_03_00.xsd": "<sha256>", "...": "..." }
    licenca: "não informada pela ANS"
    carregado_em: "<data>"
    monitorar: { url: "<página oficial>", padrao: "PadroTISSComunicao_(\\d{6})\\.zip" }
  campos_guia: { versao: "202511", ... }
  dominios_tiss: { versao: "202609", ... }
  operadoras: { versao: "<data da publicação do CADOP>", licenca: "CC BY, atribuição à ANS", ... }
```

Os arquivos oficiais (xlsx, csv, zip) **não** entram no git: o operador baixa, o script confere o SHA-256 contra o manifesto. Só os XSD ficam versionados (a geração do lote não pode depender de download).

### Tabelas (globais, sem `tenant_id`)

```sql
CREATE TABLE ref_campos_guia (
  versao_ref     text NOT NULL,             -- '202511'
  documento      text NOT NULL,             -- consulta | sp_sadt | demonstrativo_analise | demonstrativo_pagamento | recurso_glosa
  numero_campo   int,                       -- número impresso no formulário (NULL quando a mensagem não tem formulário)
  tag            text NOT NULL,             -- caminho da tag no XML
  rotulo         text NOT NULL,
  tipo           text NOT NULL,             -- como publicado (ex.: st_texto500)
  tamanho        int,
  condicao       text NOT NULL,             -- obrigatorio | condicional | opcional
  regra          text,                      -- texto da condição, como publicado
  dominio_tabela text,                      -- 'Tab 50', quando o campo é de domínio fechado
  PRIMARY KEY (versao_ref, documento, tag)
);

CREATE TABLE ref_dominios_tiss (
  versao_ref      text NOT NULL,            -- '202609'
  tabela          text NOT NULL,            -- '38', '50', ...
  codigo          text NOT NULL,
  termo           text NOT NULL,
  vigencia_inicio date, vigencia_fim date,  -- quando a planilha publica
  PRIMARY KEY (versao_ref, tabela, codigo)
);

CREATE TABLE ref_cargas (                   -- trilha de cada carga, para revisão e auditoria
  id uuid PRIMARY KEY, base text NOT NULL, versao_ref text NOT NULL, sha256 text NOT NULL,
  contagens jsonb NOT NULL,                 -- { novos, alterados, removidos, total } contra a versão anterior
  carregado_por text NOT NULL, carregado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ref_verificacoes (             -- resultado do job semanal
  base text NOT NULL, verificado_em timestamptz NOT NULL,
  versao_publicada text,                    -- o que a ANS publica hoje; NULL em erro
  situacao text NOT NULL,                   -- em_dia | defasada | erro
  PRIMARY KEY (base, verificado_em)
);
```

- A `ref_tabela38` da RFC-PEP-007 vira `ref_dominios_tiss` com `tabela = '38'`; o grupo é o prefixo de dois dígitos, calculado na consulta (agrupamento nosso, não da ANS). `ref_orientacao_recurso` continua na 007.
- Versões antigas nunca são apagadas: guia e demonstrativo validam contra a versão vigente na data deles (`refVigente(base, data)`).
- Grants em `drizzle/prelude`: `SELECT` para `app_rw`; escrita só pelo script, com o papel dono (`app_owner`). Não há RLS porque não há dado de tenant; nenhuma dessas tabelas recebe dado de clínica ou paciente.

### Carga (`pnpm --filter @pep/api ref:carregar <base> <arquivo>`)

1. Confere o SHA-256 do arquivo contra o manifesto; divergente → para. Para uma versão nova, `--nova-versao` grava a entrada no manifesto (mudança revisada em PR).
2. Lê o arquivo (xlsx com `exceljs` como `devDependency`, só no script), valida cabeçalhos e tipos, e monta as linhas.
3. Numa transação: insere a versão nova (ou não faz nada se já existe com o mesmo hash), calcula o diff contra a versão anterior e grava `ref_cargas`.
4. Imprime o diff (códigos da Tabela 38 novos ou removidos, campos que mudaram de condição) para o revisor do PR.
5. `tiss_xsd`: extrai os XSD do pacote para `ref/tiss/<versao>/` e grava os hashes no manifesto; um teste compila os XSD com `libxml2-wasm`.

### Verificação de defasagem (job `ref.verificar`, semanal)

- Agendamento repetível no worker (segunda-feira 06:00 `America/Sao_Paulo`), fora do caminho de requisição.
- Para cada base com `monitorar`: `GET` na página oficial com timeout de 10 s, só para hosts da lista fixa (`www.gov.br`, `www.ans.gov.br`, `dadosabertos.ans.gov.br`), extrai a versão publicada pelo `padrao` e compara com o manifesto. CADOP: compara a data de publicação do arquivo.
- Grava `ref_verificacoes`; `defasada` ou `erro` gera log estruturado (`base`, `versao_manifesto`, `versao_publicada`) e a métrica `ref_defasada{base}`. Nunca baixa nem carrega nada sozinho.
- Mudança de layout da página da ANS aparece como `erro`, não como `em_dia`.

### Verificação no release (`pnpm --filter @pep/api ref:verificar`, informativa)

Decisão de 13/11/2026 (ADR-PEP-0024): **não bloqueia**. Roda no pipeline de release e imprime, por base, `em_dia`, `defasada` ou `erro`, e compara a competência do PEP com a da Spike (`versao_bases`, quando a Spike expuser). Sai sempre com código 0; o responsável pelo release lê o relatório e decide. `--estrito` existe como opção para quem quiser reprovar manualmente, mas não é usado no pipeline. Sem `EXCECOES.yaml`. A mesma lógica alimenta o alerta no Backoffice, que é o dono da manutenção.

### API e telas

- `GET /api/v1/referencias/status` (`organizacao:ler`): `{ bases: [{ base, dono: 'pep' | 'spike', versao, carregadoEm, situacao, versaoPublicada }] }`, schema em `@pep/contracts`. As bases da Spike vêm da competência que ela expuser; até lá, só `versao_bases`.
- Início: o card "Sistema" troca o texto fixo ("Bases de regras atualizadas: TISS 4.01 e Rol ANS") pelos dados reais; base defasada vira aviso.
- Barra superior: a versão TISS mostrada vem do manifesto.
- Backoffice "Bases" (dono da manutenção): status, alerta de defasagem e acompanhamento da carga, pelo backend do backoffice (ADR-PEP-0021), na RFC-PEP-020.
- Layout conforme o canvas atualizado (prompt do canvas de 13/11/2026).

## Segurança e LGPD

- Nenhum dado de tenant ou de paciente nessas tabelas; saída de rede só para os hosts oficiais da lista fixa (sem URL vinda de usuário).
- Arquivos oficiais fora do git; XSD versionados com hash.
- O projeto `tiss` não é fonte nem dependência (ADR-PEP-0012); pode servir para conferir a carga manualmente.

## Critérios de aceite

- [ ] Manifesto com as quatro bases do PEP na versão vigente (Comunicação 4.03.00/202511, Conteúdo e Estrutura 202511, Representação de Conceitos 202609, CADOP do mês), com SHA-256 e licença.
- [ ] `ref:carregar` é idempotente, recusa arquivo com hash divergente e cabeçalho inesperado, e imprime o diff entre versões.
- [ ] Todo domínio referenciado em `ref_campos_guia` existe em `ref_dominios_tiss`; Tabela 38 com 470 termos na 202609.
- [ ] XSD 4.03.00 compila com `libxml2-wasm`; nenhum arquivo `Monitoramento` no diretório.
- [ ] Job semanal grava `ref_verificacoes` e marca `erro` quando a página muda de layout (teste com HTML de exemplo).
- [ ] `ref:verificar` lista cada base como em dia, defasada ou erro, compara com a Spike e sai com código 0 no pipeline.
- [ ] `GET /referencias/status` e o card do Início mostram versões reais; testes de API e E2E.

## Riscos

- **ANS muda o layout da página ou da planilha:** a verificação vira `erro` e a carga falha alto; ninguém fica com falsa sensação de "em dia".
- **Licença não informada** (XSD, Tabela 38): versionar XSD no repo segue a prática de mercado, mas pede confirmação jurídica.
- **Base defasada chegar à produção:** como a verificação não bloqueia, depende de o relatório ser lido; o alerta no Backoffice e a publicação mensal esperada reduzem o risco. Revisitar o bloqueio se acontecer.

## Perguntas abertas

| # | Pergunta | Quem | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | ~~Dono da manutenção e prazo~~ **Resolvida em 13/11/2026:** o Backoffice (equipe da plataforma). Falta o prazo para carregar uma publicação nova (sugestão: até 10 dias úteis da publicação) | Produto | Não |
| Q2 | ~~Frequência de publicação~~ **Resolvida em 13/11/2026:** mensal (CADOP e Representação de Conceitos); a verificação segue semanal | — | — |
| Q3 | ~~Licença dos XSD e da Tabela 38 para versionar no repo~~ **Resolvida em 14/11/2026 (Pedro):** a ANS permite uso, incorporação e redistribuição dos XSD e da Tabela 38 sem royalties nem autorização prévia (especificação técnica oficial de autarquia federal, RN 501/2022; Lei 9.610/98, art. 8º, IV). Registrar a fonte e a data de cada publicação no manifesto, como já feito. Parecer formal do jurídico não solicitado | Jurídico | — |
| Q4 | Usar o `tiss` como segunda validação do XML na CI (ADR-PEP-0012, em aberto) | Engenharia | Não |
| Q5 | O Backoffice só mostra o status e o alerta (padrão da primeira versão) ou também dispara a carga, com segunda aprovação (`aprovacoes_internas`, RFC-PEP-020)? | Produto | Não |

## Fatias de entrega

1. **XSD e manifesto:** `ref/tiss/4.03.00`, `MANIFEST.yaml`, teste de compilação. Destrava a geração e a validação do lote (RFC-PEP-006).
2. **Campos e domínios:** `ref_campos_guia`, `ref_dominios_tiss`, `ref_cargas`, `ref:carregar` com diff. Destrava as validações locais e a guia impressa (RFC-PEP-006) e a Tabela 38 (RFC-PEP-007).
3. **Operadoras (CADOP):** carga na tabela `operadoras`. Destrava a fatia 0 da RFC-PEP-008.
4. **Defasagem e relatório de release:** job semanal, `ref_verificacoes`, `ref:verificar` (informativo).
5. **Status nas telas:** `GET /referencias/status`, Início e barra superior; backoffice na RFC-PEP-020.
6. **Competência da Spike no relatório:** quando a Spike expuser a competência por base.

## Notas de implementação

**Fatias 1 a 3 (14/11/2026), migrações 0020 e 0021.**

- **Comando:** `pnpm --filter @pep/api ref:carregar <base> <arquivo>` (`apps/api/scripts/ref/carregar.ts`). Sem `--nova-versao` confere o SHA-256 do arquivo contra `apps/api/ref/MANIFEST.yaml` e recusa se divergir; com `--nova-versao --versao V` grava a entrada nova no manifesto (para revisar em PR). Idempotente: o mesmo arquivo uma segunda vez não faz nada, e outro arquivo para a mesma versão é recusado.
- **Bases carregadas e conferidas:** `tiss_xsd` 4.03.00 (pacote 202511; 7 XSD em `ref/tiss/4.03.00`, sem Monitoramento nem WSDL; o ZIP tem o mesmo SHA-256 registrado pelo projeto `tiss`, `db8640e…`), `campos_guia` 202511 (18 documentos: guias, demonstrativos e anexos, com a SP/SADT e a consulta entre eles), `dominios_tiss` 202609 (58 terminologias, Tabela 38 com os 470 termos) e `operadoras` (CADOP publicado em 13/11/2026, 1.101 ativas).
- **Desvio 1, sem caminho de tag:** a planilha de Conteúdo e Estrutura não traz o caminho da tag XML. `ref_campos_guia` usa a chave `(versao_ref, documento, ordem)` e guarda termo, número do campo (1–68 da SP/SADT, mais os adicionais 89 e 90), tipo, tamanho, formato, descrição, condição e a tabela de domínio citada na descrição. A ligação campo ↔ tag do XSD fica para a RFC-PEP-006 (gerador de XML).
- **Desvio 2, Tab 87 e Tab 63:** as abas `Tab 63` e `Tab 87 - Lista de terminologias` não têm o formato de terminologia e não entram. A 87 é a lista das terminologias TUSS (18, 19, 20, 22…), que moram na Spike (SPK-09); por isso o teste de integridade aceita a 87 como domínio citado e não carregado.
- **CADOP:** a versão é a data de publicação (`Last-Modified` do arquivo), a carga faz upsert das ativas e marca como `ativa = false` quem saiu do arquivo (nunca apaga). Só entra o cadastro da empresa (registro, razão social, nome fantasia, CNPJ, modalidade, UF); representante, telefone e e-mail do CADOP ficam de fora. A tabela `operadoras` mora em `src/modulos/contratos/schema.ts` (RFC-PEP-008).
- **Permissões:** `db:rls` revoga INSERT, UPDATE, DELETE e TRUNCATE de `app_rw` e `app_admin` em `ref_*` e `operadoras`; teste confirma.
- **Banco novo:** as tabelas nascem vazias. Rode os `ref:carregar` com os arquivos oficiais baixados (pacotes da ANS e `Relatorio_cadop.csv`) antes de usar o faturamento; a RFC-PEP-006 deve checar a carga e avisar quando faltar.
- **Licença dos XSD e da Tabela 38 (Q3):** resolvida em 14/11/2026, ver tabela de perguntas. Fatias 4 a 6 (defasagem, `GET /referencias/status`, competência da Spike) seguem como estão.

