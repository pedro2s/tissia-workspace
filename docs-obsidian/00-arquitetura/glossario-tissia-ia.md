---
projeto: tissia-ia
tipo: arquitetura
status: ativo
tags:
  - tissia
  - tissia-ia
  - arquitetura
  - glossario
aliases:
  - glossario
origem: tissia-ai/docs/glossario.md
---

# Glossário de domínio

| Termo | Significado |
| :--- | :--- |
| **ANS** | Agência Nacional de Saúde Suplementar. Regula operadoras de planos de saúde e publica TISS, TUSS e o Rol. |
| **Operadora** | Plano de saúde (Unimed, Bradesco Saúde, SulAmérica, Amil…). Identificada pelo **Registro ANS** (6 dígitos). |
| **Prestador** | Clínica, consultório ou hospital que atende e fatura contra a operadora. |
| **TISS** | Troca de Informação em Saúde Suplementar. Padrão obrigatório de troca de dados prestador ↔ operadora (guias, lotes XML, mensagens). Tem versões (ex.: 4.0x); a versão é configuração, não constante. |
| **TUSS** | Terminologia Unificada da Saúde Suplementar. Conjunto de tabelas de códigos do TISS. Tabela 22 = procedimentos e eventos; 18 = diárias/taxas; 19 = materiais; 20 = medicamentos; 38 = mensagens (motivos de glosa/negativa) **[validar numeração na versão vigente]**. |
| **Rol** | Rol de Procedimentos e Eventos em Saúde (RN ANS nº 465/2021 e atualizações). Lista de cobertura mínima obrigatória. |
| **DUT** | Diretriz de Utilização. Anexo II do Rol: critérios clínicos que precisam estar presentes para cobertura obrigatória de certos procedimentos. Cada DUT tem número. |
| **Correlação TUSS × Rol** | Planilha publicada pela ANS que liga código TUSS ao item do Rol e indica se há DUT associada. |
| **CID-10** | Classificação Internacional de Doenças, 10ª revisão. É a usada no TISS. Tabelas oficiais no DATASUS (inclui restrição de sexo por subcategoria). CID-11 fora do escopo até a ANS exigir. |
| **Guia** | Documento TISS de cobrança/solicitação. Tipos: **Consulta**, **SP/SADT** (Serviço Profissional / Serviço Auxiliar de Diagnóstico e Terapia — exames e terapias), **Solicitação de Internação**, **Resumo de Internação**, **Honorários**. |
| **Indicação clínica** | Campo texto da guia SP/SADT/Internação com a justificativa clínica do pedido. É o que o auditor médico da operadora lê. |
| **Lote** | Conjunto de guias enviado em um XML TISS para a operadora. |
| **Glosa** | Recusa total ou parcial de pagamento pela operadora. **Administrativa** (erro de preenchimento, código, prazo, autorização ausente), **técnica** (falta de justificativa ou não conformidade clínica/DUT), **linear** (corte sem motivo específico). |
| **Recurso de glosa** | Contestação do prestador junto à operadora. |
| **Procedimento acessório** | Código cobrado junto com um principal (ex.: Doppler junto ao ultrassom). Esquecer de lançar = receita perdida, não glosa. |
| **Autorização prévia** | Liberação exigida pela operadora antes de realizar certos procedimentos. |
| **Faturista** | Secretária ou profissional de faturamento que lança procedimentos e monta as guias. Na prática é quem opera o faturamento, não o médico. |
| **PEP** | Prontuário Eletrônico do Paciente. |
| **Evolução clínica** | Registro textual do atendimento no PEP (anamnese, exame físico, hipótese, conduta). Formato **SOAP** = Subjetivo, Objetivo, Avaliação, Plano. |
| **Integrador** | Sistema cliente da API (na F1, o PEP/faturamento do Thiago Polydoro). |
| **Clínica (tenant)** | Prestador atendido pelo integrador; unidade de feature flag e de métricas. |
| **Jev** | Modelo "System One" da TypeSafe AI: recebe estado + perguntas tipadas (`noul`, `choice`, `score`) e devolve probabilidades calibradas. Não gera texto livre. |
| **CMO** | Dr. João Pedro, sócio médico; responsável por validar critérios clínicos e golden sets. |
