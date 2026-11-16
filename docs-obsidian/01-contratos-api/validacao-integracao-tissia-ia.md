---
projeto: tissia-ia
tipo: contrato-api
status: ativo
tags:
  - tissia
  - tissia-ia
  - contrato-api
  - validacao
  - integracao
aliases:
  - validacao_spike
  - validacao_tissia_ai
origem: tissia-ai/docs/validacao_spike.md
---

# Tissia IA — Roteiro de Validação e Alinhamento de Piloto

> **Objetivo deste documento:** Coletar insumos técnicos e operacionais junto às equipes de engenharia dos sistemas parceiros (PEP / faturamento) e aos gestores de clínicas piloto para calibrar o início da operação assistida.

---

## 1. Desenho Recomendado do Piloto (30 Dias)

Para comprovar o retorno financeiro (ROI) e a redução de glosas sem impactar a rotina da clínica no primeiro dia, propomos uma implantação assistida em duas etapas de 15 dias:

| Etapa | Duração | Funcionamento | Métricas Aferidas |
| :--- | :--- | :--- | :--- |
| **Etapa 1 — Operação Silenciosa (Baseline)** | Dias 1 a 15 | O sistema integrador chama `POST /v1/guias/analisar` no fechamento das guias e grava o `analise_id`, sem exibir alertas na tela. | Volume de guias por operadora, índice de glosa de linha de base e percentual de glosas que já haviam sido sinalizadas preventivamente pela API. |
| **Etapa 2 — Operação Assistida (Intervenção)** | Dias 16 a 30 | Os alertas, o checklist de DUT, a justificativa SP/SADT e as sugestões de códigos passam a ser exibidos para médicos e faturistas. | Taxa de aceitação/edição das sugestões, taxa de autocorreção da guia na origem, tempo médio de revisão e redução financeira de glosas em R$. |

---

## 2. Questionário de Alinhamento Técnico (Engenharia do Sistema Integrador)

1. **Q1 — Ponto de Acionamento no Fluxo da Guia:**
   No fluxo atual do sistema de vocês, qual é o melhor momento para acionar `POST /v1/guias/analisar`:
   - **(a)** No encerramento do atendimento clínico pelo profissional de saúde;
   - **(b)** Na tela de conferência/faturamento da guia pela pessoa faturista; ou
   - **(c)** Em ambos os momentos?
2. **Q2 — Disponibilidade da Evolução Clínica na Conferência:**
   Quando a pessoa faturista revisa uma guia SP/SADT antes de gerar o lote TISS, o backend do sistema tem acesso imediato ao texto da evolução clínica daquele atendimento para enviá-lo no campo `evolucao_clinica` (mesmo que o faturista tenha visão restrita na interface)?
3. **Q3 — Identificador Único de Guia (`guia_ref`):**
   O sistema possui um identificador interno único e estável para a guia desde a sua criação até o retorno do demonstrativo da operadora, permitindo rastrear revisões sucessivas da mesma guia?
4. **Q4 — Conciliação do Retorno de Faturamento / Glosas:**
   Para enviar o resultado real de pagamento ou glosa das operadoras ao Tissia IA, qual formato é mais simples para a arquitetura atual de vocês:
   - **(a)** Enviar o payload JSON estruturado em `POST /v1/desfechos`; ou
   - **(b)** Enviar diretamente o arquivo XML TISS do demonstrativo de retorno em `POST /v1/tiss/lotes/ingerir`?
5. **Q5 — Conciliação de Atendimentos Realizados vs. Guias Emitidas (`POST /v1/guias/esquecidas`):**
   Os campos propostos para identificar atendimentos realizados na agenda sem guia correspondente (`atendimento_ref`, `guia_ref`, `paciente_ref`, `registro_ans`, `data_atendimento`, `codigo_tuss`, `valor_estimado`) estão disponíveis na estrutura de agenda do sistema?

---

## 3. Questionário de Alinhamento Clínico e Operacional (Clínicas Piloto)

6. **Q6 — Operadoras e Procedimentos Prioritários do Piloto:**
   Quais são as **2 a 3 operadoras de saúde** (código ANS) de maior volume financeiro na clínica piloto e os **10 procedimentos TUSS** que mais sofrem glosa ou exigência de justificativa técnica hoje?
7. **Q7 — Estrutura da Justificativa Clínica SP/SADT (Campo 23 TISS):**
   O formato de justificativa clínica estruturado em 3 blocos (*Quadro clínico* + *Exames/tratamentos prévios* + *Indicação e CID-10*) dentro do limite de 500 caracteres atende às exigências dos auditores das operadoras com que a clínica trabalha?
8. **Q8 — Tabelas Contratuais e Autorização Prévia (`POST /v1/contratos/auditar`):**
   A clínica possui tabelas negociadas vigentes (valores acordados por código TUSS e lista de procedimentos que exigem senha/autorização prévia) disponíveis em planilha ou no sistema para configurarmos a auditoria contratual automática?

