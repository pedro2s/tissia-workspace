---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - faturamento
  - guia-tiss
  - contratos
aliases:
  - ADR-PEP-0029
  - Consulta TISS e preço de contrato na guia
---

# ADR-PEP-0029 — Consulta sempre cobrada (SP/SADT única ou Guia de Consulta separada por contrato) e preço só pelo contrato

**Status:** Aceita — 20/11/2026 (implementada e validada no PR #31 do `tissia`; a partir do merge é imutável: mudanças só por nova ADR)
**Relacionadas:** [[ADR-PEP-0028-resolvedor-de-dados-da-guia|ADR-PEP-0028]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-008-contratos|RFC-PEP-008]], [[fluxo-guia-consulta-exame-e-estados|Fluxo da guia: consulta + exame e estados]]

## Contexto

Na análise manual de um atendimento de convênio com consulta + exame apareceram dois problemas:

1. **A consulta nunca era cobrada.** `ProntuarioService.fechar` criava UMA guia por atendimento (`guias.atendimento_id` único): sem pedido, a Guia de Consulta; com pedido, só a SP/SADT dos exames. Pelo padrão TISS a consulta nunca fica de fora e exame nunca entra na Guia de Consulta (ela só comporta 1 procedimento).
2. **Valor da guia divergente entre telas.** `guias.valor_total_centavos` somava só preço de contrato (item sem preço contava 0 → "R$ 0,00" na Mesa), enquanto o `EditorGuia` mostrava, sem preço, a tabela sintética do Marco D (`valorProcedimentoCentavos`, 19.000 para código fora da tabela), e o lote e o BI tinham seus próprios fallbacks sintéticos.

## Decisão

1. **Cenário por contrato** (decisão do dono do produto): `contratos.consulta_em_guia_separada boolean NOT NULL DEFAULT false`.
   - `false` (padrão, o mais comum): com exames, **UMA guia SP/SADT** com a consulta (TUSS `10101012`, quantidade 1) como primeira linha e os exames nas demais; o Tipo de atendimento (Tabela 50) padrão passa a ser o de consulta, achado pela **descrição** na base da ANS carregada (nunca inventado; sem achar vira pendência).
   - `true` (operadora que bloqueia código de consulta dentro da SP/SADT eletiva): **Guia de Consulta + SP/SADT só dos exames**, ligadas ao mesmo atendimento.
   - Sem pedido: só a Guia de Consulta (como antes). Particular não gera guia. Sem contrato vigente ou sem registro ANS: padrão `false`.
   - Implementação: `FaturamentoService.gerarGuiasDoAtendimento` (chamado por `ProntuarioService.fechar`, na mesma transação); `ContratosService.consultaEmGuiaSeparada` decide pelo contrato vigente na data do atendimento.
2. **Unicidade `(atendimento_id, tipo)`** no lugar de `(atendimento_id)` (migração 0048: cria o índice novo e remove o antigo; sem perda de dado, a regra nova é mais fraca). O fechamento segue idempotente (`onConflictDoNothing` com alvo `(atendimento_id, tipo)`; reexecutar não duplica nem perde a linha da consulta).
3. **Preço só pelo contrato, sem valor inventado.** Cada linha custa o preço vigente do contrato da operadora na data da guia (`precosFaturamento`). Item sem preço **não soma** e é contado em `guias.itens_sem_preco` (migração 0048): Mesa, histórico do paciente e editor mostram **"Sem preço no contrato"** (valor parcial identificado), a guia não conta como "pronta para lote" e o **lote exclui a guia com o motivo** (cadastrar o preço em Contratos). Removidos os fallbacks sintéticos do editor, da guia impressa (inclusive na demo), do lote e do BI (`valor_faturado` = valor da guia). Exceção documentada (`ponytail`): o detalhamento por item do BI de guia antiga da demo sem preço de contrato na data (`itens_sem_preco = 0`) ainda usa a tabela de referência.
4. **Demo:** o seed já cadastra preço de contrato para todos os TUSS da tabela do roteiro nas quatro operadoras; a Operadora B é a que pede a consulta em guia separada (A, C e D no padrão), para demonstrar os dois cenários.

## Consequências

- Contratos Zod só aditivos: `ContratoResumo.consultaEmGuiaSeparada`, `Criar/AtualizarContrato.consultaEmGuiaSeparada` (default `false`), `Atendimento.guias`, `GuiaDetalhe.outrasGuiasDoAtendimento`, `GuiaMesaItem.itensSemPreco`, `GuiaDoPaciente.itensSemPreco`, todos opcionais na leitura. Nada vai ao `SpikeAdapter`.
- A análise anti-glosa (espelho de risco) vale para a SP/SADT; com duas guias, a Guia de Consulta nasce sem risco.
- Quem usava `GuiaDetalhe.atendimentoId` como chave única da guia deve usar o `id` (dois registros por atendimento são possíveis no cenário 2).
