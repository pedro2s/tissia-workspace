---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - referencias
  - ans
  - cadop
  - anvisa
  - cmed
  - medicamentos
aliases:
  - Carga de tabelas de apoio
  - Carga de referências ANS e Anvisa
  - ref:carregar
---

# Carga de Tabelas de Apoio (ANS, CADOP e Anvisa/CMED)

Guia operacional passo a passo para popular e atualizar todas as bases globais de referência do `@pep/api` (`ref_*` e `operadoras`).

> **Referências arquiteturais:** [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]] · [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021]] · [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024]] · [[demo-dados-ficticios|Demo de dados fictícios]]. Hub: [[hub-pep-tissia]].

---

## 1. Mapa das Bases de Apoio

As tabelas abaixo são **globais** (sem `tenant_id`, somente leitura para `app_rw`), populadas via `pnpm --filter @pep/api ref:carregar <base> <arquivo>` (`apps/api/scripts/ref/carregar.ts`). Cada carga fica auditada em `ref_cargas` com `sha256` e diff contra a versão anterior.

| Base (`ref:carregar`) | Tabela / Destino | Origem oficial | Carregada no `demo:reset`? |
| :--- | :--- | :--- | :--- |
| `dominios_tiss` | `ref_dominios_tiss` | ANS — Padrão TISS, Representação de Conceitos em Saúde (`.zip`, inclui Tabela 38 e demais terminologias) | Sim (`scripts/demo-refs.sh`) |
| `campos_guia` | `ref_campos_guia` | ANS — Padrão TISS, Componente de Conteúdo e Estrutura (`.zip`) | Sim (`scripts/demo-refs.sh`) |
| `operadoras` | `operadoras` | ANS — Dados Abertos, CADOP (`Relatorio_cadop.csv`) | **Não** (seed cria só 4 operadoras fictícias) |
| `medicamentos` | `ref_medicamentos` | Anvisa / CMED (`TA_PRECO_MEDICAMENTO.csv` ou `DADOS_ABERTOS_MEDICAMENTOS.csv`) | **Não** |
| `tiss_xsd` | `apps/api/ref/tiss/<versao>/` | ANS — Padrão TISS, Componente de Comunicação (`.zip`) | Já versionado no repositório |

> [!NOTE]
> Os catálogos de **TUSS (Tabela 22)** e **CID-10** não ficam no banco do `@pep/api` — eles pertencem ao motor `tissia-ia` (ver [[hub-tissia-ia]] e [[SPK-09-catalogo-tuss|SPK-09]]).

---

## 2. Pré-requisitos

1. Banco PostgreSQL rodando e migrado:
   ```bash
   pnpm --filter @pep/api infra:up
   pnpm --filter @pep/api db:setup
   ```
2. Variável `MIGRATION_DATABASE_URL` configurada em `apps/api/.env` (ou padrão local `postgres://app_owner:dev@localhost:55442/pep`).
3. Pacotes oficiais disponíveis localmente (por padrão em `~/dev/pedro/tissia-ai-tunning/`).

---

## 3. Passo a Passo de Carga Completa

Execute todos os comandos a partir da raiz do repositório `tissia` (`/home/pedro/dev/pedro/tissia`).

### 3.1 Domínios TISS (Tabela 38 e terminologias) e Campos das Guias

```bash
pnpm --filter @pep/api ref:carregar dominios_tiss \
  /home/pedro/dev/pedro/tissia-ai-tunning/padrao_tiss_setembro_2026/Padrao_TISS_Representacao_de_Conceitos_em_Saude_202609.zip

pnpm --filter @pep/api ref:carregar campos_guia \
  /home/pedro/dev/pedro/tissia-ai-tunning/padrao_tiss_setembro_2026/PadroTISS_ComponentedeContedoeEstrutura_202511.zip
```

### 3.2 Medicamentos (Anvisa / CMED)

```bash
pnpm --filter @pep/api ref:carregar medicamentos \
  /home/pedro/dev/pedro/tissia-ai-tunning/anvisa-cmed/TA_PRECO_MEDICAMENTO.csv \
  --versao 2026-11
```

- O parser (`apps/api/scripts/ref/medicamentos.ts`) aceita separador `;` com ou sem acento no cabeçalho (compatível com `TA_PRECO_MEDICAMENTO.csv` e `DADOS_ABERTOS_MEDICAMENTOS.csv`).
- A carga é idempotente por `versao_ref` + `sha256` em `ref_cargas`. Para subir uma publicação mais recente da CMED, informe uma nova `--versao` (ex.: `--versao 2026-12`).

### 3.3 Operadoras de Saúde (CADOP ANS)

Baixe o CSV oficial publicado pela ANS e carregue na tabela `operadoras`:

```bash
curl -fsSL \
  https://dadosabertos.ans.gov.br/FTP/PDA/operadoras_de_plano_de_saude_ativas/Relatorio_cadop.csv \
  -o /tmp/Relatorio_cadop.csv

# Quando o SHA-256 coincidir com apps/api/ref/MANIFEST.yaml:
pnpm --filter @pep/api ref:carregar operadoras /tmp/Relatorio_cadop.csv

# Quando a ANS tiver publicado arquivo mais novo que o manifesto:
pnpm --filter @pep/api ref:carregar operadoras /tmp/Relatorio_cadop.csv \
  --nova-versao --versao "$(date +%F)"
```

> [!TIP]
> Se você usou `--nova-versao` apenas para popular o ambiente local/demo e **não** pretende atualizar o manifesto em PR, restaure o arquivo versionado ao final:
> ```bash
> git checkout -- apps/api/ref/MANIFEST.yaml
> ```
> Em ambiente de demonstração (`VITE_AMBIENTE_DEMO=true`), a UI continua mascarando os nomes das 4 operadoras da demo mesmo com o CADOP completo carregado (ver [[demo-dados-ficticios]]).

### 3.4 Schemas XSD TISS (somente quando a ANS publicar nova versão)

Os XSDs vigentes (`4.02.00` e `4.03.00`) já ficam versionados em `apps/api/ref/tiss/`. Só rode o comando abaixo ao incorporar uma nova versão oficial do Padrão TISS:

```bash
pnpm --filter @pep/api ref:carregar tiss_xsd \
  /caminho/para/PadroTISSComunicao_AAAAMM.zip \
  --nova-versao --versao 4.03.00 --competencia 202511
```

---

## 4. Verificação da Carga

Confirme as versões e contagens gravadas no PostgreSQL:

```bash
psql "postgresql://app_owner:dev@localhost:55442/pep" -c "
SELECT base, versao_ref, contagens, carregado_em
FROM ref_cargas
ORDER BY base;

SELECT count(*) AS operadoras_total,
       count(*) FILTER (WHERE ativa) AS operadoras_ativas
FROM operadoras;

SELECT count(*) AS ref_medicamentos FROM ref_medicamentos;
"
```

Referência esperada após carga completa:
- `dominios_tiss`: ~1.263 registros
- `campos_guia`: ~767 registros
- `operadoras`: ~1.100 operadoras ativas
- `ref_medicamentos`: ~26.634 apresentações CMED

---

## 5. Relação com `pnpm demo:reset`

- `pnpm --filter @pep/api demo:reset` executa `db:setup && bash scripts/demo-refs.sh && pnpm seed:demo`.
- Ele **não apaga** `ref_dominios_tiss`, `ref_campos_guia`, `ref_medicamentos`, `ref_cargas` nem `operadoras` — apenas limpa os dados do tenant de demonstração.
- Em um banco recém-criado (ex.: após `docker compose down -v`), basta rodar `demo:reset` e, em seguida, as etapas **3.2 (medicamentos)** e **3.3 (CADOP)** acima.
