---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - prontuario
  - guia-tiss
aliases:
  - ADR-PEP-0027
  - Indicação clínica única por atendimento
---

# ADR-PEP-0027 — Indicação clínica única por atendimento (Campo 23)

**Status:** Aceita — 19/11/2026 (implementada e validada no PR #31 do `tissia`, commit `31b9a66`; a partir do merge é imutável: mudanças só por nova ADR)
**Relacionadas:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]

## Contexto

Na rodada de testes manuais de 19/11/2026 ([[plano-guia-indicacao-unica-e-autopreenchimento|plano]]), identificou-se atrito direto entre o fluxo do médico (telas 05a–08) e o modelo de guia SP/SADT (tela 17):

- **Indicação repetida no prontuário:** a tabela `pedidos_exame` exigia `indicacao_clinica text NOT NULL` por item, e a UI (`Atendimento.tsx`, `Revisao.tsx`, `Documentos.tsx`) solicitava o preenchimento do texto a cada novo exame adicionado ao atendimento.
- **Campo 23 vazio na guia:** na guia TISS SP/SADT, o Campo 23 (*Indicação Clínica*) é **único por guia**, não por linha de procedimento. Apesar disso, `ProntuarioService.fechar` criava a revisão 1 em `guia_revisoes` com `indicacaoClinica: null`, deixando o texto apenas dentro de `itens[].indicacaoClinica` e obrigando o faturista a digitar o Campo 23 do zero.

## Decisão

1. **D1 — Indicação clínica única por atendimento (aditiva e compatível).**
   - A indicação clínica dos pedidos de exame passa a pertencer ao **atendimento**, armazenada em `atendimentos.indicacao_pedidos` (`text`, `CHECK` de 1 a 500 caracteres, adicionada na migração `0047_guia_indicacao_unica_e_cadastros_de_origem.sql`, já aplicada na fundação `feat/guia-fundacao`).
   - Em `@pep/contracts` (`packages/contracts/src/atendimento.ts`, já na fundação): `AtendimentoSchema` expõe `indicacaoPedidos: string | null`, `CriarPedidoExameSchema.indicacaoClinica` torna-se opcional e `DefinirIndicacaoPedidosSchema` valida o texto único (1 a 500 caracteres).
   - *(Implementado, fatia G1)*: a rota `PUT /api/v1/atendimentos/:id/indicacao-pedidos` define/atualiza a indicação do atendimento; ao criar pedido sem `indicacaoClinica`, ele herda `atendimentos.indicacao_pedidos`; editar um pedido atualiza a indicação do atendimento.
   - `pedidos_exame.indicacao_clinica` (`NOT NULL`) **não** é removida (migração estritamente aditiva, [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]) e passa a guardar uma **cópia** da indicação do atendimento para manter compatibilidade com leituras, impressões e registros anteriores.
   - **Sem *backfill* em SQL:** como o RLS é forçado até para o dono do schema ([[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]]), não há `UPDATE` retroativo na migração `0047`. A leitura em `ProntuarioService.montar` usa *fallback* `a.indicacaoPedidos ?? pedidos[0]?.indicacaoClinica ?? null` (já presente no commit-base `feat/guia-fundacao`).
2. **D2 — Campo 23 da revisão 1 nasce da indicação do atendimento.**
   - *(Implementado, fatia G1)*: no fechamento do atendimento (`ProntuarioService.fechar`), a revisão 1 da guia SP/SADT passa a nascer com `guia_revisoes.indicacao_clinica` preenchida a partir da indicação única do atendimento (com *fallback* para o legado).
   - O faturista continua podendo editar o Campo 23 no Editor de guia (tela 17), gerando uma nova revisão append-only com autor ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]).

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Manter indicação por pedido e deduplicar/concatenar na criação da guia | Mantém o atrito na UI do médico (pedir texto a cada exame), gera textos artificiais ao concatenar justificativas diferentes e não resolve a origem única do Campo 23 |
| Mover a indicação exclusivamente para a guia (ou remover `pedidos_exame.indicacao_clinica`) | Quebra a regra de migração só aditiva ([[ADR-PEP-0019-drizzle-kit-migracoes-e-seed\|ADR-PEP-0019]]), impede informar a indicação antes do fechamento (quando a guia ainda não existe) e prejudica atendimentos particulares ou análises antes do fechamento |

## Consequências

- **Compatibilidade preservada:** `pedidos_exame.indicacao_clinica` segue válida para pedidos existentes e impressões; o *fallback* `a.indicacaoPedidos ?? pedidos[0]?.indicacaoClinica ?? null` cobre atendimentos anteriores sem *backfill* SQL e sem violar o RLS forçado.
- **Fluxo clínico mais rápido (implementado, G1/G2):** o médico informa a indicação uma única vez por atendimento; adicionar o 2º ou 3º exame exige apenas código TUSS e quantidade.
- **Efeito no contrato com a Spike (`tissia-ia` API `v1`):** **nenhum**. Conforme verificado em `apps/api/src/modulos/tissia-ai/tissia-ai.service.ts` (`montarPayload`), a chamada `POST /v1/guias/analisar` continua enviando exatamente a chave `indicacao_clinica` (`string | null`, até 500 caracteres no PEP / 5.000 na Spike). Muda apenas a origem interna do texto no `ProntuarioService` (de `pedidos[0]?.indicacaoClinica` para a indicação do atendimento, confirmado em `ProntuarioService.analisar`).
