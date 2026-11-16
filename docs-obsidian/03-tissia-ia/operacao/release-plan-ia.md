---
projeto: tissia-ia
tipo: operacao
status: ativo
tags:
  - tissia
  - tissia-ia
  - operacao
  - release-plan
  - roadmap
aliases:
  - release-plan
  - release-plan-tissia-ia
origem: tissia-ai/docs/release-plan.md
---

# Release Plan — Tissia IA

## 1. Visão geral

| Fase | Duração indicativa | Entrega | Gate de saída |
| :--- | :--- | :--- | :--- |
| F0 — Fundações | 2 semanas | Infra, auth, bases, harness, golden set | [[test-plan-ia\|Test plan §6]] F0 + ADR-0004 decidida |
| F1 — Motor MVP | 3–4 semanas | `POST /v1/guias/analisar` completo em `prod` (sem cliente real) | Test plan §6 F1 + guia de integração publicado |
| F2 — Piloto | 6–8 semanas | 3–5 clínicas, sombra → visível, relatório | Pré-requisitos legais (§3) + métricas de piloto |
| F3 — Preditivo | a definir | Score de risco, lotes, contratos | Definido na RFC-008 |
| F4 — Scribe | a definir | Estruturação pós-consulta, depois streaming | Definido na RFC-009 |
| F5 — Extensão | a definir | Extensão standalone na Chrome Web Store | Definido na RFC-010 |

Durações assumem 1 dev (Pedro) trabalhando com agentes. Cada gate é decidido em conjunto por Pedro, Thiago e o CMO. Não há data fixa: só se avança quando o gate passa.

## 2. Ambientes e promoção

| Ambiente | Para quê | Deploy |
| :--- | :--- | :--- |
| local | Desenvolvimento | `uv run fastapi dev`, Postgres em Docker |
| `staging` | Integração com o PEP do Thiago (ambiente de homologação dele), evals, carga | Automático a cada merge em `main` |
| `prod` | Piloto e clientes | Tag `vX.Y.Z` + aprovação manual |

Versionamento: SemVer da aplicação. `versao_bases` é independente e aparece em toda resposta. Mudança só de base (TUSS nova, DUT nova) também sai por tag.

## 3. Pré-requisitos para receber dado real (antes da F2)

- [ ] Sócios do Thiago aprovaram a parceria; contrato com revenue share e responsabilidades assinado.
- [ ] Papéis LGPD definidos (Q6); DPA integrador ↔ Tissia IA; termo de adesão das clínicas-piloto (incluindo o opt-in `guardar_entrada_para_avaliacao`).
- [ ] RIPD (Relatório de Impacto à Proteção de Dados) concluído.
- [ ] Gates da ADR-0004 resolvidos para o decisor usado em `prod` (se for Jev).
- [ ] Revisão de segurança do test plan §4 feita.

## 4. Feature flags (por clínica, `clinicas.flags`)

| Flag | Default | Efeito |
| :--- | :--- | :--- |
| `modo` | `visivel` | `sombra`: analisa e grava, mas a resposta volta com `alertas: []`, `sugestoes_codigos: null`, `justificativa_sugerida: null`, e o PEP não mostra nada. `visivel`: resposta completa (default quando a flag está ausente; no onboarding de piloto recomenda-se configurar explicitamente `modo=sombra`) |
| `gerar_justificativa` | `true` | Desliga a justificativa mesmo que o request peça |
| `sugerir_codigos` | `true` | Desliga a sugestão TUSS/CID mesmo que o request peça |
| `decisor` | (global) | Override `jev` \| `gemini` por clínica |
| `guardar_entrada_para_avaliacao` | `false` | Opt-in do piloto (ADR-0007) |

`# ponytail:` flags em `jsonb`, alteradas por script. Um painel só quando houver mais de ~10 clínicas.

## 5. Rollout do piloto (F2)

1. **Integração em homologação:** o PEP do Thiago chama `staging` com dados fictícios. Contrato e UX validados com ele.
2. **Sombra (2–3 semanas):** 3–5 clínicas em `prod` com `modo=sombra`. Medimos o que **teríamos** alertado contra os desfechos reais (endpoint de desfecho). Gate: precisão estimada ≥ 70% e nenhum alerta clinicamente errado na revisão do CMO.
3. **Visível — 1 clínica (1–2 semanas):** acompanhamento próximo com a faturista; o feedback é revisado semanalmente.
4. **Visível — todas as clínicas-piloto:** até o fim da F2, com relatório final (RFC-007).
5. **Decisão comercial:** com o relatório, Thiago e os sócios definem preço e lançamento na base (upsell R$ 397–697/médico/mês).

## 6. Rollback

| Situação | Ação | Tempo |
| :--- | :--- | :--- |
| Bug de código em `prod` | Cloud Run: tráfego 100% para a revisão anterior | < 5 min |
| Base/regra/DUT errada | Revert do PR + nova tag (ou rollback de revisão, que reverte junto) | < 30 min |
| Modelo de IA degradado (qualidade ou latência) | Trocar `decisor`/`GEMINI_MODEL` por config, ou desligar `gerar_justificativa`/`sugerir_codigos` globalmente | < 5 min |
| Alerta clinicamente errado reportado | Clínica volta para `modo=sombra`; caso vira teste no golden set antes de reativar | < 5 min |
| Migração de banco | Migrações só aditivas na F1/F2 (nada de drop/rename de coluna em uso); rollback de app não exige rollback de schema | — |

## 7. Monitoramento e alertas

- Uptime check em `/healthz` (Cloud Monitoring) → e-mail/Slack.
- Métricas a partir dos logs: taxa de `degradado`, p95 por rota, erros 5xx, custo estimado de IA/dia.
- Alerta: `degradado` > 10% em 15 min; p95 > 6 s em 15 min; qualquer 5xx em sequência > 1%.
- Relatório semanal no piloto: alertas por tipo, aceitação, top regras rejeitadas.

## 8. Comunicação

| Evento | Quem é avisado | Como |
| :--- | :--- | :--- |
| Fim de cada fase (gate) | Pedro, Thiago, CMO | Reunião + doc de resultado em `docs/evals/` ou relatório do piloto |
| Mudança de contrato da API | Time do Thiago | Changelog em `docs/integracao.md` + aviso com 2 semanas de antecedência (incompatível = `/v2`) |
| Atualização de base (TUSS/DUT) | Thiago + consultor de faturamento | Nota do release com o diff resumido |
| Incidente com dado ou alerta errado | Thiago + clínica afetada | Mesmo dia; post-mortem curto em `docs/incidentes/` |
