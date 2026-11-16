---
projeto: PEP
tipo: prd
status: em-refinamento
tags:
  - tissia
  - pep
  - prd
  - produto
origem: tissia/docs/prd/PRD-003-backoffice.md
---

# PRD-003 — Backoffice interno (Tissia Backoffice)

**Versão:** 1.0
**Data:** 08/11/2026
**Autor:** Pedro Santana
**Status:** Em refinamento
**UI:** `apps/backoffice` ([[ADR-PEP-0020-backoffice-app-separado|ADR-PEP-0020]]) · **Design:** [canvas do backoffice](https://claude.ai/code/artifact/8a0b5083-bb44-4f07-966a-645b42e4546a) (telas 01–18)

Console da **equipe interna** para operar o PEP como SaaS e, numa fase posterior, a Spike. Não é usado por clínicas nem por integradores.

## 1. Precisamos de um backoffice?

**Sim, mas só uma fatia agora.** A avaliação separa o design em três blocos com justificativas diferentes.

### 1.1 Clientes do PEP (telas 01–04, 18): necessário antes do piloto em produção

Hoje não existe caminho para criar um tenant, uma clínica, uma unidade ou o primeiro administrador fora do `seed:dev`. Sem backoffice, onboarding em `prod` significaria SQL manual com o papel `app_admin` (`BYPASSRLS`):

| Sem backoffice | Problema |
| :--- | :--- |
| Criar tenant/clínica/unidade por SQL com `app_admin` | Contorna o RLS em produção, sem trilha de auditoria e sem revisão. Contraria a regra "nada de query que contorne o tenant" |
| Suporte "olha no banco" para investigar problema de uma clínica | Acesso a dado clínico sem consentimento nem registro: risco LGPD direto |
| Mudar modo da IA, módulos ou especialidades por script | Sem histórico de quem mudou o quê, fora do alcance de quem faz Customer Success |
| Saber se a clínica usa o produto | Consulta ad hoc no banco de cada tenant |

O custo é baixo, porque o monorepo já tem identidade, RLS, auditoria, contratos Zod e a UI desenhada. O risco evitado é alto: acesso privilegiado não auditado em produção.

### 1.2 Planos e cobrança (tela 05): útil depois

Até haver clínicas pagantes fora do piloto, uma planilha e a nota fiscal resolvem. Entra quando a primeira clínica pagante assinar ([[RFC-PEP-019-backoffice-planos-e-cobranca|RFC-PEP-019]], Rascunho).

### 1.3 Operação da Spike (telas 06–17): fica para depois, e com outro desenho

As telas de integradores, observabilidade, análises, incidentes, modelos, prompts, avaliações, bases, regras, DUT, feedback e risco operam a **Spike**, que é outro produto e outro repositório. Três restrições já decididas mudam o desenho:

1. **Neutralidade** ([[PRD-002-spike-standalone|PRD-002 §4]]): o PEP não acessa o banco da Spike nem tem endpoint privado. O backoffice só pode falar com a Spike por uma **API de operação** própria da Spike ([[SPK-06-api-de-operacao|SPK-06]]), autenticada pela equipe e não pelo PEP.
2. **Conteúdo editorial em git**: regras são YAML revisado em PR (Spike ADR-0009), bases são versionadas com `MANIFEST.yaml` (Spike ADR-0006), prompts ficam em `app/prompts/` e o portão de avaliação roda no harness (Spike RFC-005). Um editor no backoffice criaria uma segunda fonte de verdade. As telas 11, 13, 14 e 15 passam a **mostrar estado, diff e aprovações**; a edição continua em PR.
3. **Escala**: o release plan da Spike diz "um painel só quando houver mais de ~10 clínicas". Observabilidade e incidentes (telas 07 e 09) já existem no Cloud Monitoring/Logging. O backoffice mostra um resumo e links até isso doer.

Gatilho para começar ([[RFC-PEP-020-backoffice-operacao-da-spike|RFC-PEP-020]], Rascunho): SPK-06 aceita **e** (mais de 10 clínicas na Spike **ou** primeiro integrador externo em produção).

## 2. Usuários (papéis internos)

| Papel | Faz | Nunca |
| :--- | :--- | :--- |
| Suporte / CS | Onboarding, acompanhamento de uso e saúde, pedido de acesso de suporte | Ver prontuário ou texto clínico pelo backoffice |
| Financeiro | Planos, assinaturas, faturas (B1) | Mexer em configuração clínica |
| Engenharia de IA | Ler métricas e análises da Spike, pedir mudança de roteamento (B2) | Publicar sem segunda aprovação |
| Curadoria | Acompanhar regras e bases (B2) | Aprovar sozinho |
| Direção médica | Segunda aprovação de critérios DUT (B2) | — |
| Segurança | Administra papéis internos e lê a trilha interna | — |

Matriz completa na tela 18 do design. **Nenhum papel enxerga prontuário, texto clínico ou PII de paciente.**

## 3. Requisitos (família `BKO`)

### B0 — antes do piloto em produção (P1)

- **RF-BKO-01** Acesso só para a equipe interna: realm próprio, chave FIDO2 obrigatória para todos os papéis, escolha de ambiente; em Produção, motivo e ticket são obrigatórios e a sessão expira em 60 min (tela 01).
- **RF-BKO-02** Papéis internos fechados (§2) com autorização no backend; a UI só esconde.
- **RF-BKO-03** Trilha de auditoria interna append-only e encadeada por hash. Toda leitura de dado de cliente e toda alteração ficam registradas, com motivo quando é Produção (tela 18).
- **RF-BKO-04** Carteira de clínicas: lista com plano (texto livre na B0), médicos ativos, modo da IA por unidade e uso (análises e aceite), com filtros e exportação em CSV (tela 03).
- **RF-BKO-05** Onboarding: criar tenant, clínica e unidades e convidar o primeiro administrador da clínica. O convite cria o usuário no Keycloak do PEP; daí em diante a clínica administra os próprios usuários (tela 04).
- **RF-BKO-06** Suspender e reativar clínica. Suspensão bloqueia **novos usuários e logins administrativos**, nunca atendimento, prontuário ou IA (mesma regra da tela 05).
- **RF-BKO-07** Acesso de suporte consentido: o backoffice **pede** acesso com escopo, duração e motivo; o administrador da clínica aprova no PEP. O acesso acontece **no PEP**, por um vínculo temporário e auditado nas duas trilhas; o backoffice nunca exibe dado clínico (tela 04; [[ADR-PEP-0022-backoffice-sem-dado-clinico|ADR-PEP-0022]]).
- **RF-BKO-08** Detalhe da clínica com a configuração vigente (modo da IA por unidade, módulos, especialidades) **somente leitura**. Quem altera é o gestor da clínica (tela 15 do PEP, [[ADR-PEP-0006-modos-silencioso-assistido|ADR-PEP-0006]]) ou o suporte por acesso consentido com escopo "Configurações".
- **RF-BKO-09** Visão geral mínima: clínicas ativas, médicos ativos, unidades por modo da IA e itens que pedem ação (acessos de suporte pendentes, convites expirados). Os números de saúde da Spike entram na B2 (tela 02).

### B1 — primeira clínica pagante (P2)

- **RF-BKO-10** Planos versionados (preço por médico, teste gratuito, módulos incluídos); mudança de preço vale só para novas assinaturas.
- **RF-BKO-11** Assinatura por tenant, faturas, situação de cobrança e régua de cobrança. A inadimplência nunca bloqueia prontuário nem IA.

### B2 — operação da Spike (gatilho em §1.3)

- **RF-BKO-12** Integradores e chaves: consulta, rotação e revogação; revogar chave de Produção exige segunda aprovação. Depende de SPK-05/06.
- **RF-BKO-13** Saúde da Spike: latência por etapa, taxa de degradação e erros por código, como resumo do Cloud Monitoring.
- **RF-BKO-14** Explorador por `analise_id`: só metadados, refs opacas e tempos por etapa.
- **RF-BKO-15** Estado de modelos, prompts, avaliações, bases, regras e DUT: versões em produção, diff, resultado do portão e aprovações, com link para o PR. Duas aprovações para DUT e roteamento.
- **RF-BKO-16** Feedback e desfechos agregados; fila dos alertas mais rejeitados; calibração do modelo de risco.

## 4. Não funcionais

| ID | Requisito |
| :--- | :--- |
| RNF-BKO-01 | Ingresso separado do PEP: o backoffice não é alcançável pelos domínios das clínicas ([[ADR-PEP-0021-backoffice-backend\|ADR-PEP-0021]]) |
| RNF-BKO-02 | Zero dado clínico ou PII de paciente no backoffice, garantido por permissão de banco e não só por código ([[ADR-PEP-0022-backoffice-sem-dado-clinico\|ADR-PEP-0022]]) |
| RNF-BKO-03 | Falha do backoffice nunca afeta o PEP nem a Spike (processos e ingressos separados) |
| RNF-BKO-04 | Disponibilidade sem SLA formal; meta de 99% em horário comercial |
| RNF-BKO-05 | Logs sem corpo de requisição (regra do monorepo) |

## 5. Fora de escopo

- Portal self-service do integrador (é a SPK-05, voltada para fora).
- Editor de regras, prompts, DUT ou bases no backoffice (continua em PR no `tissia-ai`).
- Painéis próprios de observabilidade ou de incidentes (Cloud Monitoring e status page).
- Leitura de prontuário, guia com PII ou texto clínico, por qualquer papel.
- Emissão de nota fiscal e gateway de pagamento na B1 (integração avaliada na RFC-PEP-019).

## 6. Ajustes ao design

| Tela | Design | Ajuste |
| :--- | :--- | :--- |
| 04 | Botões silencioso/assistido editáveis por unidade | Somente leitura; a alteração passa pela clínica ou pelo acesso consentido (RF-BKO-08) |
| 08 | "A Spike não retém justificativa gerada" | Incorreto: a Spike guarda a resposta completa enquanto durar o contrato (Spike ADR-0007). O que não fica retido é o texto de entrada, salvo opt-in. Corrigir o texto |
| 11, 14, 15 | Edição e publicação de prompt, regra e critério na UI | Somente leitura com diff, aprovações e link para o PR (§1.3) |
| Barra lateral | "PRODUÇÃO · sa-east-1" | A região é GCP `southamerica-east1` ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]]) |
| 18 | "Resolução CFM nº 2.454/2026" | **[validar]** número e ano na fonte oficial |
| 01 | "SSO corporativo" | Hoje não há IdP corporativo definido; B0 usa o Keycloak com realm da equipe e federação opcional **[validar]** |

## 7. Métricas

| Métrica | Meta B0 |
| :--- | :--- |
| Onboarding de clínica (tenant → 1º admin logado) | ≤ 30 min, sem SQL manual |
| Acessos ao banco de `prod` com `app_admin` fora de migração | 0 por mês |
| Acessos de suporte sem aprovação da clínica | 0 |
| Eventos internos sem registro na trilha | 0 (teste automatizado) |

## 8. Perguntas abertas

1. Quem decide a passagem de silencioso para assistido durante o piloto: o gestor da clínica ou o CS, com consentimento? A proposta deste PRD é o CS via acesso consentido, para manter uma fonte de verdade.
2. A empresa terá IdP corporativo (Google Workspace?) para federar o realm da equipe?
3. Os planos da tela 05 (Pro, Clínica Blindada, Módulo IA) valem para o PEP ou são os da Spike (PRD-002 §5)? O preço do PEP ainda não está no PRD-001.
