---
projeto: ecossistema
tipo: arquitetura
status: ativo
tags:
  - tissia
  - ecossistema
  - arquitetura
  - lgpd
  - cfm
  - icp-brasil
  - ans
  - tiss
aliases:
  - conformidade
origem: tissia/docs/conformidade.md
---

# Conformidade: LGPD, CFM, ICP-Brasil, TISS/ANS, SBIS e segurança

Checklists por norma, com a fase em que cada item precisa estar pronto e a evidência que comprova. Toda referência normativa está marcada **[validar]** até a revisão jurídica e regulatória. Nada aqui substitui parecer jurídico.

## 1. LGPD (Lei nº 13.709/2018) **[validar]**

| Item | Fase | Evidência |
| :--- | :--- | :--- |
| Dado de saúde tratado como sensível (art. 5º, II; art. 11) | P0 | Classificação de dados no [[modelo-de-dados-pep\|modelo de dados]]; campos sensíveis cifrados |
| Base legal: tutela da saúde (art. 11, II, "f") para o atendimento; obrigação legal/regulatória para guarda do prontuário e faturamento; consentimento só onde a base legal não cobre (ex.: escuta ambiental, WhatsApp) | P1 | Registro de bases legais por finalidade (ROPA) |
| Papéis: clínica = controladora; PEP Tissia = operador; Spike = suboperadora (quando contratada pelo PEP) ou operadora (quando contratada direto por outro PEP); Google Cloud e demais = suboperadores **[validar com jurídico; mesma pergunta Q6 da Spike]** | P1 (antes de dado real) | Contrato com cada clínica, DPA PEP ↔ Spike, lista pública de suboperadores |
| RIPD (Relatório de Impacto) do PEP, incluindo a integração com a Spike e a voz ambiental | P1 (antes de dado real); revisar na P2 (voz) | Documento assinado pelo encarregado |
| Encarregado (DPO) nomeado e canal publicado | P1 | Página de privacidade |
| Minimização para a Spike: só `paciente.ref`, sexo e idade; evolução sem identificadores diretos | P1 | Teste de propriedade do SpikeAdapter ([[test-plan-pep\|test plan]]); DTOs com `extra="forbid"` espelhado |
| Pseudonimização: `paciente.ref` = HMAC com sal por clínica; vínculo só no PEP, cifrado | P1 | [[ADR-PEP-0005-spike-adapter\|ADR-PEP-0005]]; teste que o mesmo paciente tem refs diferentes em clínicas diferentes |
| Direitos do titular (acesso, correção, portabilidade, informação sobre compartilhamento) | P2 | Fluxo em Configurações › LGPD (tela 25); exportação do prontuário em formato aberto. Correção de prontuário só por adendo (o prontuário não pode ser apagado dentro do prazo de guarda) |
| Consentimentos registrados com versão do termo, data e canal (escuta ambiental, WhatsApp, pesquisa) | P1 (WhatsApp); P2 (voz) | Tabela `Consentimento`; selo "Consentimento LGPD ativo" (tela 04) |
| Trilha de acesso ("quem viu") a cada prontuário aberto | P1 | Log de acesso imutável; consulta na tela 25 |
| Dados sensíveis mascarados na UI; revelar exige justificativa auditada | P1 | Tela 04 (CPF/CNS mascarados) |
| Incidentes: plano de resposta, comunicação à ANPD e titulares no prazo regulamentar **[validar prazo]** | P1 | Runbook de incidente; simulado antes do piloto |
| Retenção: prontuário pelo prazo legal **[validar: 20 anos, Lei nº 13.787/2018]**; logs de acesso; anexos; dados da Spike conforme Spike ADR-0007 | P1 | Política de retenção por entidade |
| Transferência internacional: nenhuma no PEP (GCP São Paulo, [[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]]); provedores fora do Brasil só com cláusulas-padrão ANPD | P0 | Inventário de provedores com região |
| Opt-in `guardar_entrada_para_avaliacao` da Spike desligado até termo assinado pela clínica | P1 | Flag auditada ([[SPK-01-api-de-clinicas-e-flags\|SPK-01]]) |

## 2. CFM

| Item | Fase | Evidência |
| :--- | :--- | :--- |
| IA como apoio à decisão e à documentação, com responsabilidade do médico. O design cita a Resolução CFM nº 2.454/2026 **[validar número, vigência e requisitos]** | P1 | Aviso legal fixo no painel de revisão; nada da IA entra no prontuário ou na guia sem aceite |
| Autoria registrada por trecho: humano, IA aceita, IA editada | P1 | `DecisaoUsuario` + marcação visual dos trechos de IA (telas 05, 07) |
| Prontuário: conteúdo, guarda e sigilo (Res. CFM nº 1.638/2002) **[validar]** | P1 | Prontuário imutável com versões e adendos ([[ADR-PEP-0007-prontuario-imutavel\|ADR-PEP-0007]]) |
| Prontuário eletrônico e eliminação do papel com certificação (Res. CFM nº 1.821/2007) **[validar]** | P2 (assinatura) / P3 (SBIS) | Assinatura ICP-Brasil; certificação SBIS NGS2 |
| Telemedicina (Res. CFM nº 2.314/2022) **[validar]** | P3 | Consentimento, registro e guarda da teleconsulta |
| Sigilo: acesso por perfil e necessidade (faturista sem leitura clínica completa, tela 01b "somente leitura") | P0 | Matriz de permissões (RFC-PEP-001) |

## 3. ICP-Brasil

| Item | Fase | Evidência |
| :--- | :--- | :--- |
| Validade jurídica da assinatura digital (MP nº 2.200-2/2001) **[validar]** | P2 | Assinatura qualificada via PSC credenciado ([[ADR-PEP-0011-assinatura-icp-brasil\|ADR-PEP-0011]]) |
| Padrão de assinatura de documentos (PAdES para PDF) com carimbo do tempo **[validar requisito de carimbo]** | P2 | Validação no verificador oficial do ITI **[validar ferramenta]** |
| Receita digital e atestado com assinatura qualificada; receitas de controle especial com requisitos próprios **[validar]** | P2 | Fluxo de prescrição (RFC-PEP-004/011) |
| Login com certificado ICP-Brasil (tela 01a) | P0/P1 | Keycloak com autenticação X.509 ([[ADR-PEP-0010-identidade-keycloak\|ADR-PEP-0010]]) |

## 4. TISS/ANS

| Item | Fase | Evidência |
| :--- | :--- | :--- |
| Versão do padrão TISS como configuração por operadora; geração de XML 4.03.00 no MVP (vigente, competência set/2026); leitura de 4.03.00 e de versões anteriores só se o piloto exigir **[validar]**; CID fora do XML de SP/SADT e consulta | P1 (geração) / P2 (leitura) | Validação contra os XSD oficiais na CI |
| Tabela TUSS e CID-10 vigentes; versão das bases registrada em cada análise (`versao_bases` da Spike) | P1 | `AnaliseTissiaAi.versao_bases` |
| Campo 23 (indicação clínica) ≤ 500 caracteres **[validar]** | P1 | Editor com contador e validação no backend |
| Prazos de envio de lote e de recurso de glosa por contrato/operadora **[validar referências normativas]** | P2 | Prazos cadastrados em `Contrato`; alertas na tela 20 |
| Motivos de glosa pela Tabela 38 TUSS **[validar numeração]** | P2 | `Desfecho.codigo_glosa` |
| Rol de procedimentos e DUT (RN ANS nº 465/2021 e atualizações) **[validar]** | P1 (via Spike) | Spike RFC-001/004 |

## 5. SBIS/CFM — certificação NGS1/NGS2 **[validar requisitos no manual vigente]**

Meta da P3, mas estes requisitos custam caro se adicionados depois. Entram desde a P0/P1:

| Requisito | Fase | Como |
| :--- | :--- | :--- |
| Trilha de auditoria completa e inviolável (criação, leitura, alteração, impressão, exportação) | P0 | Tabela append-only, sem `UPDATE`/`DELETE` por permissão de banco; hash encadeado **[validar se exigido]** |
| Controle de acesso por perfil, sessão com expiração, bloqueio por tentativas | P0 | Keycloak + RBAC no backend (tela 01a: bloqueio após tentativas) |
| Autenticação forte para profissionais (MFA) | P0 | TOTP/FIDO2 (tela 01b) |
| Assinatura digital ICP-Brasil para eliminar papel (NGS2) | P2 | ADR-PEP-0011 |
| Sincronização de tempo confiável (timestamps de auditoria e assinatura) | P0 | NTP do provedor; carimbo do tempo na assinatura |
| Backup, restauração testada e continuidade | P0 | PITR do Cloud SQL; teste de restore trimestral |
| Integridade e imutabilidade do registro clínico | P1 | ADR-PEP-0007 |
| Privacidade: mascaramento, sigilo por perfil | P1 | Tela 04, matriz de permissões |

## 6. Segurança da aplicação

| Item | Fase | Evidência |
| :--- | :--- | :--- |
| OWASP ASVS nível 2 como referência de requisitos | P0 | Checklist ASVS por release no [[test-plan-pep\|test plan]] |
| Criptografia em trânsito (TLS 1.2+) e em repouso; chaves por tenant para campos sensíveis | P0 | Cloud KMS; cifra de campo (CPF, CNS, telefone, sal do VinculoSpike) |
| Isolamento entre tenants por RLS ([[ADR-PEP-0003-postgres-rls-multitenant\|ADR-PEP-0003]]) | P0 | Teste automatizado: usuário do tenant A nunca lê linha do tenant B, em toda tabela |
| Dependências e imagens escaneadas | P0 | `npm audit`/Dependabot e scan de imagem na CI |
| Pen test externo | Antes do piloto (P1) e anualmente | Relatório e correção dos achados altos |
| Segredos só no Secret Manager | P0 | Revisão de CI e repositórios |

## 7. Pré-requisitos para dado real

Alinhados ao [[release-plan-pep|release plan]]. Nenhuma clínica-piloto entra com dado real antes de:

- [ ] Contrato com a clínica (controladora) e DPA PEP ↔ Spike assinados; papéis LGPD confirmados pelo jurídico.
- [ ] RIPD concluído e assinado pelo encarregado.
- [ ] Bases legais e termos de consentimento (WhatsApp; escuta ambiental quando houver) revisados.
- [ ] RLS e teste de isolamento entre tenants passando em `staging`.
- [ ] Trilha de acesso e log de auditoria ativos e consultáveis.
- [ ] MFA obrigatório para todos os profissionais.
- [ ] Pen test sem achado alto aberto.
- [ ] Runbook de incidente e simulado feitos.
- [ ] Backup e restauração testados.
- [ ] Clínicas configuradas na Spike com `guardar_entrada_para_avaliacao=false` (salvo opt-in assinado) e em modo silencioso no PEP.
- [ ] Revisão regulatória dos itens **[validar]** desta página que bloqueiam a P1.
