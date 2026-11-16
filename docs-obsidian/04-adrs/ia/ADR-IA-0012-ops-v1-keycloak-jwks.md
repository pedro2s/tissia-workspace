---
projeto: tissia-ia
tipo: adr
status: aceita
tags:
  - tissia
  - tissia-ia
  - adr
  - decisao-arquitetural
aliases:
  - 0012-ops-v1-keycloak-jwks
origem: tissia-ai/docs/adr/0012-ops-v1-keycloak-jwks.md
---

# ADR-0012 — API de operação interna `/ops/v1` com JWT do Keycloak da equipe (JWKS)

**Status:** Aceita — 15/11/2026 (decisão de Pedro; espelha a [[ADR-PEP-0025-decisoes-scribe-e-ops-tissia-ai|ADR-PEP-0025]] do PEP, §2)

## Contexto

O backoffice do PEP ([[SPK-06-api-de-operacao|SPK-06]]) precisa observar e operar a Tissia AI sem acesso ao banco nem script em produção. Até aqui a única autenticação era a API key de integrador (`tsk_…`), que não serve para a equipe interna: não tem papéis por pessoa, MFA nem revogação central. Estavam em aberto o provedor de identidade e se a troca de provedor de IA (`roteamento`) seria editável por API.

## Decisão

- **`/ops/v1` é uma API separada do `/v1`:** prefixo próprio, fora do `/v1/openapi.json` público, contrato em `/ops/openapi.json` (protegido) e montada só com `OPS_HABILITADO=true`.
- **Autenticação:** JWT emitido pelo Keycloak da equipe (`tissia-equipe-<amb>`), validado por JWKS (`OPS_OIDC_JWKS_URL`, cache de 5 min), com `iss` = `OPS_OIDC_ISSUER`, `aud` = `tissia-ai-operacao`, `exp` obrigatório, algoritmos `RS256`/`ES256`. Biblioteca `PyJWT[crypto]`.
- **Autorização:** papéis no claim `papeis` (`eng_ia`, `suporte`, `curadoria`, `dir_medica`, `seguranca`). Sem token, token inválido ou API key de integrador → `401`; sem o papel da rota → `403`.
- **Auditoria:** toda chamada autorizada grava `ops_auditoria` (ator `sub`, papéis, ação, alvo, `request_id`); escritas gravam também `antes`/`depois` e `aprovacao_ref`.
- **Escrita mínima na Fase 1:** só a revogação de chave de integrador. `PUT /ops/v1/roteamento` não existe: o roteamento de provedores continua em configuração de deploy revisada por PR. A promoção de prompt a canário fica para a Fase 2, quando existir o portão formal de aprovação de prompts.
- **Minimização ([[ADR-IA-0007-minimizacao-de-dados|ADR-0007]]):** nenhuma rota devolve texto clínico, justificativa, evidência, mensagem livre de operadora ou `paciente_ref`.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| IdP próprio na Tissia AI | Duplica gestão de identidade, MFA e revogação que a equipe já tem no Keycloak |
| API key de integrador com escopo `admin` | Não identifica a pessoa (auditoria fraca) e mistura equipe interna com clientes |
| `PUT /ops/v1/roteamento` | Troca de provedor por API sem revisão; a config de deploy já é revisada e reverte com o deploy |

## Consequências

- Nova dependência `PyJWT[crypto]` e nova tabela `ops_auditoria` (migração aditiva).
- O serviço passa a depender do JWKS do Keycloak só nas rotas `/ops/v1`; o `/v1` dos integradores não é afetado se o Keycloak cair.
- Pen test do `/ops/v1` antes de habilitar escrita em produção ([[SPK-06-api-de-operacao|SPK-06]]).

