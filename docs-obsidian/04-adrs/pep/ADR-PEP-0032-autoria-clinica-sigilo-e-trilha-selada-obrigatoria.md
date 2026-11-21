---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - autoria-clinica
  - sigilo
  - auditoria
  - lgpd
aliases:
  - ADR-PEP-0032
  - Autoria clínica sigilo e trilha selada obrigatória
---

# ADR-PEP-0032 — Autoria clínica, sigilo e trilha selada obrigatória

**Status:** Proposta — 21/11/2026  
**Relacionadas:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0008-eventos-outbox-bullmq|ADR-PEP-0008]], [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0031-separacao-papeis-checkout-recepcao-e-triagem|ADR-PEP-0031]], [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]], [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia|RFC-PEP-026]]

## Contexto

A auditoria de fluxos clínicos, documentais, de triagem, checkout e faturamento identificou fronteiras que precisam ser explícitas antes de ampliar o uso produtivo:

1. Autoria e titularidade podem divergir se a API confiar em identificadores enviados pelo cliente. A impressão também não pode inferir o signatário a partir do profissional titular do atendimento.
2. Fechamento precisa encerrar a edição do conteúdo documental ligado ao atendimento. Resumo liberado e dados de triagem requerem histórico versionado para que correções não substituam autoria ou sinais vitais anteriores.
3. Papéis de recepção, gestão, suporte e faturamento precisam receber somente o mínimo necessário. Permissão ampla de agenda ou leitura administrativa não deve revelar documento clínico, alergias em listagens, SOAP ou XML TISS.
4. A trilha já possui gravação transacional em pontos do sistema e selagem por hash assíncrona, mas a cobertura de leituras clínicas e de eventos de identidade, consentimento e IA ainda precisa ser universalizada. A exportação do DPO deve levar IP e agente do usuário quando coletados no contexto autenticado.

O artigo 11 da LGPD trata dados de saúde como dados pessoais sensíveis; o dever de sigilo profissional inclui a vedação de revelar informação obtida no exercício profissional, ressalvadas hipóteses legais e éticas aplicáveis. A implementação e as bases legais específicas devem ser validadas com jurídico e governança clínica.

## Decisão

1. **Autoria clínica estrita.** Toda mutação de evolução, pedido, escuta, fechamento e assinatura deriva do usuário autenticado e da relação autorizada com o atendimento. `profissionalId` do corpo não é fonte de autoridade. O profissional titular (`atendimentos.profissional_id`) pode atuar; delegação de plantão ou coassinatura exige concessão explícita, escopo, motivo, período e evento de auditoria. O fechamento registra `fechado_por`. A impressão identifica exclusivamente `documentos.assinado_por` e o CRM/UF correspondente ao signatário validado, nunca o titular por inferência.
2. **Imutabilidade e versões.** Com `atendimentos.status = 'fechado'`, `documentos` existentes não admitem alteração ou exclusão pelo papel da aplicação. Novo conteúdo clínico exige novo documento ligado a adendo/complementação imutável. `resumo_paciente` liberado é imutável; correção gera nova versão e evento. `triagens` passa a ser append-only: cada versão conserva autor, horário e sinais vitais; `autor_id` e sinais vitais de uma versão não podem ser sobrescritos.
3. **Sigilo por finalidade e escopo.**
   - Impressão do checkout exige `documentos:imprimir_checkout`, com resposta limitada a PDF/payload sanitizado e aos documentos assinados do atendimento do dia. Essa permissão não concede `agenda:ler` genérico nem leitura clínica ampla.
   - Pacote de recurso de glosa exige `prontuario:ler_trecho_guia`; contém apenas o trecho clínico aprovado e necessário, nunca SOAP integral.
   - `guias:ler_metadados` não concede baixar XML. Download exige `lotes:baixar_xml`; os perfis suporte de faturamento por metadados, admin, recepção e gestor não recebem essa permissão por padrão.
   - `alergias` não aparece em listagens não assistenciais de pacientes nem em `visoes/pacientes`; acesso assistencial individual permanece sujeito a escopo e auditoria.
   - Admin não pode autoatribuir perfil assistencial nem elevar o próprio vínculo. Credenciais CRM/COREN/CBO não são autogeridas sem curadoria, validação e trilha de aprovação.
4. **Trilha síncrona e universal.** Toda mutação clínica, documental, de IA, identidade ou consentimento, e as leituras clínicas de linha do tempo, triagem e `guias/:id`, grava `auditoria_eventos` dentro da mesma transação de negócio. Falha ao gravar aborta a operação. O outbox pode distribuir eventos posteriormente, mas não substitui a trilha transacional. A exportação do DPO inclui `ip` e `user_agent` disponíveis, com controle de acesso e minimização.
5. **Banco e autorização.** Novas tabelas são aditivas, incluem `tenant_id uuid NOT NULL`, contexto por `Db.comTenant` e política RLS. Restrições e triggers impedem mutações proibidas; a autorização do backend continua obrigatória mesmo com RLS.

## Alternativas consideradas

| Alternativa | Motivo para rejeição |
| :--- | :--- |
| Confiar em `profissionalId` enviado pela interface | Permite falsificar autoria e não prova vínculo com o atendimento. |
| Imprimir o titular do atendimento como signatário presumido | Pode atribuir assinatura a profissional diferente do autor real. |
| Usar uma permissão genérica de agenda/documentos no checkout | Amplia acesso a pacientes e conteúdo clínico além da finalidade de impressão. |
| Colocar auditoria apenas no outbox | Falha ou atraso do consumidor deixaria operações concluídas sem trilha síncrona garantida. |
| Atualizar triagem e resumo no mesmo registro | Apaga a sequência histórica, autoria e valores que fundamentaram decisões. |

## Consequências

- Backend, contratos Zod `.strict()`, políticas RBAC, consultas e telas terão de expressar finalidade e escopo; esconder controles na UI não é controle de acesso.
- Rotas que leem conteúdo clínico falham fechadas se não conseguirem registrar o evento transacional. Isso pode elevar latência e indisponibilidade, que devem ser medidas sem enfraquecer a garantia.
- Coassinatura e delegação exigem fluxo de concessão e validação profissional; dados inválidos bloqueiam assinatura.
- A exportação de auditoria contém metadados pessoais de acesso e exige autorização, retenção e proteção adequadas.
- Mudanças estruturais serão implementadas por migrações aditivas sob [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]] e [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]. A interpretação regulatória específica e o prazo de retenção permanecem sujeitos a validação jurídica.
