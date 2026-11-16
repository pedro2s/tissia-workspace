---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0021-backoffice-backend.md
---

# ADR-PEP-0021 — Backend do backoffice: módulo no monólito, processo e ingresso próprios, realm da equipe

**Status:** Proposta — 08/11/2026
**Relacionadas:** [[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]], [[ADR-PEP-0010-identidade-keycloak|ADR-PEP-0010]], [[ADR-PEP-0020-backoffice-app-separado|ADR-PEP-0020]], [[PRD-003-backoffice|PRD-003]]

## Contexto

O `apps/backoffice` precisa de API. A B0 (PRD-003) escreve nas tabelas de plataforma do PEP (`tenants`, `clinicas`, `unidades`, convites, acessos de suporte) e lê agregados de uso. Os usuários são a equipe interna, com autenticação mais forte (FIDO2 obrigatório) e outro público. A equipe tem 1 desenvolvedor com agentes.

## Decisão

1. **Módulo `backoffice` dentro do `@pep/api`** (monólito modular, ADR-PEP-0002), e não um serviço novo. Ele fala com `organizacao` e `identidade` pelas interfaces públicas desses módulos, como qualquer outro módulo.
2. **Terceiro processo**, ao lado de `api` e `worker`: `src/backoffice.ts` sobe só os controllers do backoffice (prefixo `/bo/v1`) a partir da mesma imagem. Deploy em Cloud Run separado, com **ingresso restrito** (IAP ou balanceador interno com allowlist) e domínio próprio. O processo `api` **não registra** os controllers `/bo/*`, e o `backoffice` não registra os do PEP.
3. **Realm Keycloak da equipe** (`tissia-equipe-<ambiente>`), separado dos realms `pep-*`: usuários internos não existem no realm das clínicas e vice-versa. Cliente `bo-web` (PKCE S256) com audiência `pep-backoffice`. O guard do processo `backoffice` só aceita tokens desse realm, com `amr` contendo `hwk` ou `mfa` vindo de WebAuthn **[validar o claim emitido pelo Keycloak]**.
4. **Sessão de Produção:** `POST /bo/v1/sessao` recebe ambiente, motivo e ticket, grava na trilha interna e emite um JWT de 60 min com `papeis` e `motivo_id`. É o mesmo padrão da sessão do PEP (RFC-PEP-001), com outro segredo de assinatura.
5. **Contratos:** schemas em `@pep/contracts/src/backoffice/*` (Zod `.strict()`), usados pela UI e pela API. O OpenAPI do backoffice é um documento separado (`openapi/backoffice.json`) e não é público.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Serviço NestJS separado (`apps/backoffice-api`) | Escreveria nas tabelas do PEP de fora do módulo dono e duplicaria Db, outbox, config e CI. Reavaliar se o backoffice precisar escalar ou ser liberado em ritmo diferente |
| Rotas `/bo/*` no mesmo processo `api` | Mesma superfície de ataque e mesmo domínio das clínicas; um bug de roteamento exporia o backoffice |
| Usuários internos no realm do PEP com um "perfil interno" | Mistura públicos. Um erro de vínculo daria a um usuário de clínica um papel interno, ou o contrário |
| Ferramenta pronta (Retool, Appsmith) | Exigiria credencial de banco ou API genérica e tiraria a trilha interna e o FIDO2 do nosso controle; o design já está pronto em `apps/backoffice` |

## Consequências

- Um deploy a mais (mesma imagem, outro entrypoint) e um realm a mais por ambiente.
- Regra de fronteira nova no `dependency-cruiser`: só `src/backoffice.ts` e `modulos/backoffice` importam os controllers do backoffice.
- O e2e do backoffice usa o Keycloak real com WebAuthn virtual, como o do PEP.
