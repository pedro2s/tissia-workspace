---
projeto: PEP
tipo: prd
status: em-refinamento
tags:
  - tissia
  - pep
  - prd
  - produto
origem: tissia/docs/prd/PRD-001-pep-tissia.md
---

# PRD-001 — PEP Tissia

**Versão:** 1.0
**Data:** 07/11/2026
**Autor:** Pedro Santana
**Status:** Aprovado para P0/P1; P2–P3 em refinamento

Fontes: documento "PEP Tissia — Design de Produto e Arquitetura" (06/11/2026), [canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d) (telas 01–26) e a documentação da Spike ([[PRD-IA-001-tissia-ia|Spike PRD-001]], [[manual-integracao-tissia-ia-v1|manual de integração]]).

## 1. Tese

Os PEPs atuais (iClinic, Feegow, Amplimed, HiDoctor, MV, Tasy) tratam o faturamento como etapa posterior: o erro nasce no prontuário e só aparece na glosa, 30 a 90 dias depois. O PEP Tissia **fatura a partir da evolução clínica**: cada ato médico já nasce com TUSS, CID, DUT e justificativa validados pela Spike antes de virar guia.

**Promessa:** o médico documenta uma vez e a guia sai correta, sem retrabalho do faturista.

## 2. Princípios

| # | Princípio | Consequência no produto |
| :--- | :--- | :--- |
| 1 | Clínico primeiro, faturamento invisível | O médico nunca preenche campo de faturamento; revisa sugestões |
| 2 | Prevenção, não correção | Alertas aparecem ao fechar o atendimento, não no fim do mês |
| 3 | Human-in-the-loop em tudo que a IA toca | Nada vai à operadora sem aceite explícito (Res. CFM nº 2.454/2026 **[validar]**; [[ADR-IA-0008-human-in-the-loop\|Spike ADR-0008]]) |
| 4 | PII fica no PEP | A Spike recebe só identificadores opacos, sexo e idade ([[ADR-PEP-0005-spike-adapter\|ADR-PEP-0005]]) |
| 5 | Nunca bloquear o atendimento | Spike lenta ou fora → "análise parcial" e fluxo segue |
| 6 | Tudo auditável | Quem viu, aceitou, editou ou rejeitou cada sugestão fica registrado |

## 3. Personas

| Persona | Fase | Telas | O que ganha |
| :--- | :--- | :--- | :--- |
| **Médico(a)** | P1+ | 05, 06, 07, 08, 11, 12, 13 | Documenta uma vez (texto, comandos `/`, voz na P2); revisa a guia em ≤ 15 s |
| **Faturista** | P1+ | 13, 16, 17, 18, 19, 20, 22 | Fila ordenada por risco, revalidação, lote XML sem retrabalho; glosas e recursos na P2 |
| **Recepção** | P1+ | 03, 04 | Elegibilidade e autorização prévia visíveis no agendamento; lembretes por WhatsApp |
| **Gestor(a) da clínica** | P2+ | 15, 21, 23, 24 | Glosa evitada (silencioso × assistido), receita esquecida, repasse sobre valor recebido |
| **Administrador(a) / DPO** | P0+ | 01, 25 | Perfis, MFA, trilha de auditoria, consentimentos, integrações |
| **Paciente (indireto)** | P1+ | 08 (resumo), 04 (consentimento) | Lembretes, resumo em linguagem simples, direitos de titular (LGPD) |

## 4. Objetivos e métricas

| Objetivo | Métrica | Meta | Medido em |
| :--- | :--- | :--- | :--- |
| Guia certa na origem | % de guias SP/SADT sem alerta `bloqueante` aberto ao entrar na fila do faturista | ≥ 90% | P1 piloto |
| Revisão rápida | Tempo da abertura do painel de revisão até a decisão (tela 13) | p50 ≤ 15 s | P1 piloto |
| Sugestões úteis | Taxa de aceite (aceito + editado) / exibidas | ≥ 70% (mesma meta da Spike) | P1 piloto (modo assistido) |
| Reduzir glosa | Taxa de glosa (R$ glosado / R$ faturado) vs. baseline do modo silencioso | −30% no piloto **[validar]** com baseline real | P2 |
| Receita esquecida | Atendimentos sem guia detectados e recuperados / mês | Registrar; meta após baseline | P2 |
| Não atrapalhar | p95 de carregamento de tela (agenda, prontuário, mesa) | ≤ 1,5 s | P1 |
| Não atrapalhar | Atendimentos fechados mesmo com Spike degradada | 100% | P1 |
| Adoção | Médicos ativos semanais / médicos com licença | ≥ 80% no piloto | P1–P2 |

## 5. Escopo por fase e requisitos

Telas citadas entre parênteses. RFC responsável no [[hub-pep-tissia#telas-do-design--rfc|índice]]. Aqui ficam os requisitos-mestre de produto; cada RFC pode desdobrar sub-requisitos técnicos mantendo a mesma família (e a [[RFC-PEP-000-fundacoes|RFC-PEP-000]] usa a família técnica `RF-FND-01..08` para padrões de infraestrutura, monorepo e API, evitando colisão com `RF-PLT-*`).

### P0 — Fundações

**Identidade e acesso (ACE)** — tela 01a/01b, [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]]

- **RF-ACE-01** Login por e-mail e senha com mensagens de erro, Caps Lock, bloqueio após N tentativas e sessão expirada (01a).
- **RF-ACE-02** MFA obrigatório para profissionais: TOTP, chave FIDO2/WebAuthn e códigos de recuperação; "confiar no dispositivo" por 30 dias (01b).
- **RF-ACE-03** Login alternativo por certificado ICP-Brasil e SSO da organização (01a). SSO e certificado podem sair na P1 se o provedor exigir.
- **RF-ACE-04** Seleção de clínica e unidade após o login; a unidade define agenda, contratos e numeração de lotes (01b).
- **RF-ACE-05** Perfis e permissões por papel (médico, faturista, recepção, gestor, admin) com escopo por unidade; faturista lê o prontuário só no que a guia precisa.

**Plataforma (PLT, núcleo)** — [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]]

- **RF-PLT-01** Multi-tenant (tenant → clínica → unidade) com isolamento por RLS ([[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]]).
- **RF-PLT-02** Trilha de auditoria imutável: login, acesso a prontuário ("quem viu"), revelação de dado mascarado (com justificativa), decisões sobre sugestões de IA.

**Interface (UI)** — [[RFC-PEP-015-frontend|RFC-PEP-015]]

- **RF-UI-01** Design system com os tokens do design (Inter, teal `#0B6E78`, navy `#0F1B2D`), componentes acessíveis e estados vazio/carregando/erro/sucesso.

### P1 — MVP (90 dias)

**Interface (UI)**

- **RF-UI-02** Shell com barra lateral, barra superior (trilha, busca global, unidade, status da IA, notificações) e barra de comandos `Ctrl/Cmd+K` (02).
- **RF-UI-03** Central de notificações por categoria (clínico, faturamento, sistema) (02).
- **RF-UI-04** Atalhos de teclado documentados (`?`) (02).
- **RF-UI-05** Telas 05, 06, 13 e 16 utilizáveis em tablet e celular com alvos de toque ≥ 44 px (26a–d).

**Paciente e beneficiário (PAC)** — tela 04, [[RFC-PEP-002-paciente-e-beneficiario|RFC-PEP-002]]

- **RF-PAC-01** Cadastro de paciente com nome social, sexo TISS, CPF, CNS, contato; dados sensíveis mascarados por padrão.
- **RF-PAC-02** Revelar dado mascarado exige justificativa e gera registro de auditoria.
- **RF-PAC-03** Várias carteirinhas por paciente (operadora, plano, acomodação, titularidade, validade), com uma principal.
- **RF-PAC-04** Verificação de elegibilidade por carteirinha (manual na P1; webservice da operadora quando disponível, ver [[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]]), com data/hora e origem da verificação.
- **RF-PAC-05** Histórico de guias do paciente com status e valor.
- **RF-PAC-06** Registro de consentimentos (LGPD, escuta ambiental, canal preferido).

**Agenda e recepção (AGE)** — tela 03, [[RFC-PEP-003-agenda|RFC-PEP-003]]

- **RF-AGE-01** Agenda multiprofissional e por sala, visões Dia, Semana e Salas, filtros por profissional e convênio.
- **RF-AGE-02** Agendar, encaixar, bloquear horário, remarcar e cancelar, com conflitos de sala/profissional impedidos.
- **RF-AGE-03** Elegibilidade e autorização prévia exibidas em cada horário; regra de autorização vem do contrato (P2) ou de cadastro manual (P1).
- **RF-AGE-04** Check-in, status do atendimento (aguardando, em atendimento, atendido) e fila.
- **RF-AGE-05** Lembrete e confirmação por WhatsApp, com contadores (enviados, confirmados, sem resposta, remarcação).

**Prontuário (PRO)** — tela 05, [[RFC-PEP-004-prontuario|RFC-PEP-004]]

- **RF-PRO-01** Linha do tempo do paciente (consultas, exames, guias, anexos), problemas ativos e medicamentos em uso.
- **RF-PRO-02** Evolução SOAP ou por modelo da especialidade, com salvamento automático.
- **RF-PRO-03** Cada gravação gera versão imutável; após fechar o atendimento, só adendo com autor, data e justificativa ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]).
- **RF-PRO-04** Menu de comandos `/receita`, `/exame`, `/atestado`, `/encaminhamento` dentro da evolução.
- **RF-PRO-05** Anexos (PDF, imagem) em armazenamento de objetos, com antivírus e limite de tamanho.
- **RF-PRO-06** Fechar atendimento dispara `AtendimentoFechado`; se houver pedido de exame/procedimento, cria guia rascunho e chama a Spike.
- **RF-PRO-07** Marcação de autoria por trecho (humano × IA) visível na evolução.
- **RF-PRO-08** Rascunho offline no tablet com sincronização posterior ([[ADR-PEP-0015-offline-prontuario|ADR-PEP-0015]]).

**Documentos clínicos (DOC)** — tela 08, RFC-PEP-004

- **RF-DOC-01** Receita (simples na P1; controle especial depende de assinatura ICP, P2).
- **RF-DOC-02** Pedido de exame com indicação clínica, já ligado ao TUSS e à guia rascunho.
- **RF-DOC-03** Atestado de comparecimento e de afastamento.
- **RF-DOC-04** Impressão/PDF de todos os documentos com identificação do profissional.

**Modelos de prontuário (MOD)** — telas 09, 10, 11, RFC-PEP-004

- **RF-MOD-01** Construtor de modelos: arrastar e soltar campos, condição de exibição, vínculo com TUSS/CID, pré-visualização ao vivo (09).
- **RF-MOD-02** Biblioteca por especialidade: duplicar, versionar, compartilhar entre profissionais (10).
- **RF-MOD-03** Modelos iniciais para 2 especialidades (Ortopedia e Cardiologia **[validar]** com o piloto), com exame físico estruturado e escores calculados (11).

**Integração Spike (SPK)** — telas 12–15, [[RFC-PEP-005-spike-adapter|RFC-PEP-005]]

- **RF-SPK-01** `POST /v1/guias/analisar` ao fechar o atendimento, ao salvar a guia e na conferência do faturista, sempre com o mesmo `guia_ref`.
- **RF-SPK-02** Pseudonimização: só `paciente.ref`, sexo e idade saem do PEP; nenhum campo extra (contrato `extra="forbid"`).
- **RF-SPK-03** Timeout de 6 s; sem resposta ou `degradado: true` → aviso "análise parcial (regras TISS/ANS ativas, IA clínica indisponível)" e fluxo não bloqueado (14).
- **RF-SPK-04** Painel anti-glosa: selo de risco com `score_risco`, alertas por severidade com fonte, checklist DUT com evidência literal, Campo 23 editável com contador de 500 caracteres, sugestões TUSS/CID com 1 clique, aviso legal fixo (12, 13).
- **RF-SPK-05** Aceitar/editar/rejeitar cada item envia `POST /v1/analises/{id}/feedback` com o papel de quem decidiu.
- **RF-SPK-06** Persistir `analise_id`, `versao_bases`, `score_risco`, `degradado` e decisões, ligados à revisão da guia.
- **RF-SPK-07** Modo por clínica: desligado, silencioso (analisa e guarda, não exibe) e assistido (exibe e coleta feedback) (15; [[ADR-PEP-0006-modos-silencioso-assistido|ADR-PEP-0006]]).
- **RF-SPK-08** Truncamento com critério antes do limite de 64 KB do corpo e 20.000 caracteres da evolução (prioriza o atendimento atual).

**Guias e lotes TISS (FAT)** — telas 13, 16, 17, 18, [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]

- **RF-FAT-01** Guia de consulta e SP/SADT criadas a partir do atendimento, com revisões numeradas.
- **RF-FAT-02** Mesa do faturista: fila ordenada por risco e prazo, filtros por operadora/status/médico, ação em lote e revalidação (16).
- **RF-FAT-03** Editor completo de guia com validações TISS locais (obrigatórios, sexo/idade, acessórios/excludentes, quantidades) que funcionam sem a Spike (17).
- **RF-FAT-04** Bloqueio do Campo 23 acima de 500 caracteres no editor.
- **RF-FAT-05** Geração de lote XML TISS 4.03.00 por operadora e unidade, validado contra o XSD oficial antes do download/envio (18).
- **RF-FAT-06** Gestão de lotes: criação, envio (manual na P1: download do XML), protocolo, status e histórico (18).
- **RF-FAT-13** Guia SP/SADT impressa fiel ao formulário oficial da ANS, para o paciente levar ao executante, sem CID estruturado ([[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]).

**Dados de referência da ANS (REF)** ([[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]], [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021]])

- **RF-REF-01** Manifesto versionado de cada base da ANS usada pelo PEP (fonte, versão/competência, SHA-256, licença) e XSD TISS 4.03.00 no repositório.
- **RF-REF-02** Carga idempotente, só da publicação oficial, de campos das guias, domínios TISS (inclui a Tabela 38) e operadoras ativas (CADOP), com histórico de versões.
- **RF-REF-03** Verificação semanal de defasagem contra a ANS, com alerta no Backoffice (dono da manutenção) e relatório informativo no release; não bloqueia o release.
- **RF-REF-04** Status das bases do PEP e da Spike visível no Início, na barra superior e no backoffice.

**Plataforma (PLT)**

- **RF-PLT-03** Registro de consentimento e base legal por paciente; exportação de dados do titular.
- **RF-PLT-04** Feature flags por clínica (modo Spike, especialidades, módulos, offline).

### P2 — Ciclo de receita

**Retorno e glosas (RET)** — telas 19, 20, [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]]

- **RF-RET-01** Importação do demonstrativo XML e conciliação guia a guia com divergências (19).
- **RF-RET-02** Envio dos lotes e retornos à Spike (`/v1/tiss/lotes/ingerir`, `/v1/desfechos`) por fila assíncrona.
- **RF-RET-03** Central de glosas por motivo (Tabela 38 **[validar]**), valor e médico, com prazos de recurso (20).
- **RF-RET-04** Recurso assistido por IA com evidências da evolução (depende de [[SPK-02-recurso-de-glosa|SPK-02]]).
- **RF-RET-05** Baixa manual de glosa envia desfecho individual.

**Contratos (CON)** — tela 21, [[RFC-PEP-008-contratos|RFC-PEP-008]]

- **RF-CON-01** Cadastro de contrato por operadora e unidade, tabela negociada e pacotes.
- **RF-CON-02** Regras de autorização prévia por TUSS, usadas pela agenda e pela guia.
- **RF-CON-03** Auditoria contrato × sistema via `/v1/contratos/auditar` no cadastro e semanalmente.

**Receita esquecida (ESQ)** — tela 22, [[RFC-PEP-009-receita-esquecida|RFC-PEP-009]]

- **RF-ESQ-01** Rotina diária agenda × guias via `/v1/guias/esquecidas`, com valor em risco.
- **RF-ESQ-02** Gerar a guia faltante a partir do atendimento com 1 clique.

**Financeiro (FIN)** — tela 23, [[RFC-PEP-010-financeiro-e-repasse|RFC-PEP-010]]

- **RF-FIN-01** Contas a receber (convênio por lote, particular por atendimento) e a pagar.
- **RF-FIN-02** Repasse médico calculado sobre o valor recebido, já descontadas as glosas.
- **RF-FIN-03** Conciliação bancária por arquivo (OFX/CNAB **[validar]**).
- **RF-FIN-04** Particular: cobrança simples e recibo na P1 ([[ADR-PEP-0014-convenio-e-particular-no-mvp|ADR-PEP-0014]]); completo na P2.

**Assinatura (ASS)** — tela 07, [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]]

- **RF-ASS-01** Assinatura ICP-Brasil de evolução, receita e documentos, com a trilha de revisão de IA anexada.
- **RF-ASS-02** Assinatura só depois de todas as sugestões de IA terem decisão (07).

**Voz ambiental (VOZ)** — telas 06, 07, [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]]

- **RF-VOZ-01** Escuta com consentimento registrado, pausa e marcação de trecho (06).
- **RF-VOZ-02** Transcrição com locutores separados e conversa casual descartada.
- **RF-VOZ-03** Modelo preenchido ao vivo com o trecho de origem em cada campo; nada salvo sem revisão (06, 07).
- **RF-VOZ-04** Resumo ao paciente em linguagem simples, enviável por WhatsApp (08).

**Gestão (BI)** — telas 15, 24, [[RFC-PEP-012-painel-do-gestor|RFC-PEP-012]]

- **RF-BI-01** Painel de glosa por operadora, TUSS e médico.
- **RF-BI-02** Glosa evitada: linha de base (silencioso) × assistido; relatório "estas glosas teriam sido evitadas" (15, 24).
- **RF-BI-03** Receita esquecida recuperada, aceite da IA, produtividade.

### P3 — Expansão ([[RFC-PEP-016-expansao|RFC-PEP-016]])

- **RF-FAT-07** Guias de internação (solicitação e resumo) e honorários.
- **RF-AGE-06** Telemedicina integrada à agenda e ao prontuário.
- **RF-DOC-05** Integração com laboratórios e imagem (pedido e resultado).
- **RF-PLT-05** API pública e webhooks para integradores; marketplace de integrações.
- **RF-PLT-06** Importadores de dados de concorrentes ([[ADR-PEP-0016-migracao-de-concorrentes|ADR-PEP-0016]]).
- **RF-PLT-07** Requisitos para certificação SBIS/CFM NGS1/NGS2 **[validar]**.
- **RF-BI-04** BI avançado (coortes, previsão de receita).

### Fora de escopo (sem fase)

App nativo (a PWA cobre tablet/celular); CID-11; prescrição de controle especial sem ICP; dispensação de farmácia; envio direto de glosa à operadora pela Spike (a Spike não fala com operadoras).

## 6. Requisitos não funcionais

| # | Requisito |
| :--- | :--- |
| RNF-01 | **LGPD Art. 11** **[validar]**: base legal de tutela da saúde, minimização, RIPD antes de dado real, direitos do titular, trilha de acesso ([[conformidade-regulatoria\|conformidade]]) |
| RNF-02 | **Residência de dados:** dado clínico processado e armazenado no Brasil ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]]) |
| RNF-03 | **Disponibilidade:** 99,5% mensal na P1; 99,9% como meta da P3. A indisponibilidade da Spike não reduz a do PEP |
| RNF-04 | **Latência:** p95 ≤ 1,5 s nas telas principais; análise Spike exibida em ≤ 6 s ou cai para "análise parcial" |
| RNF-05 | **Auditoria:** toda leitura de prontuário e toda decisão sobre IA registradas com usuário, unidade, data e IP; retenção conforme prazo do prontuário **[validar]** |
| RNF-06 | **Prontuário imutável:** versões e adendos; nunca update/delete de conteúdo clínico |
| RNF-07 | **Acessibilidade:** WCAG 2.1 AA; navegação por teclado completa; contraste ≥ 4,5:1 |
| RNF-08 | **Offline:** prontuário editável sem rede no tablet; sincronização sem perda |
| RNF-09 | **Segurança:** OWASP ASVS L2; criptografia em trânsito e em repouso; MFA para profissionais; chaves por tenant para dados sensíveis |
| RNF-10 | **Observabilidade:** latência e taxa de degradação da Spike, taxa de aceite, erros de lote, glosa por operadora; logs sem PII nem texto clínico |
| RNF-11 | **Compatibilidade TISS:** gerar 4.03.00, versão vigente da ANS (competência set/2026); ler 4.03.00 e as versões anteriores que operadoras do piloto ainda usarem **[validar]**; a Spike hoje lê até 4.01.00 ([[SPK-10-tiss-4-03\|SPK-10]]). Revisado em 13/11/2026 ([[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]]) |

## 7. Dependências externas

| Dependência | Para quê | Fase | Risco |
| :--- | :--- | :--- | :--- |
| Spike (`tissia-ai`, API `v1`) | Análise, feedback, desfechos, contratos, esquecidas | P1+ | Lacunas listadas em [[mapeamento-pep-tissia-ia\|spike/README]] |
| Operadoras (webservices TISS) | Elegibilidade, autorização, envio de lote | P2 (manual na P1) | Cada operadora tem particularidades; [[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]] |
| PSC ICP-Brasil | Assinatura e login por certificado | P2 | Custo por assinatura; [[ADR-PEP-0011-assinatura-icp-brasil\|ADR-PEP-0011]] |
| WhatsApp Business API | Lembretes, confirmação, resumo ao paciente | P1 | Aprovação de templates; custo por conversa |
| Provedor de identidade | MFA, FIDO2, SSO, certificado | P0 | [[ADR-PEP-0010-identidade-keycloak\|ADR-PEP-0010]] |

Catálogo completo em [[integracoes-externas|integracoes.md]].

## 8. Modelo de negócio

| Oferta | Para quem | Referência de preço |
| :--- | :--- | :--- |
| PEP Tissia (prontuário, agenda, TISS, financeiro) | Clínicas com convênio | Por médico/mês, a definir após o piloto |
| Spike como add-on do PEP | Clientes do PEP | +R$ 397–697/médico/mês (valor de upsell da [[PRD-IA-001-tissia-ia\|Spike PRD-001 §8]]) **[validar]** |
| Spike standalone | PEPs de terceiros, extensão Chrome | Ver [[PRD-002-spike-standalone\|PRD-002]] |

O modo silencioso é argumento de venda: o relatório "glosas que teriam sido evitadas" prova valor antes do cliente pagar o add-on.

## 9. Diferenciais

| Eixo | PEPs tradicionais | PEP Tissia |
| :--- | :--- | :--- |
| Glosa | Tratada depois, pelo faturista | Prevenida na origem, com evidência da evolução |
| Justificativa SP/SADT | Texto livre manual | Gerada, editável e rastreável |
| Aprendizado | Nenhum | Loop de desfechos por operadora e TUSS |
| Receita esquecida | Descoberta no fechamento | Rotina diária automática |
| Prova de valor | Relatório genérico | Modo silencioso: "estas glosas teriam sido evitadas" |
| Entrada de clientes | Migração pesada | Extensão Chrome da Spike como porta de entrada |

## 10. Riscos

| Risco | Impacto | Mitigação |
| :--- | :--- | :--- |
| Motor TISS e webservices por operadora mais caros que o previsto | Atrasa P2 | Gerar XML próprio, envio manual na P1, parceiro avaliado para webservices ([[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]]) |
| Fadiga de alertas | Médico ignora o painel | Modo silencioso mede antes; severidade; só bloqueante interrompe; meta de aceite ≥ 70% |
| Vazamento de PII para a Spike | Incidente LGPD | SpikeAdapter único, DTOs estritos, teste de contrato que reprova qualquer campo fora da lista |
| Spike indisponível | Médico sem análise | Validações TISS locais no PEP; análise parcial; reanálise na conferência do faturista |
| Escopo do MVP grande para 90 dias | Atraso | P1 limitada a 2 especialidades, consulta + SP/SADT, envio manual de lote |
| Conflito de canal: Spike vendida a concorrentes | Cliente compara | Contrato público `v1` igual para todos; diferencial do PEP é a integração na origem ([[PRD-002-spike-standalone\|PRD-002]]) |
| Certificação SBIS | Bloqueia venda a hospitais | Requisitos NGS considerados desde P0 (auditoria, assinatura, imutabilidade); certificação na P3 |

## 11. Perguntas abertas

| # | Pergunta | Dono | Bloqueia | Proposta |
| :--- | :--- | :--- | :--- | :--- |
| Q1 | Faturamento TISS próprio ou parceiro? | Pedro + sócios | RFC-PEP-006 (envio), 007 | [[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]] |
| Q2 | Particular e convênio juntos desde o MVP? | Sócios | RFC-PEP-010 | [[ADR-PEP-0014-convenio-e-particular-no-mvp\|ADR-PEP-0014]] |
| Q3 | Onde roda o ASR da voz ambiental? | Pedro | RFC-PEP-013 | [[ADR-PEP-0013-asr-na-spike\|ADR-PEP-0013]] |
| Q4 | Hospedagem e residência de dados; política de backup | Pedro + jurídico | RFC-PEP-000 | [[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]] |
| Q5 | Estratégia de migração de concorrentes | Comercial | RFC-PEP-016 | [[ADR-PEP-0016-migracao-de-concorrentes\|ADR-PEP-0016]] |
| Q6 | Quais 2 especialidades no MVP? (default Ortopedia e Cardiologia) | CMO + piloto | RF-MOD-03 | — |
| Q7 | Quais 3–5 clínicas-piloto e quais operadoras predominam? | Comercial | Release plan §5 | — |
| Q8 | Papéis LGPD: clínica controladora, PEP operador, Spike suboperadora? | Jurídico | Dado real | [[conformidade-regulatoria\|conformidade]] |
| Q9 | Resolução CFM sobre IA citada (nº 2.454/2026) está vigente e com que exigências? | Jurídico + CMO | RF-SPK-04, RF-ASS-02 | **[validar]** |
| Q10 | Quem configura o modo Spike da clínica enquanto não existe [[SPK-01-api-de-clinicas-e-flags\|SPK-01]]? | Pedro | RF-SPK-07 | [[ADR-PEP-0006-modos-silencioso-assistido\|ADR-PEP-0006]] |
