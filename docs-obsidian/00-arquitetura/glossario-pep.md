---
projeto: PEP
tipo: arquitetura
status: ativo
tags:
  - tissia
  - pep
  - arquitetura
  - glossario
aliases:
  - glossario
origem: tissia/docs/glossario.md
---

# Glossário do PEP Tissia

Complementa o [[glossario-tissia-ia|glossário da Spike]], que define ANS, operadora, prestador, TISS, TUSS, Rol, DUT, CID-10, guia, indicação clínica, lote, glosa, recurso de glosa, procedimento acessório, autorização prévia, faturista, PEP, evolução clínica, integrador e Jev. Aqui ficam só os termos do PEP ou o sentido que eles têm no PEP.

## Organização e acesso

| Termo | Significado |
| :--- | :--- |
| **Tissia** | Nome do produto (antes "MedScribe", renomeado em 09/11/2026). No texto corrido, "a Tissia"; para distinguir do motor de IA ou de outros PEPs, "PEP Tissia". Ver [[marca]] |
| **Tissia IA** | A IA anti-glosa e a voz ambiental do produto, entregues pela Spike (repositório `tissia-ai`). É o nome que o usuário vê; "Spike" é o codinome técnico |
| **Tenant** | Cliente contratante do PEP (grupo, empresa ou médico autônomo). Unidade de isolamento de dados (RLS) e de chaves de criptografia. |
| **Clínica** | Estabelecimento dentro de um tenant. Um tenant pode ter várias. |
| **Unidade** | Local de atendimento de uma clínica, com CNES e identificação de prestador próprios **[validar]**. Define agenda, contratos com operadoras e numeração de lotes. É o `clinica_ref` enviado à Spike. |
| **Profissional** | Médico ou outro profissional de saúde com conselho (CRM, CREFITO…) que atende e assina. |
| **Usuário** | Login no PEP. Um profissional é um usuário; recepção e faturista também. |
| **Perfil** | Conjunto de permissões (médico, faturista, recepção, gestor, admin) aplicado a um usuário em uma ou mais unidades. |
| **Backoffice** | Console da equipe interna (`apps/backoffice`), fora do PEP das clínicas. Nunca exibe dado clínico ([[PRD-003-backoffice\|PRD-003]]). |
| **Operador interno** | Pessoa da equipe com papel interno (suporte, financeiro, eng. de IA, curadoria, direção médica, segurança) no realm da equipe. Não é usuário do PEP. |
| **Sessão de ambiente** | Sessão do backoffice aberta para Homologação ou Produção; em Produção exige motivo e ticket e dura 60 min. |
| **Acesso de suporte** | Vínculo temporário (≤ 4 h) de um operador interno a um tenant, pedido pelo backoffice e aprovado pelo administrador da clínica no PEP; sem acesso a prontuário ([[ADR-PEP-0022-backoffice-sem-dado-clinico\|ADR-PEP-0022]]). |
| **Tabela de plataforma** | Tabela que o papel `app_plataforma` pode ler entre tenants (`tenants`, `clinicas`, `unidades`, `convites`, `acessos_suporte`, `uso_diario`…). Nunca clínica. |

## Paciente e convênio

| Termo | Significado |
| :--- | :--- |
| **Beneficiário** | Vínculo paciente × operadora × plano × carteirinha × validade. Um paciente pode ter vários. |
| **Carteirinha** | Número de identificação do beneficiário na operadora. Dado pessoal: nunca vai à Spike. |
| **Elegibilidade** | Confirmação de que o beneficiário pode ser atendido naquele dia e procedimento (plano ativo, sem carência). Verificada por webservice da operadora ou manualmente. |
| **Autorização prévia (senha)** | Liberação da operadora antes de certos procedimentos. Gera um número de senha com validade, registrado na guia. A regra de quando exigir vem do contrato. |

## Clínico

| Termo | Significado |
| :--- | :--- |
| **Atendimento** | Encontro paciente × profissional numa data, originado na agenda. Agrupa evolução, documentos e guias. Fechá-lo dispara `AtendimentoFechado`. |
| **Evolução** | Registro clínico do atendimento. Cada gravação gera uma **versão** imutável. |
| **Adendo** | Acréscimo a uma evolução já fechada, com autor, data e justificativa. É a única forma de corrigir depois do fechamento. |
| **SOAP** | Formato Subjetivo, Objetivo, Avaliação, Plano. Padrão da evolução quando não há modelo da especialidade. |
| **Modelo de prontuário** | Formulário estruturado por especialidade (campos, condições de exibição, vínculos TUSS/CID, escores). Versionado e compartilhável. |
| **Documento clínico** | Receita, pedido de exame, atestado, encaminhamento, laudo ou resumo ao paciente gerado no atendimento. |
| **Autoria (humano × IA)** | Marcação, por trecho, de quem escreveu: o profissional ou uma sugestão de IA aceita. Exigida para auditoria e CFM. |

## Faturamento

| Termo | Significado |
| :--- | :--- |
| **Guia rascunho** | Guia criada automaticamente a partir do atendimento, ainda não conferida. |
| **Revisão da guia** | Cada versão salva da guia. Todas compartilham o mesmo `guia_ref` na Spike. |
| **Lote** | Conjunto de guias de uma operadora e unidade num XML TISS. |
| **Protocolo** | Número que a operadora devolve ao receber o lote. |
| **Demonstrativo** | Retorno da operadora com o resultado de cada guia (pago, glosado, motivo). Chega em XML. |
| **Desfecho** | Resultado final de uma guia: `pago`, `glosado_parcial` ou `glosado_total`, com código da Tabela 38 **[validar]**. Enviado à Spike para aprendizado. |
| **Recurso** | Contestação de glosa enviada à operadora com argumento e evidências, dentro do prazo do contrato. |
| **Repasse** | Valor devido ao profissional, calculado sobre o que a clínica efetivamente recebeu (pós-glosa). |
| **Receita esquecida** | Atendimento realizado na agenda sem guia emitida. Dinheiro que nunca foi cobrado. |

## Spike

| Termo | Significado |
| :--- | :--- |
| **Spike** | Motor de IA anti-glosa (repo `tissia-ai`, API `Tissia IA v1`). Sistema separado, vendável à parte. Não gera XML nem fala com operadoras. |
| **SpikeAdapter** | Módulo `tissia-ai` do backend do PEP. Único ponto de comunicação com a Spike: pseudonimiza, monta DTOs estritos, aplica timeout e persiste a análise. |
| **Análise** | Resultado de `POST /v1/guias/analisar` (`analise_id`, risco, alertas, DUT, justificativa, sugestões, `versao_bases`). Persistida no PEP ligada à revisão da guia. |
| **Modo silencioso** | O PEP analisa e guarda, mas não exibe. Mede a linha de base de glosa. |
| **Modo assistido** | O PEP exibe a análise e coleta feedback. |
| **Análise parcial** | Exibição quando a Spike responde `degradado: true` ou não responde em 6 s: regras TISS/ANS ativas, IA clínica indisponível. O fluxo nunca é bloqueado. |
| **`paciente.ref`** | Identificador opaco do paciente enviado à Spike: HMAC-SHA256 do id do paciente com sal por clínica. O vínculo fica só no PEP, criptografado. |

## Conformidade e técnica

| Termo | Significado |
| :--- | :--- |
| **ICP-Brasil** | Infraestrutura de Chaves Públicas Brasileira. Certificado digital com validade jurídica para assinar documentos clínicos. |
| **PSC** | Prestador de Serviço de Confiança: guarda o certificado na nuvem e assina remotamente **[validar]** nomenclatura ITI vigente. |
| **PAdES** | Padrão de assinatura digital embutida em PDF. |
| **RIPD** | Relatório de Impacto à Proteção de Dados Pessoais (LGPD). Obrigatório antes de tratar dado real de saúde. |
| **NGS1 / NGS2** | Níveis de Garantia de Segurança da certificação SBIS/CFM de S-RES **[validar]**. NGS2 permite dispensar o papel. |
| **Outbox** | Tabela onde o evento de domínio é gravado na mesma transação do dado; um processo publica na fila depois. Evita evento perdido. |
| **RLS** | Row-Level Security do PostgreSQL. Filtra linhas por tenant no próprio banco. |
