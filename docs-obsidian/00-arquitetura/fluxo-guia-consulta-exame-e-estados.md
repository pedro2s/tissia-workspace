---
projeto: PEP
tipo: arquitetura
status: em-implementacao
tags:
  - tissia
  - pep
  - faturamento
  - guia-tiss
  - fluxo
aliases:
  - Fluxo da guia: consulta + exame e estados
---

# Fluxo da guia: consulta + exame, preço do contrato e estados

Contexto e decisão: [[ADR-PEP-0029-consulta-tiss-e-preco-de-contrato-na-guia|ADR-PEP-0029]]. Máquina de estados e rotas: [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]. Contratos: [[RFC-PEP-008-contratos|RFC-PEP-008]].

## O que nasce no fechamento do atendimento de convênio

| Situação | Guias geradas | Itens |
| :--- | :--- | :--- |
| Sem pedido de exame | 1 Guia de **Consulta** | consulta (`10101012`) |
| Com exames, contrato com `consulta_em_guia_separada = false` (padrão) | 1 **SP/SADT** | consulta (linha 1) + exames; Tipo de atendimento (Tabela 50) de consulta |
| Com exames, contrato com `consulta_em_guia_separada = true` | 1 Guia de **Consulta** + 1 **SP/SADT** (mesmo atendimento e paciente) | consulta / só os exames |
| Particular | nenhuma | — |

O contrato vigente é o da operadora do beneficiário principal, para a unidade, na data do atendimento. Sem contrato: padrão (SP/SADT única). Cada guia tem tipo, status, número, valor, lote e preço de contrato próprios.

## Valor da guia

Cada linha = preço vigente do contrato da operadora (`precos_faturamento`) na data da guia (consulta: data do atendimento; SP/SADT: data da solicitação). Total = soma. Item sem preço: **"Sem preço no contrato"** (valor parcial e `itensSemPreco`), nunca R$ 0,00 silencioso nem preço sintético. Mesa, editor, histórico do paciente, lote (XML) e BI usam o mesmo valor; a guia sem preço não entra em lote (o motivo aparece na checagem do lote). A demo cadastra preços de contrato para todos os TUSS do roteiro.

## Estados e onde cada perfil enxerga

Fluxo: `rascunho` → `em_revisao` → `validada` → `em_lote` → `enviada` → (`paga` | `glosada_parcial` | `glosada_total`); `cancelada` é final. **Salvar** registra uma revisão e **não muda o estado** (rascunho continua rascunho); "Enviar ao faturista" leva a `em_revisao` (médico ou faturista); "Liberar para lote" (só faturista) leva a `validada`; a geração do lote leva a `em_lote`; o registro do envio leva a `enviada`; o retorno da operadora leva a paga/glosada.

| Estado | Mesa do faturista (fila) | Lotes | Histórico de guias do paciente | Quem vê |
| :--- | :---: | :---: | :---: | :--- |
| rascunho, em revisão, validada | sim | — | sim | médico, faturista, recepção, gestor (`guias:ler`; a Mesa é por unidade ativa) |
| em lote | não | sim | sim | idem |
| enviada, paga, glosada | não | sim (lote) / Glosas e Retorno | sim | idem |
| cancelada | não | — | sim | idem |

O editor mostra, abaixo do título, onde a guia está e a(s) guia(s) irmã(s) do mesmo atendimento. A Mesa é filtrada pela **unidade ativa** da sessão: quem entra em outra unidade não vê as guias da primeira (o Histórico do paciente não depende da unidade).
