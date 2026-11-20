---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
  - prontuario
  - faturamento
  - guia-tiss
aliases:
  - RFC-PEP-023
  - Faturamento clínico execução vs solicitação e retorno
---

# RFC-PEP-023 — Separação Solicitação Externa × Execução na Unidade e Janela de Retorno no Fechamento

**Fase:** P2 (alinhamento clínico-financeiro pós-P1) · **Status:** Rascunho · **ADRs:** [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]], [[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028]], [[ADR-PEP-0029-consulta-tiss-e-preco-de-contrato-na-guia|ADR-PEP-0029]], [[ADR-PEP-0030-separacao-pedido-externo-execucao-local-e-retorno|ADR-PEP-0030]]
**Telas:** 05 (Prontuário — pedidos de exame/procedimento), 08 (Documentos — guia de solicitação impressa), 16/17 (Mesa e Editor — apenas itens faturáveis da clínica), 21 (Contratos — `dias_retorno`)
**Requisitos:** RF-PRO-06, RF-DOC-02, RF-FAT-01, RF-FAT-13, RF-CON-01
**Depende de:** [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-008-contratos|RFC-PEP-008]], [[RFC-PEP-009-receita-esquecida|RFC-PEP-009]]

## Objetivo

Corrigir duas premissas do fechamento automático de guias consolidadas após `tissia#31` e `ADR-PEP-0029`:
1. Evitar que exames solicitados para realização **fora da clínica** (`solicitacao_externa`) sejam inseridos na guia `sp_sadt` faturável da clínica, separando-os dos procedimentos **executados na unidade** (`execucao_unidade`).
2. Evitar cobrança automática indevida em atendimentos de **retorno** realizados dentro da janela contratual (`dias_retorno`).

## Estado atual (após `tissia#31` e `feat/guia-correcoes-int`)

- `ProntuarioService.fechar` envia **todos** os registros de `pedidos_exame` para `FaturamentoService.gerarGuiasDoAtendimento`.
- Conforme `ADR-PEP-0029`, se houver pedidos, nasce ou uma `SP/SADT` única (`TUSS_CONSULTA_CONSULTORIO` + todos os pedidos) ou `Guia de Consulta` + `SP/SADT` com todos os pedidos.
- Não há distinção de destino (`solicitacao_externa` × `execucao_unidade`), nem verificação de `agendamentos.tipo === 'retorno'` contra a janela do contrato.

## Escopo

**Dentro**
- **Fatia 1 — Destino do pedido (`solicitacao_externa` × `execucao_unidade`):**
  - Migração aditiva em `pedidos_exame`: `destino text NOT NULL DEFAULT 'solicitacao_externa' CHECK (destino IN ('solicitacao_externa', 'execucao_unidade'))`.
  - Contratos aditivos em `@pep/contracts` (`atendimento.ts`): `DestinoPedidoSchema = z.enum(['solicitacao_externa', 'execucao_unidade'])`, campo opcional `destino` em `CriarPedidoExameSchema` (default `'solicitacao_externa'`) e campo `destino` em `PedidoExameSchema`.
  - UI em `Atendimento.tsx` (e `Revisao.tsx`): ao adicionar exame/procedimento, toggle segmentado simples: **"Solicitação externa (paciente faz fora)"** vs **"Executado aqui na clínica"**.
  - `ProntuarioService.fechar`:
    - Filtra apenas `pedidos.filter(p => p.destino === 'execucao_unidade')` para compor a `sp_sadt` faturável enviada a `FaturamentoService.gerarGuiasDoAtendimento`.
    - Se houver apenas pedidos `solicitacao_externa` em atendimento de convênio (e a consulta for cobrável): gera **somente a Guia de Consulta** para faturamento da clínica, além do documento/guia de solicitação externa para impressão pelo paciente (`RF-FAT-13`).
    - Análise da Spike (`POST /atendimentos/:id/analisar` e `/sugestoes-ia`) continua avaliando também os pedidos `solicitacao_externa` para apoiar o médico com Campo 23 e DUT sem erros, registrando em metadados que a guia faturável da clínica conterá apenas os itens `execucao_unidade`.
- **Fatia 2 — Regra de janela de retorno:**
  - Migração aditiva em `contratos`: `dias_retorno smallint CHECK (dias_retorno IS NULL OR dias_retorno BETWEEN 0 AND 180)`.
  - Campo configurável em `Contratos.tsx` e `NovoContrato.tsx` ("Janela de retorno sem cobrança (dias)").
  - Ao fechar atendimento vinculado a `agendamentos.tipo === 'retorno'`:
    - Consulta o contrato vigente na data do atendimento (mesmo critério de `ADR-PEP-0028`/`0029`) para obter `dias_retorno` (ou *fallback* da unidade).
    - Verifica se há atendimento fechado anterior (`agendamentos.tipo IN ('consulta', 'procedimento')` ou sem marcação de retorno) do mesmo `paciente_id` + `profissional_id` dentro do intervalo `[data_atendimento - dias_retorno, data_atendimento)`.
    - Se estiver **dentro da janela** (e o médico não marcou `cobrarConsultaRetorno: true`): **não** gera Guia de Consulta nem inclui `TUSS_CONSULTA_CONSULTORIO` na SP/SADT (e, no particular, não cria recebível de consulta). Se houver itens `execucao_unidade`, gera apenas a SP/SADT desses procedimentos.

**Fora**
- Integração eletrônica direta com laboratórios/centros de imagem externos (P3, `RFC-PEP-016`).
- Lançamento de materiais/medicamentos de sala (Tabela 19/20 TISS) — permanece fora até decisão específica em `RFC-PEP-009` Q1.

## Design — Matriz de Geração de Guias no Fechamento (Convênio)

| Tipo de atendimento | Pedidos `execucao_unidade` | Contrato (`consulta_em_guia_separada`) | Guias faturáveis geradas para a clínica |
| :--- | :---: | :---: | :--- |
| Consulta (ou retorno fora da janela) | 0 (mesmo com pedidos externos) | — | **1 Guia de Consulta** (`10101012`) |
| Consulta (ou retorno fora da janela) | ≥ 1 | `false` | **1 SP/SADT** (`10101012` + procedimentos locais) |
| Consulta (ou retorno fora da janela) | ≥ 1 | `true` | **1 Guia de Consulta** + **1 SP/SADT** (procedimentos locais) |
| Retorno dentro da janela | 0 | — | **Nenhuma guia faturável** (atendimento fechado sem cobrança) |
| Retorno dentro da janela | ≥ 1 | — | **1 SP/SADT** (apenas os procedimentos locais, sem `10101012`) |

> Pedidos com `destino = 'solicitacao_externa'` geram documentos de pedido médico / Guia SP/SADT impressa de solicitação ao executante externo, mas nunca entram na tabela `guias` faturáveis da clínica.

## Impacto em `Receita Esquecida` (`RFC-PEP-009`)

- Atendimentos de retorno fechados dentro da janela sem guia de consulta passam a ser reconhecidos como regulares (não entram como `sem_guia`).
- Atendimentos com pedidos exclusivamente `solicitacao_externa` e Guia de Consulta emitida passam a contar como cobrança completa.

## Critérios de aceite

- [ ] Consulta de convênio com 1 exame marcado como `solicitacao_externa` gera apenas 1 Guia de Consulta faturável (`10101012`) e permite imprimir a solicitação externa com Campo 23.
- [ ] Consulta de convênio com 1 procedimento `execucao_unidade` + 1 exame `solicitacao_externa` fatura apenas a consulta + o procedimento local (respeitando `consulta_em_guia_separada`).
- [ ] Atendimento de `retorno` em até `dias_retorno` dias de consulta anterior com o mesmo profissional fecha sem criar Guia de Consulta (nem recebível particular de consulta), salvo marcação explícita de cobrança.
- [ ] Retrocompatibilidade: `POST /api/v1/atendimentos/:id/pedidos` sem campo `destino` continua funcionando com default `'solicitacao_externa'`.
