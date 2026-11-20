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
  - faturamento
  - guia-tiss
aliases:
  - ADR-PEP-0030
  - Separação pedido externo vs execução local e janela de retorno
---

# ADR-PEP-0030 — Separação entre solicitação externa e execução na unidade, e regra de janela de retorno no fechamento

**Status:** Aceita — 20/11/2026 (aceita pelo dono do produto após o merge de `pedro2s/tissia#31` e `pedro2s/tissia-workspace#2`, habilitando a [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023]]; a partir daqui é imutável: mudanças só por nova ADR)
**Relacionadas:** [[ADR-PEP-0014-convenio-e-particular-no-mvp|ADR-PEP-0014]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]], [[ADR-PEP-0029-consulta-tiss-e-preco-de-contrato-na-guia|ADR-PEP-0029]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-008-contratos|RFC-PEP-008]], [[RFC-PEP-009-receita-esquecida|RFC-PEP-009]], [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023]]

## Contexto

Com `tissia#31` e a suíte de correções (`ADR-PEP-0027`, `ADR-PEP-0028`, `ADR-PEP-0029` em `tissia-workspace#2`), o fechamento do atendimento de convênio passou a garantir que a consulta nunca fique sem cobrança (em SP/SADT única ou em Guia de Consulta separada conforme `contratos.consulta_em_guia_separada`).

Entretanto, duas distorções entre o modelo atual e a operação real de uma clínica permanecem abertas no fechamento do atendimento (`ProntuarioService.fechar` → `FaturamentoService.gerarGuiasDoAtendimento`):

1. **Todo pedido de exame entra como item faturável da própria clínica.** Hoje `pedidos_exame` não distingue se o médico está *solicitando* um exame para o paciente realizar em um serviço diagnóstico externo (ex.: ressonância magnética ou hemograma em laboratório terceiro) ou registrando um procedimento/exame *executado na própria clínica* (ex.: infiltração, ECG, biópsia em sala). Como consequência, todo pedido acaba incluído na guia SP/SADT faturável da clínica.
2. **Atendimento do tipo `retorno` gera cobrança automática de consulta.** Agendamentos marcados como `retorno` em convênio (e atendimentos particulares em janela de retorno) não verificam a janela contratual de retorno sem cobrança (`dias_retorno`), gerando guias de consulta destinadas à glosa por duplicidade/prazo de retorno.

## Decisão

1. **Destino explícito no pedido de exame/procedimento (`pedidos_exame.destino`).**
   - Adicionar de forma estritamente aditiva ([[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]) a coluna `destino text NOT NULL DEFAULT 'solicitacao_externa' CHECK (destino IN ('solicitacao_externa', 'execucao_unidade'))` em `pedidos_exame` (e campo opcional correspondente em `@pep/contracts`, mantendo compatibilidade retroativa com `DEFAULT 'solicitacao_externa'` ou regra explícita por catálogo/UI).
   - **`solicitacao_externa`:** gera documento clínico de solicitação / Guia SP/SADT de solicitação para impressão (`RF-FAT-13`), continua participando da análise proativa e anti-glosa da Spike para qualificar o Campo 23 e os critérios de DUT (ajudando o paciente e o executante externo a não sofrerem negativa), mas **não** entra como item faturável na guia executante da própria clínica.
   - **`execucao_unidade`:** representa procedimento ou exame realizado pela própria unidade no atendimento (ou agendado para execução na clínica) e **entra** na composição da guia `sp_sadt` faturável da clínica, seguindo as regras de `ADR-PEP-0029` (junto à consulta na SP/SADT única ou em SP/SADT separada).
2. **Regra de janela de retorno no fechamento (`agendamentos.tipo = 'retorno'`).**
   - Adicionar `dias_retorno smallint` (opcional, `NULL` = sem regra no contrato) em `contratos` (previsto na [[RFC-PEP-008-contratos|RFC-PEP-008]] e [[RFC-PEP-009-receita-esquecida|RFC-PEP-009]]), com *fallback* configurável por unidade (`dias_retorno_padrao`, padrão sugerido `15` dias **[validar com piloto]**).
   - Ao fechar um atendimento cuja origem na agenda tenha `tipo = 'retorno'`, o PEP verifica o último atendimento fechado do mesmo paciente com o mesmo profissional (ou especialidade, conforme política da unidade):
     - **Dentro da janela:** **não** gera Guia de Consulta nem item `TUSS_CONSULTA_CONSULTORIO` (nem recebível particular de consulta). Se houver procedimentos com `destino = 'execucao_unidade'`, fatura apenas esses procedimentos em SP/SADT.
     - **Fora da janela** (ou sem consulta anterior na janela): comporta-se como consulta cobrável normal.
   - O médico pode sinalizar exceção explícita no fechamento ("Cobrar consulta mesmo sendo retorno" com justificativa curta, ex.: nova queixa aguda não relacionada).
3. **Alinhamento com Receita Esquecida (`RFC-PEP-009`).**
   - `RFC-PEP-009` passa a considerar como atendimentos "com cobrança completa" retornos fechados dentro da janela sem guia de consulta e pedidos `solicitacao_externa` sem item em guia da clínica, eliminando falsos positivos.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Inferir local vs. externo apenas por lista estática `tuss_executaveis` na unidade | Um mesmo TUSS (ex.: ECG ou ultrassonografia) pode ser feito na clínica em um caso e solicitado para fora em outro por falta de agenda/equipamento; o médico precisa escolher no ato |
| Deixar o faturista excluir manualmente os exames externos da SP/SADT na Mesa | O faturista não tem visibilidade clínica se o exame foi feito na sala ou pedido para fora, aumentando retrabalho e risco de faturamento indevido (fraude/glosa) |
| Tratar retorno apenas no job noturno de Receita Esquecida (`RFC-PEP-009`) | A guia indevida já teria nascido na Mesa do Faturista; prevenir na origem (`fechar`) evita ruído na fila e glosa |

## Consequências

- **Fidelidade à clínica real:** consultas com solicitação de exames externos passam a gerar apenas a Guia de Consulta faturável da clínica + o Pedido/Guia impressa de solicitação externa para o paciente.
- **Impacto no legado:** registros anteriores em `pedidos_exame` recebem `DEFAULT 'solicitacao_externa'` na migração aditiva; guias já criadas antes desta ADR não são reescritas.
- **Impacto no `SpikeAdapter`:** mínimo ou nulo na API `v1` da `tissia-ia` — o PEP pode continuar enviando pedidos externos para checagem de DUT/Campo 23 quando o médico desejar validar a solicitação, diferenciando apenas no PEP o que vira guia faturável da unidade.
