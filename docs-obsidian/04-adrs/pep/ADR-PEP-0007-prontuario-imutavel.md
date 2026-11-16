---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0007-prontuario-imutavel.md
---

# ADR-PEP-0007 — Prontuário imutável: versões e adendos

**Status:** Aceita — 07/11/2026

## Contexto

Prontuário é documento médico-legal: o que foi registrado e assinado não pode ser alterado sem rastro (Res. CFM nº 1.638/2002 **[validar]**; requisitos SBIS NGS1/NGS2 **[validar]**). O design mostra "Salvo automaticamente 09:42 · versão 3" (tela 05) e a trilha de revisão da IA (tela 07): cada trecho marcado como humano ou IA, e o que foi aceito, editado ou rejeitado.

## Decisão

- **Antes de finalizar:** a evolução é um rascunho com autosave; cada gravação gera uma **versão** nova (`evolucao_versao`), append-only, com autor, data/hora, origem (`digitado | voz | ia_aceita | ia_editada`) e hash do conteúdo. Evento `EvolucaoVersionada`.
- **Ao finalizar** (`AtendimentoFechado`) e, quando houver, ao assinar ([[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]]): a última versão vira definitiva e é **somente leitura**.
- **Depois disso, só adendo:** registro novo ligado à evolução, com autor, data, justificativa obrigatória e, se aplicável, nova assinatura. Nunca `UPDATE` no conteúdo.
- **Banco:** tabelas de versões e adendos sem `UPDATE`/`DELETE` para o papel da aplicação (permissões + trigger que rejeita). Exclusão por direito do titular segue a política de retenção legal do prontuário, não apaga histórico clínico **[validar prazo de guarda]**.
- **Autoria de trechos de IA:** cada bloco inserido a partir de sugestão guarda `analise_id`/sugestão de origem e a decisão do profissional.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Registro editável com log de auditoria separado | O log pode divergir do dado; imutabilidade no próprio modelo é mais simples de provar |
| Event sourcing completo do domínio clínico | Poder demais para a necessidade; versões append-only bastam |

## Consequências

- Leitura "atual" = última versão; consultas usam visão materializada ou coluna `versao_atual_id`.
- Offline: conflito de versão vira adendo, nunca sobrescrita ([[ADR-PEP-0015-offline-prontuario|ADR-PEP-0015]]).
- Detalhes na [[RFC-PEP-004-prontuario|RFC-PEP-004]] e no [[modelo-de-dados-pep|modelo de dados]].
