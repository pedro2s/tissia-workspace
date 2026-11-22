---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - test-plan
  - qualidade
aliases:
  - test-plan
origem: tissia/docs/test-plan.md
---

# Test Plan — PEP Tissia

**Escopo:** P0, P1 e P2 (Ondas A–F) cobertos em detalhe; P3 em linhas gerais. A qualidade clínica da IA é testada na Spike ([[test-plan-ia|test plan da Spike]]); aqui testamos o PEP e a integração com ela.

> **Baseline verificada em 22/11/2026 (`tissia` `main` @ `e28182f`):**
> - `@pep/api`: **67 suítes / 564 testes** integrados em PostgreSQL 16 + Redis reais (`jest --runInBand`, migrações `0000`–`0063`), incluindo RLS forçado em todas as tabelas de domínio (`rls.spec.ts`), OpenAPI snapshots (`openapi.json`, `backoffice.json`, `tissia-ai-openapi_v1.json`), XSD TISS `4.02.00`/`4.03.00`, `hashTiss` ISO-8859-1, assinatura ICP-Brasil (`rfc011-assinatura-icp.spec.ts`), COFEN/LGPD (`onda-e2-f2-unit.spec.ts`), Receita Esquecida (`rfc009-receita-esquecida.spec.ts`), Repasse/NFS-e (`rfc010-f2-repasse-nfse.spec.ts`) e Backoffice (`backoffice-planos-ops.spec.ts`).
> - `@pep/contracts`, `@pep/ui` e `@pep/backoffice`: 100% verdes em Vitest, `pnpm typecheck`, `pnpm build` e `pnpm deps` (218 módulos, 0 violações de fronteira).

## 1. Estratégia

| Camada | O que garante | Onde roda | Ferramenta |
| :--- | :--- | :--- | :--- |
| Unitário | Regras de domínio (validações TISS locais, cálculo de repasse, pseudonimização, truncamento) e contratos/escores puros (`@pep/contracts`) | Todo PR | Jest (`@pep/api`), Vitest (`@pep/ui` e `@pep/contracts`) |
| Integração | Repositórios Drizzle, RLS, outbox, filas | Todo PR | Postgres e Redis reais (sem SQLite, sem mock de banco) |
| Contrato UI ↔ API | `@pep/ui` e `@pep/api` compilam contra os mesmos schemas Zod em `@pep/contracts` + snapshot OpenAPI versionado | Todo PR (`pnpm turbo run typecheck test`) | TypeScript + Zod + teste de snapshot OpenAPI |
| Contrato com a Spike | SpikeAdapter fala o `v1` exato | Todo PR + diário contra Spike `staging` | Snapshot do OpenAPI da Spike + testes consumer-driven |
| E2E | Fluxos ponta a ponta na UI | PR na `main` + noturno | Playwright contra `staging` (e MSW nos testes isolados de estados de UI) |
| Acessibilidade | WCAG 2.1 AA | PR que toca UI | axe-core no Playwright |
| Regressão visual | Telas fiéis ao design (incluindo zero regressão das telas `.sc04`…`.sc26` com Tailwind `@layer`) | PR que toca UI | Screenshots do Playwright comparados com a baseline |
| Não funcional | Carga, resiliência, offline, segurança, LGPD | Antes de cada release para `prod` | k6, toxiproxy, ZAP, revisão manual |
| Validação humana | Utilidade para médico e faturista | Piloto | Sessões observadas |

Sem framework de mock além do nativo; a Spike vira **stub HTTP** (servidor que responde fixtures gravadas da Spike real) nos testes de integração e E2E.

## 2. Integração e isolamento (RLS)

- Toda tabela com `tenant_id` tem teste que conecta como tenant A e prova que não lê, atualiza nem apaga linha do tenant B (incluindo `JOIN` e `COUNT`).
- Teste que reprova a migração se uma tabela nova com `tenant_id` não tiver policy RLS habilitada.
- Usuário da aplicação não é dono das tabelas (sem `BYPASSRLS`); teste confere.
- Outbox: dado gravado e evento gravado na mesma transação; rollback não deixa evento órfão; evento publicado uma vez (idempotência no consumidor).

## 3. Contrato com a Spike

| Caso | Esperado |
| :--- | :--- |
| Snapshot do `openapi.json` da Spike (`tests/snapshots/openapi_v1.json` no repo dela) copiado para o PEP | Diferença quebra o build e exige revisão do SpikeAdapter |
| Payload gerado pelo SpikeAdapter validado contra o schema | Nenhum campo fora do schema (a Spike responde 422 `extra_forbidden`) |
| **Zero PII** (teste de propriedade) | Para pacientes gerados com nome, CPF, CNS, carteirinha, telefone, e-mail: nenhum desses valores aparece serializado no payload; `paciente` só tem `ref`, `sexo`, `idade` |
| `paciente.ref` | Estável para o mesmo paciente e clínica; diferente entre clínicas; não reversível sem a tabela de vínculo |
| Evolução > 20.000 caracteres ou corpo > 64 KB | Truncamento prioriza o atendimento atual; nunca 413 |
| Mesmo `guia_ref` em todas as revisões | Confirmado nas chamadas sucessivas |
| Spike 401/403 | Alerta operacional; fluxo segue como análise parcial |
| Spike 422 | Erro logado (sem corpo), métrica, fluxo segue; teste falha no CI |
| Spike 5xx, timeout (> 6 s), conexão recusada | Análise parcial; atendimento fecha; guia segue para a mesa |
| `degradado: true` | UI mostra "análise parcial" (tela 14) |
| Feedback com `item_id` de cada tipo (`alerta.id`, `<tuss>:justificativa`, `guia:justificativa`, `sug_cid:`, `sug_tuss:`) | 204; decisão persistida no PEP antes da chamada |
| Modo silencioso | Análise persistida, nada exibido, nenhum feedback enviado |
| Contra Spike `staging` (diário) | 5 casos fixos (consulta, SP/SADT com DUT, sem evolução, guia vazia, campo 23 longo) respondem 200 e validam no schema |

## 4. E2E (Playwright)

Fluxos obrigatórios na P1:

1. Login com MFA (TOTP) → escolher unidade → shell.
2. Agendar paciente com carteirinha → check-in → abrir prontuário.
3. Evoluir SOAP com `/exame` → fechar atendimento → painel anti-glosa aparece com risco e alertas (Spike stub) → aceitar justificativa → feedback enviado.
4. Spike stub fora do ar → mesmo fluxo termina com "análise parcial".
5. Mesa do faturista → filtrar por operadora → revalidar → gerar lote XML → XML válido no XSD.
6. Adendo em evolução fechada exige justificativa e aparece na linha do tempo.
7. Revelar CPF mascarado exige justificativa e gera registro em auditoria (tela 25).
8. Tablet (viewport 834×1194) e celular (390×844): fluxos 3 e 5 com alvos ≥ 44 px.

## 5. Interface

- **Acessibilidade:** axe sem violação `serious`/`critical` em todas as rotas; navegação completa por teclado nos fluxos E2E; foco visível; `aria-label` em botões só de ícone.
- **Regressão visual:** baseline gerada das telas 01–26 do design; diferença acima do limiar exige aprovação no PR. A baseline é o [canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d) renderizado em 1440 px (as telas estáticas de `apps/ui/src/modulos/` reproduzem o canvas).
- **Estados:** cada tela com dados tem teste de vazio, carregando, erro e sucesso (tela 14 serve de referência).

## 6. Não funcionais

| Tipo | Como | Meta | Quando |
| :--- | :--- | :--- | :--- |
| Carga | k6 em `staging`: 50 usuários simultâneos por clínica × 5 clínicas; mix agenda, prontuário, mesa | p95 ≤ 1,5 s; 0 erro 5xx | Antes de cada release `prod` |
| Carga Spike | k6 no fechamento de atendimento a 5 req/s | p95 da análise ≤ 5 s (meta da Spike) | Antes do piloto |
| Resiliência | toxiproxy: Spike com latência de 10 s, 50% de erro, fora | Fluxo não bloqueia; análise parcial; nenhum job perdido | Fim da P1 |
| Offline | Playwright com rede cortada no tablet: evoluir, reconectar | Nenhuma perda; conflito vira adendo ([[ADR-PEP-0015-offline-prontuario\|ADR-PEP-0015]]) | Fim da P1 |
| XML TISS | Lotes gerados validados contra o XSD 4.03.00 oficial; nenhuma tag de CID no XML de SP/SADT e consulta; leitura de 4.03.00 (e das versões anteriores que o piloto exigir) com arquivos de exemplo | 100% válidos | Todo PR do módulo `faturamento` |
| Segurança | SAST (CodeQL), auditoria de dependências (`npm audit`), ZAP baseline em `staging`, checklist OWASP ASVS L2 | Sem achado alto | Todo PR (SAST) / release (resto) |
| Pentest | Externo | Sem achado alto aberto | Antes de dado real |
| Backup | Restore PITR em projeto separado | Restore < 1 h **[validar]** RTO | Antes do piloto e trimestral |

## 7. LGPD e auditoria

- Teste de log: requisições com PII sintética (nome, CPF, evolução marcada) → nenhuma linha de log contém os marcadores.
- Toda abertura de prontuário gera registro "quem viu" com usuário, unidade, data e IP; teste por rota que lê prontuário.
- Decisões sobre sugestões de IA gravadas com usuário e papel antes do feedback à Spike.
- Exportação de dados do titular devolve tudo do paciente e nada de outro.
- Revisão manual de amostra de logs de `staging` no fim da P0.

## 8. Casos obrigatórios por RFC

| RFC | Casos |
| :--- | :--- |
| PEP-000 | RLS (§2); outbox; `/healthz`; migração sem policy RLS falha |
| PEP-001 | MFA obrigatório para profissional; bloqueio após N tentativas; recuperação; troca de unidade muda escopo; perfil faturista não lê evolução fora da guia |
| PEP-002 | Mascaramento por padrão; revelar com justificativa e auditoria; várias carteirinhas, uma principal; elegibilidade registra origem e hora |
| PEP-003 | Conflito de sala/profissional impedido; encaixe; autorização pendente visível; lembrete WhatsApp idempotente |
| PEP-004 | Versão a cada gravação; edição pós-fechamento só por adendo; comandos `/`; autoria humano × IA; `AtendimentoFechado` cria guia rascunho |
| PEP-005 | Tabela §3 inteira; modos desligado/silencioso/assistido |
| PEP-006 | Validações TISS locais sem a Spike; Campo 23 > 500 bloqueado; revisões com mesmo `guia_ref`; lote válido no XSD; numeração de lote por unidade sem buraco nem repetição |
| PEP-014 | §7 inteira; flags por clínica só por admin |
| PEP-015 | §5 inteira; fluxos E2E §4 |
| PEP-017 | Token do realm do PEP rejeitado no backoffice e vice-versa; processo `api` sem rotas `/bo/*`; sessão de Produção sem motivo → 422; toda requisição gera `auditoria_interna` e falha da trilha → 503; `db:rls` falha se `app_plataforma` tiver grant em tabela clínica |
| PEP-018 | E2E de onboarding (convite por e-mail → 1º login com 2º fator); idempotência; unidade nasce `desligado`; acesso de suporte pendente → aprovado → expirado, sem nenhuma rota de prontuário acessível; suspensão não bloqueia atendimento |
| PEP-007..013 (P2) | Definidos ao detalhar cada RFC; no mínimo: conciliação de demonstrativo com divergência, idempotência por `lote_ref`, repasse sobre valor recebido, assinatura só com todas as sugestões decididas, áudio não retido |

Cobertura: sem meta numérica. Todo ramo de validação TISS local, pseudonimização, truncamento, cálculo de repasse e RLS tem teste.

## 9. Validação humana (piloto)

- **Médico:** 20 atendimentos observados no modo assistido. Pergunta-chave: "o painel mudou alguma decisão sua ou só atrapalhou?" Tempo de revisão medido.
- **Faturista:** 50 guias revisadas na mesa. Pergunta-chave: "precisou mexer na guia que chegou do médico?"
- **CMO:** revisa uma amostra de justificativas aceitas sem edição. Achado de fato não suportado vira caso no golden set da Spike.

Registro em `docs/piloto/AAAA-MM-DD-<clinica>.md`.

## 10. Critérios de saída por fase

| Fase | Sai quando |
| :--- | :--- |
| D | [[release-plan-pep#128-gate-de-saída\|Release plan §12.8]]: roteiro da demo como E2E verde, invariantes da Spike testados, telas não implementadas marcadas como prévia |
| P0 | Critérios de aceite das RFC-PEP-000/001/015; §2 e §7 verdes; revisão de logs sem PII |
| P1 | Critérios das RFC-PEP-002..006 e 014; backoffice B0 (RFC-PEP-017/018) antes da 1ª clínica em `prod`; E2E §4 verdes; carga, resiliência, offline e XML §6 ok; pentest sem achado alto; pré-requisitos do [[release-plan-pep#4-pré-requisitos-para-dado-real\|release plan §4]] |
| P2 | Metas do PRD-001 §4 no piloto (aceite ≥ 70%, revisão p50 ≤ 15 s, redução de glosa medida); casos P2 da §8 |
| P3 | Definido na RFC-PEP-016 (inclui requisitos de certificação SBIS **[validar]**) |
