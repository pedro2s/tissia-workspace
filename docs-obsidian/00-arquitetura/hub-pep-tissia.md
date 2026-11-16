---
projeto: PEP
tipo: arquitetura
status: ativo
tags:
  - tissia
  - pep
  - arquitetura
  - hub
  - indice
aliases:
  - README
  - tissia-docs-readme
origem: tissia/docs/README.md
---

# PEP Tissia — Documentação Técnica

Fonte de verdade para implementar o **PEP Tissia** (prontuário, agenda, faturamento TISS e gestão) e a integração nativa com a **Spike** (motor de IA anti-glosa, repositório `tissia-ai`, API `Tissia IA v1`), que também é vendida à parte.

Cada fase é executada RFC por RFC. Nenhuma RFC começa sem as ADRs dela aceitas. RFCs em **Rascunho** só têm escopo, dependências e perguntas abertas: detalhar antes de iniciar a fase, com o que se aprendeu na fase anterior.

## Os projetos e pacotes

| Peça | Localização | Stack | Papel |
| :--- | :--- | :--- | :--- |
| **UI do PEP** (`@pep/ui`) | `tissia/apps/ui` | React + Vite + TS, PWA, Tailwind + shadcn, TanStack Query | Telas do design (01–26), migradas progressivamente a cada RFC |
| **Backoffice interno** (`@pep/backoffice`) | `tissia/apps/backoffice` | React + Vite + TS; telas do design (01–18) estáticas por ora | Operação da plataforma e da Spike; só equipe interna ([[ADR-PEP-0020-backoffice-app-separado\|ADR-PEP-0020]]) |
| **Backend do PEP** (`@pep/api`) | `tissia/apps/api` | NestJS 12 (monólito modular), Drizzle ORM, PostgreSQL 16 + RLS, Redis/BullMQ | Domínio clínico, agenda, TISS, financeiro, SpikeAdapter |
| **Contratos compartilhados** (`@pep/contracts`) | `tissia/packages/contracts` | TypeScript + Zod estrito | Fonte única de DTOs, schemas `jsonb`, erros RFC 9457 e cálculos puros entre UI e API |
| **Spike (Tissia IA)** | `tissia-ai` (`/home/pedro/dev/pedro/tissia-ai`, repo separado) | Python 3.12, FastAPI, Cloud Run SP | Anti-glosa: regras TISS/ANS, DUT, justificativa Campo 23, sugestão TUSS/CID, preditivo, contratos, guias esquecidas. Produto vendável à parte |

A Spike tem documentação própria (PRD-001, RFC-000…014, ADR-0001…0012, [[manual-integracao-tissia-ia-v1|manual de integração]]). Este hub **não a duplica**: referencia e registra o que o PEP precisa dela. Links relativos para `../../tissia-ai/` funcionam com os dois repos lado a lado em `~/dev/pedro/`.

## Índice

| Tipo | Documento |
| :--- | :--- |
| Produto | [[PRD-001-pep-tissia\|PRD-001 — PEP Tissia]] |
| Produto | [[PRD-002-spike-standalone\|PRD-002 — Spike como produto standalone]] |
| Produto | [[PRD-003-backoffice\|PRD-003 — Backoffice interno]] (avaliação de necessidade e escopo por fase) |
| Marca | [[marca\|Marca Tissia: nome, logotipo, tipografia e cores]] |
| Glossário | [[glossario-pep\|Glossário (PEP; complementa o da Spike)]] |
| Arquitetura | [[modelo-de-dados-pep\|Modelo de dados do PEP]] |
| Integrações | [[integracoes-externas\|Catálogo de integrações externas]] |
| Conformidade | [[conformidade-regulatoria\|LGPD, CFM, ICP-Brasil, TISS/ANS, SBIS]] |
| Spike | [[mapeamento-pep-tissia-ia\|Spike no PEP: contrato, mapeamento e lacunas]] |
| Testes | [[test-plan-pep\|Test Plan]] |
| Release | [[release-plan-pep\|Release Plan]] |
| Demonstração | [[demo\|Ambiente de demonstração]] (Docker Compose + Caddy, `infra/demo/`) |
| Decisões | [[MOC-Index#04 — ADRs\|ADRs]] |
| Design | [[MOC-Index#02 — PEP\|RFCs]] |
| Design visual | [Canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d) (telas 01–26) |
| Design visual | [Canvas do backoffice](https://claude.ai/code/artifact/8a0b5083-bb44-4f07-966a-645b42e4546a) (telas 01–18) |

## Mapa de fases

| Fase | Objetivo | RFCs | Status |
| :--- | :--- | :--- | :--- |
| **D — Demo para investidores** (≈ 5–6 semanas) | Fatia real atendimento → guia SP/SADT → Spike → faturista e painel de glosa evitada sobre lote sintético; resto do PEP navegável como prévia ([[release-plan-pep#12-marco-d--mvp-de-demonstração-para-investidores\|release plan §12]]) | Fatias de 002, 003, 004, 005, 006, 012, 013, 015 | **Prioridade atual**: D1 e D2 entregues, D3 em andamento, voz ambiental (013) antecipada |
| **P0 — Fundações** (≈ 3 semanas) | Monorepo `pnpm` + `Turborepo`, `@pep/api` + `@pep/contracts`, infra, CI, tenancy com RLS + Drizzle, identidade e MFA, auditoria de acesso, base do design system em `@pep/ui` | [[RFC-PEP-000-fundacoes\|RFC-PEP-000]], [[RFC-PEP-001-identidade-e-acesso\|001]], [[RFC-PEP-015-frontend\|015]] | Em implementação |
| **P1 — MVP** (90 dias) | Cadastro, agenda, prontuário com templates (2 especialidades), pedidos de exame, guias de consulta e SP/SADT, Spike `analisar` + `feedback`, lote XML 4.03, modos silencioso/assistido | [[RFC-PEP-002-paciente-e-beneficiario\|002]], [[RFC-PEP-003-agenda\|003]], [[RFC-PEP-004-prontuario\|004]], [[RFC-PEP-005-spike-adapter\|005]], [[RFC-PEP-006-guias-e-lotes-tiss\|006]], [[RFC-PEP-021-dados-de-referencia-ans\|021]], [[RFC-PEP-014-plataforma-lgpd-auditoria\|014]] (núcleo) | Em implementação — ondas 0–6 entregues |
| **P2 — Ciclo de receita** | Retorno e glosas, contratos e autorização prévia, receita esquecida, financeiro e repasse, assinatura ICP-Brasil, voz ambiental, painel do gestor | [[RFC-PEP-007-retorno-e-glosas\|007]], [[RFC-PEP-008-contratos\|008]], [[RFC-PEP-009-receita-esquecida\|009]], [[RFC-PEP-010-financeiro-e-repasse\|010]], [[RFC-PEP-011-assinatura-icp-brasil\|011]], [[RFC-PEP-012-painel-do-gestor\|012]], [[RFC-PEP-013-voz-ambiental\|013]] | Em implementação — RFCs 007, 008, 010 fatia 1, 012 fatia 1 e 013 fatias F1–F5 entregues; ADRs 0012 e 0014 aceitas |
| **P3 — Expansão** | Internação, telemedicina, laboratórios/imagem, BI avançado, SBIS NGS1/NGS2, API pública e marketplace, migração de concorrentes | [[RFC-PEP-016-expansao\|016]], [[RFC-PEP-014-plataforma-lgpd-auditoria\|014]] (API pública) | Rascunho |

Ordem de execução no Marco D: D1 (002 + 003) → D2 (004) → D3 (005 + 006 rascunho) → D4 (mesa do faturista + 012) → D5 (ambiente). Cada RFC entra só com a fatia da demo; o resto fica em "Notas de implementação" e volta na P1.

Plano do MVP funcional por ondas (o que liga cada tela à API, na ordem): [[release-plan-pep#13-plano-do-mvp-funcional-ondas|release plan §13]].

Ordem de execução dentro da P1: 002 → 003 → 004 → 005 → 006. A 014 (auditoria, consentimento) entra junto com a 000/001 no que é transversal.

### Trilha do backoffice ([[PRD-003-backoffice|PRD-003]])

| Etapa | Objetivo | RFCs | Quando |
| :--- | :--- | :--- | :--- |
| **B0 — Clientes do PEP** | Acesso interno com FIDO2, trilha interna, onboarding de clínica sem SQL, acesso de suporte consentido | [[RFC-PEP-017-backoffice-fundacoes-e-acesso\|017]], [[RFC-PEP-018-backoffice-clinicas-e-onboarding\|018]] | Entregue em homologação em 15/11/2026 |
| **B1 — Cobrança** | Planos versionados, assinaturas, faturas, régua | [[RFC-PEP-019-backoffice-planos-e-cobranca\|019]] | P2, na primeira clínica pagante fora do piloto |
| **B2 — Operação da Spike** | Saúde, análises, chaves, estado de modelos/prompts/bases/regras/DUT (só leitura e ações reversíveis) | [[RFC-PEP-020-backoffice-operacao-da-spike\|020]] | SPK-06 aceita **e** > 10 clínicas na Spike ou 1º integrador externo |

Até a B0, clínicas de desenvolvimento e homologação continuam vindo do `seed:dev`.

## Telas do design → RFC

| Tela | RFC | Fase |
| :--- | :--- | :--- |
| 01a/01b Login, MFA, unidade | 001 | P0 |
| 02 Shell, barra de comandos, notificações | 015 | P1 |
| 03 Agenda | 003 | P1 |
| 04 Paciente e carteirinhas | 002 | P1 |
| 05 Prontuário · 08 Documentos · 09 Construtor · 10 Biblioteca · 11 Modelo preenchido | 004 | P1 |
| 06 Voz ambiental | 013 | P2 |
| 07 Revisão pós-consulta e assinatura | 013 + 011 (revisão de IA da evolução na P1 via 004/005) | P1/P2 |
| 12 Anti-glosa · 13 Revisão SP/SADT · 14 Análise parcial · 15 Modos | 005 | P1 |
| 16 Mesa do faturista · 17 Editor de guia · 18 Lotes | 006 | P1 |
| 19 Retorno · 20 Glosas e recurso | 007 | P2 |
| 21 Contratos | 008 | P2 |
| 22 Receita esquecida | 009 | P2 |
| 23 Financeiro e repasse | 010 | P2 |
| 24 Painel do gestor | 012 | P2 |
| 25 Perfis, auditoria, LGPD, integrações | 014 | P1 (núcleo) / P3 (API pública) |
| 26a–d Tablet e mobile | 015 | P1 |

Backoffice ([canvas](https://claude.ai/code/artifact/8a0b5083-bb44-4f07-966a-645b42e4546a)):

| Tela | RFC | Etapa |
| :--- | :--- | :--- |
| 01 Acesso (SSO, FIDO2, ambiente) · barras lateral e superior · 18 Papéis e trilha | 017 | B0 |
| 02 Visão geral (parcial) · 03 Clínicas · 04 Detalhe da clínica e acesso de suporte | 018 | B0 |
| 05 Planos e cobrança | 019 | B1 |
| 06–17 Integradores, observabilidade, análises, incidentes, modelos, prompts, avaliações, bases, regras, DUT, feedback, risco | 020 | B2 |

## ADRs

| # | Decisão | Status |
| :--- | :--- | :--- |
| [[ADR-PEP-0001-repositorios\|0001]] | UI, backend e Spike em repositórios separados; docs no repo da UI | Substituída parcialmente por 0018 |
| [[ADR-PEP-0002-backend-nestjs-monolito-modular\|0002]] | Backend NestJS 12, monólito modular (um módulo por domínio) | Aceita |
| [[ADR-PEP-0003-postgres-rls-multitenant\|0003]] | PostgreSQL 16 com Row-Level Security por tenant | Aceita |
| [[ADR-PEP-0004-frontend\|0004]] | React + Vite + TS, PWA, Tailwind + shadcn com os tokens do design, migração progressiva das telas geradas | Aceita |
| [[ADR-PEP-0005-spike-adapter\|0005]] | Camada anticorrupção (SpikeAdapter) e pseudonimização no PEP | Aceita |
| [[ADR-PEP-0006-modos-silencioso-assistido\|0006]] | Modo silencioso/assistido controlado pelo PEP; Spike sempre `visivel` para o PEP | Aceita |
| [[ADR-PEP-0007-prontuario-imutavel\|0007]] | Prontuário imutável: versões e adendos | Aceita |
| [[ADR-PEP-0008-eventos-outbox-bullmq\|0008]] | Eventos de domínio com outbox transacional + BullMQ | Aceita |
| [[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|0009]] | Hospedagem GCP `southamerica-east1` (mesma região da Spike) | Aceita |
| [[ADR-PEP-0010-identidade-keycloak\|0010]] | Identidade com Keycloak (OIDC, TOTP, WebAuthn/FIDO2, certificado ICP, SSO) | Aceita |
| [[ADR-PEP-0011-assinatura-icp-brasil\|0011]] | Assinatura ICP-Brasil via PSC em nuvem (PAdES) | Proposta |
| [[ADR-PEP-0012-motor-tiss\|0012]] | Geração de XML TISS própria; webservices por operadora via parceiro avaliado | Aceita |
| [[ADR-PEP-0013-asr-na-spike\|0013]] | ASR/voz ambiental como módulo da Spike (Scribe), vendável à parte | Aceita |
| [[ADR-PEP-0014-convenio-e-particular-no-mvp\|0014]] | MVP com convênio completo e particular simples | Aceita |
| [[ADR-PEP-0015-offline-prontuario\|0015]] | Offline do prontuário: PWA + fila local; conflito vira adendo | Aceita |
| [[ADR-PEP-0016-migracao-de-concorrentes\|0016]] | Migração de concorrentes por importadores de arquivo, não por integração direta | Proposta |
| [[ADR-PEP-0017-orm-drizzle-rls\|0017]] | Acesso a dados com Drizzle ORM mantendo isolamento RLS por transação | Aceita |
| [[ADR-PEP-0018-monorepo-pnpm-turborepo\|0018]] | Monorepo `pnpm` + `Turborepo` para UI, API e `@pep/contracts`; Spike permanece separada | Aceita |
| [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed\|0019]] | Drizzle Kit para migrações e seed; RLS por convenção em `drizzle/rls` | Aceita |
| [[ADR-PEP-0020-backoffice-app-separado\|0020]] | Backoffice interno como app separado no monorepo (`apps/backoffice`) | Proposta |
| [[ADR-PEP-0021-backoffice-backend\|0021]] | Backend do backoffice: módulo no monólito, processo e ingresso próprios, realm da equipe | Proposta |
| [[ADR-PEP-0022-backoffice-sem-dado-clinico\|0022]] | Backoffice sem dado clínico (papel `app_plataforma`); suporte só por acesso consentido no PEP | Proposta |
| [[ADR-PEP-0023-spike-no-backoffice\|0023]] | Spike no backoffice só pela API de operação; conteúdo editorial continua em git | Proposta |
| [[ADR-PEP-0024-dados-de-referencia-ans\|0024]] | Dados de referência da ANS: fonte oficial, versão com proveniência e alerta de defasagem | Aceita |
| [[ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai\|0025]] | Decisões de integração para Scribe (streaming) e `/ops/v1` na `tissia-ai` | Aceita |

**Aceita** = vem do documento de design de produto e arquitetura, ou foi aceita pelo responsável depois de revisada (0006, 0010 e 0013 em 09/11/2026; 0012, 0014 e 0024 em 13/11/2026). **Proposta** = recomendação deste hub para uma decisão em aberto (§12 do design de produto) ou uma lacuna; precisa de aceite antes da RFC que depende dela.

## Propostas para a Spike

Lacunas do contrato `v1` encontradas ao desenhar o PEP. Ficam aqui como proposta; quando aceitas, viram RFC no repo `tissia-ai` com a próxima numeração livre (RFC-011+).

| # | Proposta | Necessária para |
| :--- | :--- | :--- |
| [[SPK-01-api-de-clinicas-e-flags\|SPK-01]] | API de configuração de clínicas e flags (hoje só por script) | Onboarding de clínica no PEP; venda standalone |
| [[SPK-02-recurso-de-glosa\|SPK-02]] | Recurso de glosa assistido por IA | Tela 20 (P2) |
| [[SPK-03-processamento-assincrono\|SPK-03]] | Jobs assíncronos e webhooks para XML, contratos e esquecidas | Lotes grandes, rotinas diárias |
| [[SPK-04-requisitos-do-scribe\|SPK-04]] | Requisitos do PEP para o Scribe (RFC-009 da Spike) | Telas 06, 07, 08 (P2) |
| [[SPK-05-empacotamento-standalone\|SPK-05]] | Empacotamento standalone: portal do integrador, medição de uso, limites, SLA | PRD-002 |
| [[SPK-06-api-de-operacao\|SPK-06]] | API de operação interna (`/ops/v1`) para a equipe | Backoffice B2 ([[RFC-PEP-020-backoffice-operacao-da-spike\|RFC-PEP-020]]) |
| [[SPK-07-latencia-da-analise\|SPK-07]] | Latência da análise com IA (p95 ≤ 5 s com IA real) | Gate de saída do Marco D (§12.8) |
| [[SPK-08-fallback-local-sinalizado\|SPK-08]] | Fallback local da IA sinalizado (`degradado` / `origem_ia`) | Transparência na análise parcial e métrica `taxa_degradado` do piloto |
| [[SPK-09-catalogo-tuss\|SPK-09]] | Catálogo de terminologia TUSS e CID-10 (listagem e busca) | Pedido de exame ([[RFC-PEP-004-prontuario\|RFC-PEP-004]]) e guia SP/SADT ([[RFC-PEP-006-guias-e-lotes-tiss\|RFC-PEP-006]]) |
| [[SPK-10-tiss-4-03\|SPK-10]] | Leitura de XML TISS 04.03.00 no `ingerir` | Retorno e glosas ([[RFC-PEP-007-retorno-e-glosas\|RFC-PEP-007]]) e histórico de desfechos (P2) |
| [[SPK-11-contratos-auditar-substituicao\|SPK-11]] | Semântica de substituição e idempotência em `POST /v1/contratos/auditar` | Auditoria de contratos ([[RFC-PEP-008-contratos\|RFC-PEP-008]]) |
| [[SPK-12-desfecho-recurso-manual\|SPK-12]] | Desfecho de recurso manual sem sobrescrever o desfecho original | Retorno e glosas ([[RFC-PEP-007-retorno-e-glosas\|RFC-PEP-007]]) |
| [[SPK-13-ingestao-mapa-guia-ref\|SPK-13]] | Mapa `numeroGuiaPrestador` ↔ `guia_ref` e `lote_ref` em `POST /v1/tiss/lotes/ingerir` | Retorno e glosas ([[RFC-PEP-007-retorno-e-glosas\|RFC-PEP-007]]) e receita esquecida ([[RFC-PEP-009-receita-esquecida\|RFC-PEP-009]]) |

## Convenções

- Nome: o produto se chama **Tissia** desde 09/11/2026 (antes "MedScribe"; ver [[marca]]). Mudança de nome de marca é editorial e não conta como alteração de ADR aceita. `PEP` continua sendo a categoria do produto e o prefixo de IDs e pacotes.
- Idioma: português nos docs e no domínio; inglês só para termos técnicos consagrados. Mesmo padrão da Spike.
- IDs com prefixo para não colidir com os da Spike: `ADR-PEP-NNNN`, `RFC-PEP-NNN`, `SPK-NN`. Ao citar documento da Spike, escrever "Spike RFC-002", "Spike ADR-0007".
- ADR: imutável depois de aceita; mudança = nova ADR que a substitui.
- RFC: atualizar "Status" ao iniciar/concluir; desvios de implementação viram "Notas de implementação" no fim da RFC.
- Requisitos com ID estável (`RF-<FAMÍLIA>-NN`, `RNF-NN`) no PRD-001; RFCs citam e detalham por família sem colidir códigos. Famílias: FND (fundações técnicas da API/monorepo), BKO (backoffice, no PRD-003), ACE, PAC, AGE, PRO, DOC, MOD, SPK, FAT, RET, CON, ESQ, FIN, ASS, VOZ, BI, PLT, UI, REF (dados de referência da ANS); na P3 também INT (integrações clínicas) e TEL (telemedicina), a numerar quando a RFC-PEP-016 for detalhada.
- Eventos de domínio principais: `AtendimentoFechado`, `EvolucaoVersionada`, `GuiaRascunhoCriada`, `GuiaRevisada`, `GuiaValidada`, `AnaliseTissiaAiConcluida`, `FeedbackRegistrado`, `GuiaAssinada`, `DocumentoAssinado`, `LoteEmPreparacao`, `LoteGerado`, `LoteEnviado`, `RetornoImportado`, `DesfechoRegistrado`. Auxiliares de módulo: `PacienteCriado`, `BeneficiarioAtualizado`, `ElegibilidadeVerificada`, `ConsentimentoRevogado`, `AgendamentoCriado`, `AgendamentoAlterado`, `PacienteChegou`, `AtendimentoIniciado`.
- Refs enviados à Spike: `clinica_ref = "un_" + unidade.id`, `paciente.ref = "pa_" + base32(HMAC-SHA256(paciente.id, sal da clínica))[0:24]`, `guia_ref = "gu_" + guia.id`, `atendimento_ref = "at_" + atendimento.id` ([[RFC-PEP-005-spike-adapter|RFC-PEP-005]]). Modo da IA por unidade, padrão `desligado`.
- Afirmação regulatória, número de resolução ou código TUSS/CID/Tabela 38 usado como exemplo fica marcado **[validar]** até ser conferido na fonte oficial. Não inventar códigos; usar `<TUSS_…>` quando não souber.
- Nada de PII real em exemplos. Pacientes e profissionais são "de exemplo", como no design.
