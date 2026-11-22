---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-020-backoffice-operacao-da-spike.md
---

# RFC-PEP-020 — Backoffice: operação da Spike

**Fase:** B2 (gatilho: SPK-06 aceita **e** mais de 10 clínicas na Spike **ou** primeiro integrador externo em produção) · **Status:** Implementada · **ADRs:** ADR-PEP-0021, 0023
**Telas do backoffice:** 02 (bloco Spike), 06 (integradores e chaves), 07 (observabilidade), 08 (explorador de análises), 09 (incidentes), 10 (modelos e roteamento), 11 (prompts), 12 (avaliações), 13 (bases), 14 (regras), 15 (DUT), 16 (feedback e desfechos), 17 (modelo de risco)
**Requisitos:** RF-BKO-12…16 ([[PRD-003-backoffice|PRD-003]])
**Depende de:** RFC-PEP-017; [[SPK-06-api-de-operacao|SPK-06]] (e SPK-05 para chaves) implementadas no `tissia-ai`

## Objetivo

Dar à equipe uma tela única para a saúde e o estado da Spike, sem acessar o banco dela e sem editar conteúdo fora do git (ADR-PEP-0023).

## Escopo por tela

| Tela | Modo | Fonte |
| :--- | :--- | :--- |
| 06 Integradores e chaves | Ler; rotacionar e revogar (revogar em prod com 2ª aprovação) | `/ops/v1/integradores`, `/ops/v1/chaves` (SPK-05/06) |
| 07 Observabilidade | Resumo (p95 por etapa, degradação, erros por código) + links | `/ops/v1/saude` + Cloud Monitoring |
| 08 Explorador | Ler por `analise_id`: metadados, tempos por etapa, alertas (id, tipo, severidade), feedback e desfecho | `/ops/v1/analises` |
| 09 Incidentes | Mostrar o incidente aberto e a linha do tempo | Status page / `/ops/v1/incidentes` **[validar ferramenta]** |
| 10 Modelos e roteamento | Ler; alterar o percentual entre provedores com 2ª aprovação | `/ops/v1/roteamento` |
| 11 Prompts | Ler versões, diff e estado por ambiente; promover canário **já aprovado no portão** | `/ops/v1/prompts` (git é a fonte) |
| 12 Avaliações | Ler resultados do harness e do portão | `/ops/v1/avaliacoes` |
| 13 Bases | Ler `versao_bases`, `MANIFEST` e diff; agendar a publicação de uma versão já mesclada | `/ops/v1/bases` |
| 14 Regras | Ler regras, casos de teste e impacto estimado; link para o PR | `/ops/v1/regras` |
| 15 DUT | Ler critérios e registrar a 1ª/2ª aprovação, que vira check no PR | `/ops/v1/dut/aprovacoes` |
| 16 Feedback e desfechos | Agregados; fila dos alertas mais rejeitados | `/ops/v1/feedback` |
| 17 Modelo de risco | Calibração e versões; modo sombra | `/ops/v1/risco` |

## Esboço de design

- O módulo `backoffice` tem um cliente `SpikeOperacaoClient` que repassa o token do operador (ADR-PEP-0023). Timeout de 5 s; falha da Spike → a tela mostra "indisponível", sem retry automático.
- Nada da Spike é persistido no PEP, salvo a trilha interna (ação e ids).
- Segunda aprovação: `aprovacoes_internas` (plataforma) com ação, payload hash, solicitante e aprovador ≠ solicitante; a ação só é enviada à Spike depois de aprovada.

## Perguntas abertas

1. A aprovação de DUT pelo backoffice registra um check no PR do GitHub (via app) ou só na Spike?
2. Ferramenta de incidentes e status page (Better Stack, Statuspage, própria)?
3. Esta RFC deveria morar no `tissia-ai`, se a Spike ganhar equipe própria (ADR-PEP-0023)?

## Notas de implementação (22/11/2026 — PR `pedro2s/tissia#41`)

- Migração `0063_onda_f_f4_backoffice_planos_ops.sql` (`bo_integrador_chaves`, `bo_incidentes`) e cliente `spike-operacao.client.ts`.
- Rotas `/bo/v1/ops/chaves` (`tsk_*` com hash SHA-256), `/bo/v1/ops/bases` (`MANIFEST.yaml`), `/bo/v1/ops/incidentes` e `/bo/v1/ops/observabilidade` conectadas às telas reais em `apps/backoffice`, com telas ilustrativas ocultas em produção.
