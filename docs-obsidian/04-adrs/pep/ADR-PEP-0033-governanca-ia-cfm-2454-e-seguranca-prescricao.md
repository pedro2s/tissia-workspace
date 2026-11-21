---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - inteligencia-artificial
  - prescricao
  - seguranca-clinica
  - consentimento
aliases:
  - ADR-PEP-0033
  - Governança IA CFM 2454 e segurança da prescrição
---

# ADR-PEP-0033 — Governança de IA, CFM nº 2.454/2026 e segurança da prescrição

**Status:** Proposta — 21/11/2026  
**Relacionadas:** [[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]], [[ADR-PEP-0013-asr-na-spike|ADR-PEP-0013]], [[ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai|ADR-PEP-0025]], [[RFC-PEP-005-spike-adapter|RFC-PEP-005]], [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]], [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]], [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia|RFC-PEP-026]], [[SPK-14-sugestao-de-codigos-e-indicacao-a-partir-da-evolucao|SPK-14]], [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas|SPK-15]]

## Contexto

A revisão de prescrição e dos fluxos assistidos por IA identificou riscos de falso negativo por comparação lexical de alergias, perda de estado clínico ao trocar de atendimento sem remontar editores, dificuldade de reconstruir qual modelo e prompt produziram uma sugestão, e aceitação em massa sem revisão suficiente. A escuta ambiental também precisa vincular gravação, consentimento, transcrição e revogação em um único fluxo verificável.

A referência a eventual Resolução CFM nº 2.454/2026 está **[validar]**: existência, vigência, texto e aplicação devem ser confirmadas pelo jurídico antes de declarar conformidade. Esta ADR define controles internos de segurança e rastreabilidade sem antecipar conclusão normativa.

## Decisão

1. **Prescrição segura.** É proibido mostrar estado verde “sem conflito” quando a avaliação se baseia somente em substring ou correspondência lexical (`conflitosAlergia`). A avaliação precisa considerar princípio ativo e classes terapêuticas com fontes versionadas e curadoria clínica; ausência de correspondência confiável resulta em “não avaliado” ou alerta, nunca em garantia de ausência de risco. Um cabeçalho clínico sticky deve estar visível nas telas de atendimento com nome, idade, peso, alergias em destaque e medicações em uso. Alertas críticos exigem justificativa clínica antes de override e geram evento auditável ligado à decisão.
2. **Receitas e backend.** `DocumentosService` valida classe e tipo documental no servidor. Até ativação dos controles e assinatura previstos em [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]], medicamentos sujeitos a controle especial e antimicrobianos não podem ser emitidos como receita simples. A interface não é fronteira de segurança.
3. **Isolamento de estado.** A árvore da tela de atendimento remonta por `atendimentoId` (`key={id}`), e a troca de ID limpa TipTap, refs, formulários, estados derivados e cache local antes de exibir o novo atendimento. Respostas assíncronas antigas não podem ser aplicadas ao atendimento corrente.
4. **Rastreabilidade da IA.** Persistir nome e versão exatos do modelo, versão do prompt e identificadores de correlação em cada análise/sessão. Sugestões anti-glosa, proativas e de Scribe/Escuta e suas decisões ficam em trilha append-only. Evolução assistida não é concluída sem `trilhaAnexada`; a trilha distingue sugestão, edição, aceite e rejeição. Aceite em massa exige revisão granular por item/achado e confirmação explícita antes da ação coletiva.
5. **Consentimento de escuta.** Unificar `consentimentos_gravacao` com `pacientes.consentimentos`, finalidade `escuta_ambiental`, versão integral do termo v2 e registro de concessão/revogação. A revogação fica disponível durante a gravação, interrompe a captura e inicia expurgo dos `transcricao_segmentos` associados conforme política documentada e auditável. Segmentos marcados `clinico: false` são descartados antes de persistir. O áudio não deve ser retido além do necessário ao processamento definido pela RFC.
6. **Integridade do contexto.** Remover conteúdo clínico fictício hardcoded, inclusive `MODELO_ORTOPEDIA_JOELHO`; `CONSULTA_EXEMPLO` em `Voz.tsx` só pode ser habilitada em ambiente de demonstração identificado no servidor. Produção rejeita flag e fixture de demonstração.

## Alternativas consideradas

| Alternativa | Motivo para rejeição |
| :--- | :--- |
| Manter comparação textual e exibir sucesso quando não houver coincidência | Confunde ausência de coincidência com ausência de risco clínico. |
| Registrar apenas o resultado final da IA | Impede reconstruir modelo, prompt, sugestão apresentada e decisão profissional. |
| Permitir aceite em massa sem revisão por item | Aumenta risco de rubber-stamping e de inserir achados não confirmados. |
| Manter consentimento separado por sessão sem vínculo com o cadastro | Fragmenta prova de finalidade e dificulta revogação efetiva. |
| Manter consulta fictícia como fallback geral | Pode inserir conteúdo não observado no prontuário real. |

## Consequências

- A classificação de princípio ativo e classe exige fonte autorizada, versionamento, curadoria clínica e tratamento explícito de desconhecidos; não se inventam códigos nem equivalências.
- A exigência de revisão aumenta interações na UI, mas preserva decisão clínica humana e autoria.
- Persistir metadados do modelo e prompt deve evitar armazenar texto clínico desnecessário nos registros de auditoria; conteúdo assistencial segue controles de acesso próprios.
- Revogação requer coordenação entre captura, fila de processamento, segmentos já persistidos e eventos de auditoria. A semântica de expurgo e retenção precisa ser compatibilizada com guarda legal e validada juridicamente.
- A decisão regulatória sobre CFM nº 2.454/2026 permanece **[validar]** e não constitui afirmação de conformidade legal.
