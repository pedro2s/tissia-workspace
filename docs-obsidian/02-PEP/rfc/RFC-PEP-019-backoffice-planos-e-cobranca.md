---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-019-backoffice-planos-e-cobranca.md
---

# RFC-PEP-019 — Backoffice: planos, assinaturas e cobrança

**Fase:** B1 (P2, gatilho: primeira clínica pagante fora do piloto) · **Status:** Rascunho · **ADRs:** ADR-PEP-0021, 0022
**Telas do backoffice:** 05 (planos e cobrança), 04 (aba Cobrança), 02 (bloco Negócio)
**Requisitos:** RF-BKO-10, 11 ([[PRD-003-backoffice|PRD-003]])
**Depende de:** RFC-PEP-018

## Objetivo

Cobrar clínicas sem planilha: plano versionado, assinatura por tenant, faturas e régua de cobrança que nunca bloqueia atendimento.

## Escopo

- Planos versionados (`planos`, `planos_versoes`): preço por médico ativo/mês, teste gratuito, desconto anual, módulos incluídos (mapeados às flags `modulos` do release plan §5). Nova versão vale só para assinaturas novas; as vigentes mantêm o preço até a renovação.
- `assinaturas` por tenant (plano_versao, ciclo, início, renovação, situação) e `faturas` (competência, médicos faturados, valor, vencimento, forma, situação).
- Contagem de médicos ativos na competência a partir de `uso_diario` (RFC-PEP-018).
- Régua: aviso no PEP em D+5, bloqueio de **novos usuários** em D+30. Prontuário, atendimento e IA nunca são bloqueados.
- KPIs: MRR, ARR, cancelamento mensal, em atraso e margem após custo de IA (custo vindo da `/ops/v1` da Spike na B2; até lá, "—").

## Fora

- Emissão de NFS-e e gateway (boleto, PIX, cartão): avaliar um provedor (Asaas, Iugu, Stripe BR) **[validar]**. O backoffice guarda o id externo e recebe a situação por webhook.
- Cobrança de integradores da Spike (PRD-002; SPK-05 `uso_mensal`).

## Perguntas abertas

1. Os planos da tela 05 (Pro R$ 497, Clínica Blindada R$ 797, Módulo IA R$ 397–697) são os preços do PEP? O PRD-001 não define preço **[validar]**.
2. "Médico ativo" = profissional com ao menos um atendimento fechado na competência?
3. Teste gratuito = período em modo silencioso (30 dias, release plan §6)?
4. Provedor de cobrança: integrado já na B1 ou manual com o registro só no backoffice?
