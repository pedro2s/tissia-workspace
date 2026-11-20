---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - identidade
  - agenda
  - financeiro
  - enfermagem
aliases:
  - ADR-PEP-0031
  - Separação de papéis check-out recepção e triagem
---

# ADR-PEP-0031 — Separação operacional entre Recepção (check-out financeiro), Enfermagem (triagem), Médico e Faturista

**Status:** Aceita — 20/11/2026 (aceita pelo dono do produto após o merge de `pedro2s/tissia#31` e `pedro2s/tissia-workspace#2`, habilitando a [[RFC-PEP-022-triagem-e-checkout-recepcao|RFC-PEP-022]]; a partir daqui é imutável: mudanças só por nova ADR)
**Relacionadas:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0014-convenio-e-particular-no-mvp|ADR-PEP-0014]], [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-003-agenda|RFC-PEP-003]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-010-financeiro-e-repasse|RFC-PEP-010]], [[RFC-PEP-022-triagem-e-checkout-recepcao|RFC-PEP-022]]

## Contexto

Na implementação atual (incluindo `tissia#31` e `RFC-PEP-010` Fatia 1), três fronteiras operacionais ainda misturam papéis dentro das telas do PEP em desacordo com a rotina de uma clínica:

1. **Cobrança particular dentro da tela do médico:** `RecebimentoParticular.tsx` é renderizado em `Atendimento.tsx` (e `Revisao.tsx`) logo após o fechamento da consulta. Embora protegido pela permissão `financeiro:receber_particular`, o fluxo posiciona a cobrança no espaço do consultório, em vez da fila de saída/check-out da recepção.
2. **Médico com permissão administrativa de edição de guia (`guias:escrever`):** em `packages/contracts/src/identidade.ts` (`PERMISSOES_POR_PERFIL`), o perfil `medico` possui `guias:escrever`. Na prática clínica, o médico deve gerenciar atos clínicos (evolução, CID, pedidos, Campo 23, aceite de sugestões de IA e adendos), não editar campos administrativos TISS (CNES, código de contratado, senha, transição na Mesa do Faturista).
3. **Ausência de etapa de Triagem / Pré-consulta para Enfermagem:** o perfil `enfermagem` existe em `PERMISSOES_POR_PERFIL` com `prontuario:escrever`, mas não há estrutura separada para sinais vitais, antropometria, revisão de alergias e medicações em uso antes de o médico abrir a evolução SOAP.

## Decisão

1. **Check-out operacional e financeiro na Recepção (não no consultório).**
   - Mover o fluxo principal de recebimento particular (`RecebimentoParticular`), emissão de recibo PDF, impressão/entrega de documentos assinados e agendamento de retorno para uma fila/painel de **Check-out** acessível pela **Agenda / Recepção** (módulos `agenda` e `financeiro`).
   - Na tela do médico (`Atendimento.tsx`), após fechar o atendimento particular, exibe-se apenas um resumo discreto ("Encaminhado para check-out na recepção", com status do recebimento se o usuário também tiver `financeiro:ler` ou perfil composto).
2. **Fronteira estrita de RBAC entre Médico e Faturista (`identidade.ts`).**
   - Remover `guias:escrever` do conjunto padrão do perfil `medico`, mantendo `guias:ler` (para acompanhar o status e o risco das guias ligadas aos seus atendimentos e no Histórico de Guias do paciente).
   - A transição inicial para a fila de conferência (`rascunho` → `em_revisao`) passa a ocorrer automaticamente ao fechar o atendimento quando a guia não possui pendências críticas de origem clínica, ou via ação clínica específica de correção de Campo 23 / adendo, deixando a edição de campos TISS, `em_revisao → validada` e envio em lote exclusivos de `faturista`.
   - Clínicas pequenas onde o próprio médico também atua como faturista continuam suportadas nativamente pela atribuição múltipla de perfis (`medico` + `faturista`) no vínculo da unidade (`RFC-PEP-001`).
3. **Módulo/etapa dedicada de Triagem (Pré-consulta).**
   - Introduzir registro estruturado de pré-consulta (`triagens`) no módulo `prontuario` (entre `chegou` e `em_atendimento`), preenchível por `enfermagem` (e `medico`), exibido no cabeçalho/painel de contexto do atendimento médico sem exigir criação prematura de versão SOAP médica.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Manter `guias:escrever` no perfil `medico` para ele clicar em "Enviar ao faturista" no Editor | Expõe campos administrativos TISS e revisão de guia ao médico; se o fechamento do atendimento já nasce com revisão 1 e dados resolvidos (`ADR-PEP-0028`), não há motivo para o médico entrar no Editor de Guia |
| Deixar `enfermagem` escrever diretamente na `evolucao_versoes` do médico | Mistura coleta de triagem (PA, FC, peso, temperatura) com o texto médico SOAP e gera conflitos de versão/autoria no atendimento |
| Manter cobrança particular apenas dentro de `Atendimento.tsx` | Obriga a recepcionista a navegar até a tela do prontuário fechado para receber e emitir recibo |

## Consequências

- **Separação limpa de responsabilidades (CFM/LGPD + produtividade):** cada persona opera sua própria bancada de trabalho — Recepção (Agenda + Check-in/Check-out + Particular), Enfermagem (Triagem), Médico (Prontuário + IA co-piloto) e Faturista (Mesa + Editor + Lotes + Retorno/Glosas).
- **Mudança de permissão padrão requer atualização de testes/E2E:** testes E2E ou scripts que usam apenas perfil `medico` para editar guia TISS na tela 17 precisarão usar o perfil `faturista` (ou vínculo `medico,faturista`), refletindo a operação real.
- **Nenhum impacto no contrato com a Spike (`tissia-ia`):** decisões de IA do médico continuam sendo enviadas pelo `SpikeAdapter` durante ou ao fechar o atendimento; dados de triagem podem enriquecer opcionalmente o texto enviado ao scrubber no futuro sem alterar o schema HTTP `v1`.
