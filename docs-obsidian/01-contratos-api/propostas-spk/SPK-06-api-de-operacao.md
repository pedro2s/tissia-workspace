---
projeto: ecossistema
tipo: spk
status: ativo
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-06-api-de-operacao.md
---

# SPK-06 — API de operação interna (`/ops/v1`)

**Status:** Fase 1 implementada na `tissia-ia` ([[RFC-IA-014-api-de-operacao-interna|RFC-IA-014]], [[ADR-IA-0012-ops-v1-keycloak-jwks|ADR-IA-0012]]: `app/api/ops.py`, `ops_observabilidade.py`, `ops_estado.py` com JWT Keycloak JWKS e revogação de chave; `GET /ops/v1/avaliacoes` vazio em prod pendente); Fase 2 (`canario` e afins) condicionada a portão formal e `PUT /ops/v1/roteamento` descartado nesta etapa; falta integrar o cliente do Backoffice B2 no `PEP` ([[RFC-PEP-020-backoffice-operacao-da-spike|RFC-PEP-020]]) · **Necessária para:** backoffice B2 ([[RFC-PEP-020-backoffice-operacao-da-spike|RFC-PEP-020]], [[ADR-PEP-0023-spike-no-backoffice|ADR-PEP-0023]])

## Objetivo

Dar à equipe interna uma API para observar e operar a Spike sem acesso ao banco, sem script em produção e sem misturar com a `/v1` dos integradores.

## Motivação

- Hoje integradores e clínicas são criados por script (`criar_integrador.py`, `configurar_clinica.py`), e o relatório do piloto também (`relatorio_piloto.py`). Operar em produção = shell com credencial de banco.
- O backoffice (PRD-003) não pode ler o banco da Spike (neutralidade, PRD-002 §4).
- Regras, bases e prompts já são versionados em git (Spike ADR-0006/0009). A operação precisa **ver** o estado em produção e acionar mudanças reversíveis, não editar conteúdo.

## Contrato proposto

- Prefixo `/ops/v1`, documentado num OpenAPI **separado** do `/v1` público e não publicado para integradores.
- Autenticação: JWT do realm da equipe (`tissia-equipe-<amb>`), audiência `spike-operacao`, validado por JWKS. Papéis no claim `papeis`. API key de integrador → 401.
- Toda rota grava `ops_auditoria` (ator `sub`, ação, alvo, antes/depois).
- Somente leitura, exceto onde indicado.

| Rota | Retorno / efeito | Papéis |
| :--- | :--- | :--- |
| `GET /ops/v1/saude?janela=1h` | p50/p95 por etapa (regras, DUT, justificativa, sugestão), taxa de degradação, contagem por status HTTP | todos |
| `GET /ops/v1/analises?clinica_ref=&degradada=&de=&ate=&cursor=` | Lista de metadados: `analise_id`, hora, tipo, risco, `degradado`, latência, nº de itens de feedback, desfecho | eng_ia, suporte |
| `GET /ops/v1/analises/{id}` | Metadados, tempos por etapa, alertas (`id`, `tipo`, `severidade`, `fonte`), feedback (`item_id`, `acao`) e desfecho. **Sem** texto de justificativa, evidência ou mensagem livre | eng_ia, suporte |
| `GET /ops/v1/integradores`, `GET /ops/v1/integradores/{id}/uso?janela=24h` | Integradores, clínicas, uso por rota e erros por código (para 422: só o caminho `loc` e a contagem) | suporte, eng_ia |
| `POST /ops/v1/integradores/{id}/chaves/{chave}/revogacao` | Revoga (escrita; o backoffice exige 2ª aprovação em prod) | seguranca |
| `GET /ops/v1/versoes` | API, `versao_bases`, prompts e modelo de risco em produção | todos |
| `GET /ops/v1/roteamento` · `PUT /ops/v1/roteamento` | Percentual por provedor e cadeia de fallback (escrita com `aprovacao_ref`) | eng_ia |
| `GET /ops/v1/prompts`, `GET /ops/v1/prompts/{familia}/diff?de=&para=` | Versões em git, estado por ambiente, resultado do portão | eng_ia, curadoria |
| `POST /ops/v1/prompts/{familia}/canario` | Promove a canário uma versão **aprovada no portão** (escrita) | eng_ia |
| `GET /ops/v1/avaliacoes` | Resultados do harness (Spike RFC-005) | eng_ia, curadoria |
| `GET /ops/v1/bases`, `GET /ops/v1/bases/{versao}/diff` | `MANIFEST`, diff resumido entre versões | curadoria, dir_medica |
| `GET /ops/v1/regras?fonte=` | Regras carregadas, casos de teste, impacto estimado | curadoria |
| `GET /ops/v1/feedback/agregado?de=&ate=` | Aceite, edição e rejeição por tipo de alerta e regra | eng_ia, curadoria |
| `GET /ops/v1/risco/calibracao` | Curva de calibração e versões do modelo | eng_ia |

## Impacto na Spike

- Módulo `app/api/ops.py` com dependência de autenticação JWT (nova; hoje só existe API key).
- `ops_auditoria` (aditiva).
- Métricas por etapa: o log já tem `latencia_ms` total. É preciso registrar tempos por etapa na análise (`analises.tempos jsonb`) **[validar se já existe]**.
- O conteúdo editorial é lido do próprio deploy (arquivos em `data/`, `app/prompts/`), sem banco novo.

## LGPD e segurança

- Nenhuma rota devolve texto clínico, evidência literal, justificativa gerada nem `paciente_ref`. Erros 422 expõem só o caminho do campo e a contagem.
- Escritas são reversíveis e auditadas; a 2ª aprovação é controlada pelo backoffice e referenciada em `aprovacao_ref`.
- Pen test do `/ops/v1` antes de habilitar escrita em produção.

## Critérios de aceite

- [ ] API key de integrador em `/ops/v1/*` → 401; JWT sem o papel → 403.
- [ ] `GET /ops/v1/analises/{id}` não contém nenhum campo de texto livre gerado (teste de schema).
- [ ] Promover a canário uma versão reprovada no portão → 409.
- [ ] Toda escrita gera linha em `ops_auditoria`.

## Perguntas abertas e decisões encaminhadas (15/11/2026)

1. **A `tissia-ai` aceita depender do Keycloak da equipe (JWKS) ou prefere um IdP próprio?**
   - **Decisão encaminhada:** usar o **Keycloak da equipe** (`tissia-equipe-<amb>`) validado via JWKS com audiência dedicada (`aud: tissia-ai-operacao`), rejeitando API keys de integrador com `401` e exigindo claims de papel (`papeis`) com `403` quando ausentes. Evita duplicação de gestão de identidade, MFA e revogação.
2. **`PUT /ops/v1/roteamento` deve existir ou o roteamento continua em config de deploy?**
   - **Decisão encaminhada:** **não implementar `PUT /ops/v1/roteamento` na Fase 1**. Manter o roteamento de provedores em configuração de deploy revisada por PR/GitOps, expondo apenas `GET /ops/v1/roteamento` (somente leitura) para visibilidade no backoffice.
3. **Fatiamento para aceitação na `tissia-ai`:**
   - **Fase 1 (Observabilidade + Revogação de emergência):** autenticação JWKS, `ops_auditoria`, rotas `GET` (`/saude`, `/analises`, `/analises/{id}`, `/integradores`, `/integradores/{id}/uso`, `/versoes`, `/roteamento`, `/prompts`, `/avaliacoes`, `/bases`, `/regras`, `/feedback/agregado`) e `POST /ops/v1/integradores/{id}/chaves/{chave}/revogacao`.
   - **Fase 2 (Escrita condicionada a portão):** `PUT /ops/v1/roteamento` e `POST /ops/v1/prompts/{familia}/canario` ficam bloqueados até existir o portão formal de aprovação de prompts e dupla aprovação validada ponta a ponta.
