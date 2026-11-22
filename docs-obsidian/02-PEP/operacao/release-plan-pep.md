---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - release-plan
  - roadmap
aliases:
  - release-plan
origem: tissia/docs/release-plan.md
---

# Release Plan — PEP Tissia

## 1. Visão geral

> **Estado atual (22/11/2026 — Pronto para Go-Live Piloto):** 100% do escopo técnico de **Marco D, P0, P1, P2, B0, B1, B2** e toda a remediação da auditoria clínica (Ondas A–F, PRs `#35`–`#41` na `main`, migrações `0000`–`0063`, 67 suítes / 564 testes `@pep/api` + UI + Backoffice + Contracts verdes) está concluído. Restam exclusivamente os **pré-requisitos operacionais e jurídicos de produção** listados abaixo e no §4.
>
> **Checklist objetivo de Go-Live (pré-piloto):**
> - [ ] Configurar segredos de produção: `TISSIA_AI_API_KEY` (`tsk_live_*`), credenciais PSC ICP-Brasil (Vidaas/BirdID), credenciais gateway NFS-e, Keycloak (realms PEP + Backoffice) e `DATABASE_URL`/`REDIS_URL`.
> - [ ] Validar DNS/TLS, backup PITR + restore testado (< 1 h) e âncora de auditoria em bucket GCS.
> - [ ] Fechar pré-requisitos jurídicos/LGPD (§4: DPA PEP ↔ Spike, termos com clínicas-piloto, RIPD) e pentest externo.
> - [ ] Iniciar rollout conforme §6: staging → produção silencioso (dias 1–15) → assistido 1 clínica (dias 16–22) → assistido todas (dias 23–30).

| Fase | Duração indicativa | Entrega | Gate de saída |
| :--- | :--- | :--- | :--- |
| **D — Demo para investidores** | ≈ 5–6 semanas | Fatia real do fluxo atendimento → guia → Spike → faturista, com o restante do PEP navegável como prévia, em ambiente de demonstração com dados sintéticos | [§12.8](#128-gate-de-saída) |
| P0 — Fundações | ≈ 3 semanas | Monorepo `pnpm` + `Turborepo` (`@pep/ui`, `@pep/api`, `@pep/contracts`), infra, CI, tenancy com RLS + Drizzle, identidade e MFA, auditoria, base do design system | [[test-plan-pep#10-critérios-de-saída-por-fase\|Test plan §10]] P0 + ADRs 0008, 0009 (aceitas) e 0010 aceita |
| P1 — MVP | 90 dias | Cadastro, agenda, prontuário (2 especialidades), pedidos, guias consulta e SP/SADT, Spike `analisar`+`feedback`, lote XML 4.03, modos | Test plan §10 P1 + pré-requisitos §4 + piloto silencioso iniciado |
| P2 — Ciclo de receita | a definir na saída da P1 | Retorno e glosas, contratos, esquecidas, financeiro, ICP-Brasil, voz, painel do gestor | Métricas do piloto (PRD-001 §4) + ADRs 0011, 0012, 0013 aceitas |
| P3 — Expansão | a definir | Internação, telemedicina, labs/imagem, BI avançado, SBIS, API pública | Definido na RFC-PEP-016 |

Durações assumem 1 dev (Pedro) com agentes. Gates decididos por Pedro, sócios e CMO. Sem data fixa: só se avança quando o gate passa.

## 2. Trilhas de versão

Três artefatos de deploy, versões independentes ([[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]]):

| Artefato | Localização | Versionamento | Deploy |
| :--- | :--- | :--- | :--- |
| UI (`@pep/ui`) | `tissia/apps/ui` | SemVer, tag `ui-vX.Y.Z` | Estático (CDN) |
| Backend (`@pep/api`) | `tissia/apps/api` | SemVer, tag `api-vX.Y.Z` | Cloud Run ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]]) |
| Spike (`tissia-ai`) | repo `tissia-ai` | SemVer próprio + `versao_bases` | Release plan da Spike |

- A UI declara a versão mínima da API que exige (`/v1/meta` devolve a versão); UI nova com API velha mostra aviso e não quebra.
- O PEP fixa o contrato **Spike `v1`**. Mudança incompatível da Spike sai em `/v2` com no mínimo 90 dias de convivência (Spike ADR-0005). O SpikeAdapter migra dentro desse prazo.
- Migrações de banco do PEP só aditivas na P1/P2: rollback de app nunca exige rollback de schema.

## 3. Ambientes e promoção

| Ambiente | Para quê | Spike usada | Deploy |
| :--- | :--- | :--- | :--- |
| local | Desenvolvimento | Stub do SpikeAdapter ou Spike local (`uv run fastapi dev`) | `docker compose` (Postgres, Redis, Keycloak) |
| `staging` | Integração, E2E, carga, homologação com clínicas-piloto (dados fictícios) | Spike `staging`, API key de staging | Automático a cada merge em `main` |
| `prod` | Piloto e clientes | Spike `prod`, API key de prod | Tag + aprovação manual |

Uma API key de integrador (`tsk_test_…` em sandbox, local e staging; `tsk_live_…` em produção) por ambiente, guardada no Secret Manager; nunca no repo nem no front.

## 4. Pré-requisitos para dado real

- [ ] Papéis LGPD definidos: clínica controladora, PEP operador, Spike suboperadora **[validar]** com jurídico ([[conformidade-regulatoria|conformidade]]).
- [ ] Contrato e termo de uso com as clínicas-piloto; DPA PEP ↔ Spike (mesmo que sejam da mesma empresa, a Spike é produto separado).
- [ ] RIPD concluído para o PEP (prontuário, agenda, faturamento, voz quando houver).
- [ ] Decidir `guardar_entrada_para_avaliacao` da Spike por clínica e corrigir o manual da Spike, que hoje diz que a evolução nunca é armazenada ([[mapeamento-pep-tissia-ia|spike/README]]).
- [ ] Revisão de segurança do test plan §6 feita; pentest externo sem achado alto.
- [ ] Backup e restore do banco testados (PITR) ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009]]).

## 5. Feature flags por clínica (PEP)

| Flag | Default | Efeito |
| :--- | :--- | :--- |
| `spike.modo` | `desligado` | `desligado`: não chama a Spike. `silencioso`: chama, guarda a análise, não exibe. `assistido`: exibe e envia feedback ([[ADR-PEP-0006-modos-silencioso-assistido\|ADR-PEP-0006]]) |
| `spike.gerar_justificativa` / `spike.sugerir_codigos` | `true` | Repassados em `opcoes` do `analisar` |
| `especialidades` | `[]` | Modelos de prontuário habilitados (P1: Ortopedia, Cardiologia **[validar]**) |
| `modulos` | P1 | Libera módulos P2 por clínica: `retorno`, `contratos`, `esquecidas`, `financeiro`, `assinatura`, `voz`, `bi` |
| `offline` | `false` | Rascunho offline do prontuário no tablet ([[ADR-PEP-0015-offline-prontuario\|ADR-PEP-0015]]) |

Do lado da Spike, a clínica (unidade) fica com `modo=visivel` para o PEP ler a análise completa mesmo no silencioso; quem esconde é o PEP. Atenção: a Spike autoprovisiona a clínica no primeiro `analisar` com flags `{}` e o código trata ausência como `visivel` (`app/api/guias.py:200`), embora o release plan da Spike diga `sombra`. Até existir [[SPK-01-api-de-clinicas-e-flags|SPK-01]], configurar cada unidade via `scripts/configurar_clinica.py` antes de ligar a flag no PEP.

`# ponytail:` flags em `jsonb` na tabela da clínica, alteradas pela tela 25 só por admin. Sem serviço de flags até haver dezenas de clínicas.

## 6. Rollout do piloto

1. **Homologação (staging):** 3–5 clínicas com convênio escolhidas (PRD-001 Q7); dados fictícios; agenda, prontuário e guia validados com médico e faturista de cada uma.
2. **Produção — silencioso (dias 1–15):** clínicas em `prod` com `spike.modo=silencioso`. Medimos glosa de linha de base e quantas guias glosadas tinham alerta. Gate: nenhum bloqueio de atendimento por causa do PEP; p95 dentro da meta; zero incidente de PII.
3. **Assistido — 1 clínica (dias 16–22):** acompanhamento diário com médico e faturista; feedback revisado na semana.
4. **Assistido — todas (dias 23–30):** gate de saída: aceite ≥ 70%, revisão p50 ≤ 15 s, ≥ 90% das guias sem bloqueante aberto ao chegar ao faturista.
5. **Decisão comercial:** relatório "glosas que teriam sido evitadas" + métricas do PRD-001 §4 → preço e próxima leva de clínicas.

## 7. Rollback

| Situação | Ação | Tempo |
| :--- | :--- | :--- |
| Bug de código no backend | Cloud Run: tráfego 100% para a revisão anterior | < 5 min |
| Bug na UI | Republicar o build anterior na CDN | < 5 min |
| Migração com problema | Migrações aditivas; app anterior roda com o schema novo | — |
| Spike com respostas erradas ou lenta | `spike.modo=silencioso` (ou `desligado`) nas clínicas afetadas; validações TISS locais seguem | < 5 min |
| Alerta clinicamente errado reportado | Clínica volta para silencioso; caso enviado à Spike para virar teste no golden set dela | < 5 min |
| Lote XML rejeitado pela operadora | Lote volta a "gerado"; corrigir e gerar nova versão; o XML anterior fica arquivado | horas |
| Falha no sync offline | Desligar `offline` na clínica; rascunhos locais exportáveis para não perder dado | < 30 min |

## 8. Monitoramento e alertas

| Métrica | Alerta |
| :--- | :--- |
| Disponibilidade (`/healthz` da API, UI) | Uptime check falhou 2× → Slack/e-mail |
| p95 por rota da API | > 1,5 s por 15 min |
| Latência da Spike (p95) | > 5 s por 15 min |
| Taxa de análise parcial (`degradado` ou timeout) | > 10% em 15 min |
| Erros 5xx da API do PEP | > 1% em 15 min |
| Falha de geração/validação de lote XML | Qualquer |
| Fila de eventos (outbox não publicado, jobs com falha) | Idade > 10 min ou DLQ > 0 |
| Taxa de aceite de sugestões | Semanal no piloto; queda > 10 p.p. → revisão |
| Acesso anômalo a prontuário | Usuário abrindo > N prontuários/h **[validar]** limiar |

## 9. Comunicação

| Evento | Quem é avisado | Como |
| :--- | :--- | :--- |
| Fim de fase (gate) | Pedro, sócios, CMO | Reunião + nota em `docs/` |
| Release em `prod` | Clínicas ativas | Notas de versão na central de notificações (tela 02) |
| Mudança no contrato da Spike que afeta o PEP | Pedro (como integrador) | Changelog do manual da Spike; aviso de 2 semanas |
| Incidente com dado ou indisponibilidade | Clínicas afetadas, DPO | Mesmo dia; post-mortem em `docs/incidentes/`; ANPD quando aplicável **[validar]** prazo |

## 10. Trilha da Spike standalone

A Spike segue o [[release-plan-ia|release plan próprio]]. Para a venda à parte ([[PRD-002-spike-standalone|PRD-002]]):

| Marco | Depende de | Gate |
| :--- | :--- | :--- |
| PEP Tissia como 1º cliente em `prod` | P1 deste plano | Piloto concluído |
| Venda a 1º integrador externo | [[SPK-01-api-de-clinicas-e-flags\|SPK-01]], [[SPK-05-empacotamento-standalone\|SPK-05]], manual corrigido | DPA padrão + SLA publicado |
| Arquivos grandes e rotinas diárias para integradores | [[SPK-03-processamento-assincrono\|SPK-03]] | Carga de XML de 5 MB sem timeout |
| Recurso de glosa | [[SPK-02-recurso-de-glosa\|SPK-02]] | Avaliação no golden set da Spike |
| Scribe e extensão Chrome | [[SPK-04-requisitos-do-scribe\|SPK-04]], Spike RFC-009/010 | Gates da Spike |

## 11. Trilha do backoffice

Deploy próprio: mesma imagem do `@pep/api` com entrypoint `backoffice` e o estático do `apps/backoffice`, em domínio e ingresso separados ([[ADR-PEP-0021-backoffice-backend|ADR-PEP-0021]]). Tags `backoffice-vX.Y.Z`.

| Marco | Depende de | Gate |
| :--- | :--- | :--- |
| B0 em homologação ✅ (concluída em 15/11/2026) | [[RFC-PEP-017-backoffice-fundacoes-e-acesso\|RFC-PEP-017]], [[RFC-PEP-018-backoffice-clinicas-e-onboarding\|018]] | E2E de onboarding verde; `app_plataforma` sem grant clínico (teste) |
| B0 em produção | B0 em homologação, realm da equipe com FIDO2 em `prod` | **Antes da 1ª clínica do piloto em `prod`** (release plan §6, passo 2). A partir daí, zero SQL com `app_admin` fora de migração |
| B1 | [[RFC-PEP-019-backoffice-planos-e-cobranca\|RFC-PEP-019]] | Primeira clínica pagante fora do piloto |
| B2 | [[SPK-06-api-de-operacao\|SPK-06]] no `tissia-ai`, [[RFC-PEP-020-backoffice-operacao-da-spike\|RFC-PEP-020]] | > 10 clínicas na Spike ou 1º integrador externo |

Rollback do backoffice não afeta PEP nem Spike: é só reverter o serviço `pep-backoffice-<amb>`.

## 12. Marco D — MVP de demonstração para investidores

### 12.1 Objetivo

Mostrar a investidores, ao vivo e em até 15 minutos, três coisas:

1. **O PEP é um produto inteiro e bem desenhado:** agenda, prontuário, faturamento, glosas e gestão aparecem navegáveis.
2. **A Spike funciona de verdade:** uma guia montada a partir de um atendimento recebe na hora alertas TISS/ANS, checklist da DUT com evidência literal, justificativa do Campo 23 e sugestão de códigos, com IA real.
3. **Há dinheiro em jogo e ele é mensurável:** glosas que teriam sido evitadas, em reais, sobre um lote de guias com desfecho conhecido.

Não é piloto e não recebe dado real. Não precisa de LGPD operacional completa, auditoria encadeada, ICP-Brasil, lote XML, financeiro nem backoffice.

### 12.2 Roteiro da demo

| # | Passo | Real ou prévia |
| :--- | :--- | :--- |
| 1 | Login da médica de exemplo e escolha da unidade (telas 01a/01b) | **Real** (já entregue na P0) |
| 2 | Agenda do dia com 6 a 8 pacientes de exemplo; abrir o próximo atendimento (tela 03) | **Real**, em visão simplificada (só Dia) |
| 3 | Prontuário: evolução SOAP de Ortopedia (caso de LCA do golden set), pedido de RM e de artroscopia com indicação clínica (telas 05/08) | **Real**: editor em blocos com comandos `/`, linha do tempo, pedido, documentos (receita, atestado, encaminhamento, resumo ao paciente) e fechamento |
| 4 | Fechar atendimento → guia SP/SADT rascunho → análise da Spike; painel anti-glosa com risco, alertas, DUT 110 com evidência, Campo 23 sugerido e editável (telas 12/13) | **Real**, com a Spike e o Gemini |
| 5 | A médica aceita a justificativa, edita uma frase e rejeita uma sugestão; a reanálise baixa o risco | **Real** (feedback e reanálise com o mesmo `guia_ref`) |
| 6 | Derrubar a IA (flag) e mostrar "análise parcial": regras TISS seguem e o atendimento não trava (tela 14) | **Real** |
| 7 | Mesa do faturista com as guias do dia ordenadas por risco (tela 16) | **Real**, só lista e filtro por risco |
| 8 | "Glosas que teriam sido evitadas": lote sintético com desfecho conhecido, em R$ por operadora e motivo (telas 15/24) | **Real sobre dados sintéticos**, com aviso visível |
| 9a | Voz ambiental: consentimento, escuta com transcrição por locutor, estruturação em SOAP pela Spike e revisão trecho a trecho até assinar (telas 06/07) | **Real**, com o Gemini (fatia antecipada da RFC-PEP-013; sem IA, cai no classificador local da Spike) |
| 9 | Tour rápido: lotes XML, retorno e glosas, contratos, receita esquecida, financeiro, painel do gestor, tablet (telas 06–26) | **Prévia** (telas estáticas existentes) |
| 10 | Opcional: backoffice e a Spike vendida a outros PEPs (canvas do backoffice, PRD-002) | **Prévia** |

O que é prévia aparece marcado como "Prévia" na barra lateral. É honestidade com o investidor e evita a pergunta "isso funciona?" sem resposta.

### 12.3 Escopo real por RFC (fatia da demo)

Cada RFC entra só com a fatia abaixo. O resto fica registrado em "Notas de implementação" da própria RFC como pendente da P1.

| RFC | Entra na demo | Fica para a P1 |
| :--- | :--- | :--- |
| [[RFC-PEP-002-paciente-e-beneficiario\|002]] Paciente | Cadastro mínimo (nome, sexo TISS, nascimento, 1 carteirinha) via seed; ficha só leitura | Mascaramento e revelação auditada, várias carteirinhas, elegibilidade, consentimentos, KMS por tenant |
| [[RFC-PEP-003-agenda\|003]] Agenda | Visão Dia de um profissional, abrir atendimento, status | Semana/Salas, conflitos, encaixe, WhatsApp, autorização na agenda |
| [[RFC-PEP-004-prontuario\|004]] Prontuário | Evolução SOAP com versões insert-only, pedido de exame/procedimento com TUSS do catálogo da Spike (só os códigos das 6 DUTs curadas + consulta), fechar atendimento → `AtendimentoFechado`; antecipados: editor em blocos com comandos `/`, linha do tempo, documentos da tela 08 com assinatura imutável e alergias | Anexos |
| [[RFC-PEP-005-spike-adapter\|005]] SpikeAdapter | `analisar` e `feedback` reais, pseudonimização (`paciente.ref` HMAC), timeout 6 s, `degradado`, painel anti-glosa (telas 12/13/14) com a aba "Enviado à Spike" (payload real, para mostrar que só vão ref, sexo e idade), modo `assistido` por unidade | Modo silencioso, truncamento fino, observabilidade completa |
| [[RFC-PEP-006-guias-e-lotes-tiss\|006]] Guias | Guia SP/SADT rascunho a partir do atendimento, revisões com o mesmo `guia_ref`, Campo 23 ≤ 500, mesa do faturista (lista por risco) | Guia de consulta, editor completo, validações TISS locais, lote XML 4.03, XSD |
| [[RFC-PEP-013-voz-ambiental\|013]] Voz ambiental | Antecipada: consentimento registrado, escuta em trechos de áudio por HTTP, Scribe da Spike (transcrição e estruturação), revisão trecho a trecho, zero retenção de áudio (telas 06/07) | Streaming por WebSocket, termo versionado assinado, modelos por especialidade, tablet/mobile, RIPD da gravação |
| [[RFC-PEP-012-painel-do-gestor\|012]] Painel do gestor | Um painel de glosa evitada calculado sobre o lote sintético (§12.5) | Projeções por evento, linha de base real, PDF |
| [[RFC-PEP-015-frontend\|015]] Frontend | Componentes que o painel anti-glosa pede (selo de risco, aviso de IA, trecho de IA, abas); selo "Prévia" no menu | Restante do design system |

Ficam fora da demo: RFC-PEP-001 (pendências: recuperação, Keycloakify, dispositivo confiável), 007–011, 014, 016 e a trilha do backoffice (§11).

**Invariantes que valem mesmo na demo:** só `paciente.ref`, sexo e idade vão para a Spike; falha da Spike não trava nada; nada da IA entra na guia sem aceite humano registrado; evolução fechada não é editada; RLS em toda tabela de tenant; nada de PII nem texto clínico em log. Custa pouco manter, e é o que um investidor técnico vai perguntar.

### 12.4 Sequência de entregas

| # | Entrega | Estimativa | Pronto quando |
| :--- | :--- | :--- | :--- |
| D1 | Paciente mínimo + agenda do dia + seed da demo (pacientes e agenda; os casos do golden set entram no D2) | 1 semana | Médica abre o dia e entra no atendimento. **Entregue em 08/11/2026** (notas nas RFC-PEP-002 e 003) |
| D2 | Atendimento: evolução SOAP versionada, pedido de exame/procedimento com TUSS, fechamento | 1 semana | Fechar gera `AtendimentoFechado` e guia rascunho. **Entregue em 08/11/2026** (notas na RFC-PEP-004) |
| D3 | SpikeAdapter + painel anti-glosa + feedback + reanálise + análise parcial | 1,5–2 semanas | Passos 4–6 do roteiro funcionam com a Spike real. **Em andamento:** análise, painel, feedback, reanálise e análise parcial entregues em 08–09/11/2026, com o Gemini 3.x (notas na RFC-PEP-005); circuit breaker e modo por unidade (silencioso | assistido) entregues em 09/11/2026; E2E do roteiro entregue (ver D5) |
| D4 | Mesa do faturista por risco + painel de glosa evitada sobre o lote sintético | 1 semana | Passos 7–8 do roteiro com números reproduzíveis. **Entregue em 09/11/2026** (notas nas RFC-PEP-006 e 012) |
| D5 | Ambiente de demo, selo "Prévia", reset em 1 comando, roteiro ensaiado e vídeo gravado | 0,5–1 semana | Gate §12.8. **Em andamento:** selo "Prévia", `pnpm demo:reset`, banner, E2E do roteiro (`pnpm e2e:demo`, com login real), realm `pep-demo`, ambiente Docker em `infra/demo/` e medição de latência entregues em 09/11/2026; faltam a VM do GCP, a CI com E2E, a latência da análise (SPK-07) e o vídeo narrado |

D1–D4 entram como E2E (Playwright) do roteiro: o teste é o próprio roteiro da demo.

### 12.5 Dados da demo

- **100% sintéticos.** Pacientes e profissionais "de exemplo", como no design. Nenhum dado real, nem anonimizado.
- **Casos clínicos:** os do golden set da Spike (`tissia-ai/evals/casos/sinteticos.jsonl`, revisados pelo CMO): Ortopedia (DUT 65, 110) e Cardiologia (DUT 3, 10, 38, 54). Códigos TUSS e CID vêm das bases da Spike, nunca inventados.
- **Lote para "glosa evitada":** cerca de 200 guias geradas a partir dos casos do golden set (com variações), analisadas pela Spike e com desfecho sintético atribuído por regra explícita: guia não conforme com a DUT → glosa, guia conforme → paga. Valores por procedimento de uma tabela de preços sintética e marcada como tal. O painel mostra "simulação sobre guias sintéticas" em destaque.
- **Reset:** `pnpm demo:reset` recria o banco e roda o seed. A agenda fica sempre no "hoje" da demo.

### 12.6 Ambiente de demonstração

| Item | Decisão |
| :--- | :--- |
| Hospedagem | `# ponytail:` uma VM no GCP `southamerica-east1` com `docker compose` (UI, API, worker, Keycloak, Postgres, Redis, Spike) atrás de Caddy (HTTPS automático). É descartável, e não é a arquitetura de produção ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]] volta na P1) |
| Domínio | `demo.<domínio>` com banner fixo "Ambiente de demonstração · dados fictícios" |
| Login | Realm `pep-demo`: usuários de demo sem segundo fator, para não travar ao vivo. A tela 01b mostra o MFA como prévia; o MFA real existe e tem E2E na P0 |
| Spike | A mesma imagem do `tissia-ai` (sem fork), integrador `demo`, decisor `gemini` (Vertex SP), clínica com `modo=visivel` explícito |
| Plano B | Vídeo do roteiro completo gravado no ambiente; notebook com `docker compose` local caso a rede falhe |
| Custo | VM e2-standard-4 + Vertex AI: na casa de US$ 100–150/mês **[validar]** |

Os pré-requisitos para dado real (§4) **não** se aplicam, porque não há dado real. Se um investidor ou clínica quiser testar com dado próprio, a resposta é "no piloto (P1)".

### 12.7 Material de apoio para o deck

| Número | Fonte | Cuidado |
| :--- | :--- | :--- |
| Recall de critério DUT ausente, precisão de alerta, Top-3 TUSS/CID, latência | [[demo-numeros-do-deck\|Reavaliação de 09/11/2026 com Gemini real e a Spike refinada]] | **Não citar a avaliação de 04/11/2026** ([[2026-11-04-decisor\|relatório]]): rodou na heurística local, sem IA. Com a Spike refinada (`e469883`), em duas rodadas: recall DUT 100%, recall@50 100%, precisão de alerta ≈ 88–89% (meta 80% **atendida**), Top-3 CID 93,3%, Top-3 TUSS 95–100% (não citar "100%": varia entre rodadas). Dizer que é golden set sintético (60 casos), não produção. **Não citar** latência do harness nem custo por guia (constante no código, não medido) |
| Glosa evitada em R$ | Painel da demo (§12.5) | Sempre "simulação sobre guias sintéticas"; nunca apresentar como resultado de clínica |
| Latência p95 da análise | Medida no ambiente de demo | Citar com o volume medido |
| Tamanho do problema (glosa no Brasil) | Fonte pública citável **[validar]** | Não usar número sem fonte |

### 12.8 Gate de saída

Estado em 09/11/2026 (✅ comprovado · ⬜ pendente):

- ⬜ Roteiro §12.2 executado de ponta a ponta 3 vezes seguidas **no ambiente de demo**, sem intervenção técnica. *(O E2E `pnpm e2e:demo` cobre os passos 1–8 e 9a, com login real no Keycloak (`pep-demo`) e a Spike com Gemini, e passa localmente em ~1,5 min; falta executar na VM do GCP (instruções em [[demo|demo.md]]). Quando o Gemini estoura o tempo da 2ª fase, o painel mostra "não ficaram prontas agora" e o apresentador aperta "Reanalisar".)*
- ✅ E2E do roteiro verde na CI. *(Workflow `.github/workflows/e2e-demo.yml` (push na `main` e manual): sobe Postgres, Redis e Keycloak (realm `pep-demo`), faz checkout da Spike (`tissia-ai`, privado) por deploy key **somente leitura** (`SPIKE_DEPLOY_KEY`), sobe a Spike com `IA_PROVEDOR=local` e roda os 4 testes do roteiro com login real: verde no GitHub em ~51 s. **Limite:** na CI a IA é a heurística local (sem Vertex no runner), então a decisão trecho a trecho da voz e o Gemini continuam validados só localmente (`pnpm e2e:demo`). O log da Spike não é publicado (o repositório do PEP é público).)*
- ✅ Análise com IA em ≤ 5 s p95. *(A análise passou a vir em **duas fases** (09/11/2026): risco, alertas e checklist da DUT na 1ª fase, medida em **p50 1,9 s / p95 2,6 s**; justificativa do Campo 23 e sugestões de código chegam em segundo plano (p95 7,6 s) e o painel avisa "Gerando…". No navegador, risco e alertas aparecem em ~2,4 s após o agrupamento de edições da tela. Medição: `pnpm --filter @pep/api medir:latencia`. A [[SPK-07-latencia-da-analise|SPK-07]] segue útil para encurtar a análise completa.)* ✅ Queda da IA → "análise parcial" imediata e sem travar (circuit breaker, teste de API e E2E do passo 6).
- ✅ Nenhuma chamada à Spike com campo além de `paciente.ref`, sexo e idade (teste do adapter em `apps/api/test/spike.spec.ts`; o E2E também confere que a aba "Enviado" não mostra nome, código nem data de nascimento).
- ✅ Toda tela não implementada marcada como "Prévia" (selo no menu lateral; some nas páginas que são a fatia real).
- ✅ `pnpm demo:reset` em menos de 2 min (~5 s localmente; ~9 s no ambiente Docker). ⬜ Vídeo de backup *narrado*: o E2E já grava vídeo de cada passo (`apps/e2e/test-results-demo/*/video.webm`), que serve de plano B técnico, mas a gravação ensaiada com narração não existe.
- ✅ Números do deck conferidos (§12.7). *(Os de 04/11 não vinham de IA. Reproduzidos em 09/11 com Gemini real e a Spike refinada, em duas rodadas: precisão de alerta ≈ 88–89% (meta de 80% atendida), recall DUT 100%; Top-3 TUSS varia de 95% a 100%. Ressalva: duas rodadas, não uma distribuição. Ver [[demo-numeros-do-deck|demo-numeros-do-deck.md]].)*

Autenticação da demo: login real pelo Keycloak (realm `pep-demo`, usuários fictícios `dra.exemplo`, `dr.exemplo.b`, `faturista.exemplo`, `gestora.exemplo`, `recepcao.exemplo`, senha `demonstracao-pep`, sem segundo fator). Nenhum tenant ou unidade vem de variável de ambiente: o login resolve os vínculos. `MFA_AMR_ACEITOS=pwd` (aceita só senha) é recusado pela API em produção. O banner "Ambiente de demonstração · dados fictícios" liga com `VITE_AMBIENTE_DEMO=true`.

### 12.9 Riscos

| Risco | Mitigação |
| :--- | :--- |
| A IA demora ou falha ao vivo | Passo 6 transforma isso em demonstração de resiliência; vídeo de backup |
| Investidor pergunta sobre dado real e LGPD | Mostrar a pseudonimização no request (passo 4, painel "o que foi enviado") e as ADRs 0005/0007 |
| Prévia confundida com produto pronto | Selo "Prévia" e a frase no roteiro: "o que vocês viram funcionando é o núcleo; o resto é o design aprovado" |
| A demo vira o produto e a P1 herda atalhos | Atalhos ficam em "Notas de implementação" das RFCs; a VM é descartada na P1; o realm `pep-demo` nunca vai para `prod` |

### 12.10 Depois da demo

A P1 retoma as RFCs 002–006 a partir da fatia entregue (nada é jogado fora: schema, adapter, painel e mesa são os definitivos) e soma o que ficou de fora: modo silencioso, validações TISS locais, lote XML, auditoria (014), backoffice B0 e o ambiente `staging`/`prod` em Cloud Run.

## 13. Plano do MVP funcional (ondas)

Objetivo: tirar o selo "Prévia" de todas as telas do design, ligando cada uma à API na ordem do mapa de fases. Cada onda segue a RFC indicada (ler a RFC e as ADRs antes), migra as próprias rotas para o caminho canônico da [[RFC-PEP-015-frontend#roteamento|RFC-PEP-015]] e confere a tela contra o [canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d). O que depende de outra onda fica visível e inerte, com o motivo no `title`.

### 13.1 P1 — MVP

| Onda | RFC | Entrega | Telas | Estado |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 015 | Remover telas estáticas que duplicavam fluxos reais; selo "Prévia" no Início; canvas como fonte canônica no `CLAUDE.md`; aceite das ADRs 0006, 0010 e 0013 | — | ✅ 09/11/2026 |
| 1 | 002 + núcleo da 014 | Cadastro completo com PII cifrada, revelar auditado, carteirinhas, elegibilidade manual, consentimentos, histórico de guias; trilha `auditoria_eventos` | 04 | ✅ 09/11/2026 ([[RFC-PEP-002-paciente-e-beneficiario#notas-de-implementação\|notas]]) |
| 2 | 003 + 015 | Criar, remarcar e cancelar agendamento (conflito com `EXCLUDE USING gist`), encaixe, Semana/Salas; elegibilidade da agenda derivada de `verificacoes_elegibilidade`; "Agendar" da ficha; Início "Meu dia" com agenda e mesa reais | 02, 03 | ✅ 09/11/2026 ([[RFC-PEP-003-agenda#notas-de-implementação\|notas]]) |
| 3 | 004 | Adendo após fechado, atendimento avulso, módulo `modelos` (biblioteca, construtor com `@dnd-kit`, modelo preenchido), PDF e impressão dos documentos, documentos por paciente (aba da tela 04), offline do prontuário (ADR-PEP-0015), "quem viu" do prontuário | 05, 08, 09–11, 26a | ✅ 10/11/2026: adendo, avulso, quem viu, documentos do paciente, PDF/impressão, modelos e offline da evolução entregues ([[RFC-PEP-004-prontuario#notas-de-implementação\|notas]]); fora: rascunho de documento offline e leitura offline |
| 4 | 005 | Consumidores do outbox com uma fila por consumidor (ADR-PEP-0008), SSE, status real da IA na Topbar, Modos completo (comportamento e exceções), catálogo TUSS vindo da Spike, campos que a 012 exige (`analises_tissia_ai.modo`, `exibida_em`) | 12–15 | ✅ 11/11/2026: status real da IA, Modos completo, `modo`/`exibida_em`, consumidores do outbox (uma fila por consumidor, retry e DLQ), feedback pela fila, SSE da análise e catálogo TUSS/CID-10 da Spike ([[SPK-09-catalogo-tuss\|SPK-09]]) entregues ([[RFC-PEP-005-spike-adapter#notas-de-implementação\|notas]]) |
| 5 | 006 (ADRs 0012 e 0014 aceitas em 13/11/2026) | Guia `/faturamento/guias/:id` com revisões e máquina de estados, validações TISS locais, guia de consulta, ações em lote, filtros da mesa com `nuqs`, lote XML 4.03 com XSD; fatia 0 da 008 (catálogo de operadoras) antecipada; **antes:** fatias 1–3 da [[RFC-PEP-021-dados-de-referencia-ans\|021]] (XSD 4.03.00, campos e domínios da guia, Tabela 38, CADOP) | 13, 16, 17, 18, 26c, 26d | ✅ 14/11/2026: guias SP/SADT e de consulta com revisões e máquina de estados, validações locais (obrigatoriedade, formato, domínios e registro ANS no CADOP), lote XML 4.03.00 validado no XSD por tipo, envio manual, Mesa com filtros, abas e ações em lote (p95 de 32 ms com 5.000 guias), catálogo de operadoras e fatias 1–3 da 021. Fora, e registrado na [[RFC-PEP-006-guias-e-lotes-tiss#notas-de-implementação\|RFC-PEP-006]]: regras da operadora (backoffice/contrato), sexo e idade, autorização, conferência do hash TISS com lote oficial e assinatura do XML |
| 6 | 014 (núcleo) + 015 | Perfis e guarda de rota, menu filtrado por permissão, selagem por hash da trilha, tela 25 (núcleo), notificações reais, barra de comandos global (`Ctrl+K`) | 25, Topbar | ✅ 14/11/2026 |

### 13.2 P2 — Ciclo de receita

As RFCs foram detalhadas em 09/11/2026 e têm "Fatias de entrega" próprias. Ordem sugerida pelas dependências entre elas:

| Ordem | RFC | Pode começar quando | Bloqueios para começar |
| :--- | :--- | :--- | :--- |
| 1 | [[RFC-PEP-008-contratos\|008]] Contratos (tela 21) | Onda 5 concluída | ✅ 15/11/2026: fatias 1–6 + [[SPK-11-contratos-auditar-substituicao\|SPK-11]] ([[RFC-PEP-008-contratos#notas-de-implementação\|notas]]); fora: XLSX e E2E dedicado |
| 2 | [[RFC-PEP-010-financeiro-e-repasse\|010]] fatia 1 (particular com recibo) | Pode ir para a P1 (lacuna da ADR-PEP-0014) | ✅ 15/11/2026: tabela particular, recebível, recebimento/desconto, estorno, recibo PDF, `RecebimentoRegistrado` e UI no atendimento ([[RFC-PEP-010-financeiro-e-repasse#notas-de-implementação\|notas]]); NFS-e fora do MVP (ADR-PEP-0014) |
| 3 | [[RFC-PEP-007-retorno-e-glosas\|007]] Retorno e glosas (telas 19, 20) — **implementada (pendente SPK-12/SPK-13)** | 006 completa (lote enviado) | Q10 resolvida (Tabela 38 pela [[RFC-PEP-021-dados-de-referencia-ans\|021]]) |
| 4 | [[RFC-PEP-009-receita-esquecida\|009]] Receita esquecida (tela 22) | Fatias 1–3 com a P1 concluída | Q7 só para a fatia 4 (fonte clínica) |
| 5 | [[RFC-PEP-010-financeiro-e-repasse\|010]] restante (tela 23) | 007 entregue | Q1 (OFX × CNAB), Q2–Q4 (modelo de repasse) |
| 6 | [[RFC-PEP-012-painel-do-gestor\|012]] Painel do gestor (tela 24) | Fatia 1 entregue em 15/11/2026 ([[RFC-PEP-012-painel-do-gestor#notas-de-implementação\|notas]]) | Q2 (mapa alerta → Tabela 38) e RFC-PEP-007 para a fatia 2 |
| 7 | [[RFC-PEP-011-assinatura-icp-brasil\|011]] ICP-Brasil (telas 07, 08) | Fatias 0–1 já (adaptador `simulado`) | ADR 0011; Q1 (escolha do PSC); Q4 (política de assinatura) |
| 8 | [[RFC-PEP-013-voz-ambiental\|013]] restante (telas 06, 07, 26b) | Fatias F1–F5 já | Q2, Q4, Q9 (jurídico/DPO) antes de `prod`; SPK-04 etapa 2 para o streaming (F7) |

### 13.3 Pendências levantadas no detalhamento da P2

Não alteradas ainda; entram junto com a RFC que as usa:

- Propostas de SPK a abrir: mapear número da guia ↔ `guia_ref` no `ingerir` e não deduplicar (007 Q7); desfecho de recurso sem sobrescrever o original (007 Q8); `contratos/auditar` com semântica de substituição (008 Q3); `natureza` das sugestões e `esquecidas` por `paciente_ref` (009 Q8/Q9); PCM16 e campo `marcado` no Scribe (013 Q6).
- [[modelo-de-dados-pep|Modelo de dados]]: dinheiro em centavos inteiros (`*_centavos`), não `numeric(12,2)`.
- Eventos auxiliares novos a listar nas convenções: `RetornoConciliado`, `RecursoEnviado`, `ContratoAtualizado`, `AuditoriaContratoConcluida`, `ExecucaoReceitaEsquecidaConcluida`, `PendenciaCobrancaDecidida`, `RecebimentoRegistrado`, `RepasseAprovado`.
- Permissões novas propostas (RFC-PEP-001 e `PERMISSOES_POR_PERFIL`): `retornos:*`, `glosas:*`, `contratos:*`, `financeiro:receber_particular`, `financeiro:aprovar_repasse`, `repasse:ler_proprio`, `gestao:ler`, `escuta:usar`. A `pacientes:revelar` já entrou na onda 1.
- Requisitos propostos ao PRD sem colisão: RF-ESQ-03..05, RF-FIN-05..06.

### 13.4 Sequência das próximas fases

Definida em 13/11/2026, com as ADRs 0012, 0014 e 0024 aceitas e a onda 5 em andamento. A ordem segue duas regras: (1) o que o piloto silencioso (§6) precisa para medir vem antes dele; (2) o que depende de lote realmente enviado ou de decisão jurídica começa a esperar cedo, em paralelo.

| # | Etapa | RFC | Entrega | Pronto quando |
| :--- | :--- | :--- | :--- | :--- |
| 1 | **Onda 5** ✅ (concluída em 14/11/2026) | 006 + fatia 0 da 008 | Ordem interna: máquina de estados e revisões da guia → validações TISS locais de formato e domínios (XSD e Tabela 38; as de quantidade e excludentes da operadora esperam o B0, regras cadastradas no backoffice) → guia de consulta → lote XML 4.03.00 com XSD (`numeroLote` único por operadora, ≤ 12 caracteres; assinatura configurável por operadora, desligada por padrão) → ações em lote e filtros `nuqs`; catálogo de operadoras (008 fatia 0) | Lote XML válido no XSD da 021 gerado a partir de guias fechadas. Perguntas 1–4 da RFC-PEP-006 resolvidas em 14/11/2026 |
| 2 | **Onda 6** ✅ (concluída em 14/11/2026, com pendências nas notas das RFCs 014 e 015) | 014 núcleo + 015 | Perfis e guarda de rota, menu por permissão, selagem por hash da trilha, tela 25, notificações, `Ctrl+K` | Perfil sem permissão não alcança a rota nem pela URL (E2E por perfil) |
| 3 | **Medição do piloto** ✅ (fatia 1 entregue em 15/11/2026) | 012 fatia 1 | Projeções por evento e linha de base real, no lugar do lote sintético. **Antecipada**: a decisão comercial (§6, passo 5) usa o relatório "glosas que teriam sido evitadas" e a linha de base só existe se a medição estiver ligada desde o dia 1 do silencioso | Painel lê `analises_tissia_ai.modo`/`exibida_em` de clínica real em homologação ([[RFC-PEP-012-painel-do-gestor#notas-de-implementação\|notas]]) |
| 4 | **B0 do backoffice** ✅ (concluída em 15/11/2026) | 017 → 018 | Acesso interno com FIDO2, onboarding de clínica sem SQL, suporte consentido (§11) | E2E de onboarding verde; é gate de `prod` e vem **antes** da 1ª clínica real |
| 5 | **Portão do piloto** | ADR-PEP-0009 + SPK-01 | `staging` e `prod` em Cloud Run, clínica da Spike configurada por `configurar_clinica.py`, pré-requisitos do §4 (RIPD, DPA, PITR, pentest) | §4 todo marcado; piloto silencioso iniciado (§6, passo 2) |
| 6 | **P2, durante o piloto** | **007 (implementada, pendente SPK-12/SPK-13)** → **010 fatia 1 ✅** → ~~008 (restante)~~ ✅ (15/11/2026) → 009 fatias 1–3 | Retorno e glosas só faz sentido com lote enviado (fim da etapa 5); recibo particular (010 f1) entregue em 15/11/2026; contratos (008 fatias 1–6 + [[SPK-11-contratos-auditar-substituicao\|SPK-11]]) entregue ([[RFC-PEP-008-contratos#notas-de-implementação\|notas]]); receita esquecida usa o lote real | Cada RFC com seu gate de saída |
| 7 | **P2, restante** | 010 restante → 011 → 013 restante → 009 fatia 4 | Repasse (Q1–Q4), ICP-Brasil (ADR 0011, escolha do PSC), voz (streaming, SPK-04 etapa 2) | Decisões abertas resolvidas |
| 8 | **B1 e B2** | 019, 020 | Cobrança na 1ª clínica pagante; operação da Spike com > 10 clínicas ou 1º integrador externo (§11) | Gatilhos do §11 |

**Mudanças em relação ao §13.2:** a 012 (fatia 1) sobe da posição 6 para a etapa 3, e a 010 fatia 1 deixa de ser "P2 pura" porque a ADR-PEP-0014 já foi aceita. A 007 não começa antes de existir lote enviado, mesmo com a Q10 resolvida pela 021.

**Em paralelo e sem código (começar já, têm prazo externo):**
- Jurídico/DPO: papéis LGPD, DPA PEP ↔ Spike e RIPD (§4), e Q2/Q4/Q9 da RFC-PEP-013, antes de qualquer `prod` com voz.
- Escolher as 3–5 clínicas-piloto (PRD-001 Q7) e confirmar com cada uma se ainda exige TISS anterior à 4.03.00 (RFC-PEP-006, Q1).
- Fechar os 3 ensaios do roteiro no ambiente de demo e gravar o vídeo narrado (§12.8): é a única parte pendente do Marco D.
