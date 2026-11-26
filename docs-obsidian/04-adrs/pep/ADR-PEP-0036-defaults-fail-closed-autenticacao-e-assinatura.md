---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - seguranca
  - autenticacao
  - assinatura
aliases:
  - ADR-PEP-0036
  - Defaults fail-closed de autenticação e assinatura
---

# ADR-PEP-0036 — Defaults fail-closed de autenticação e assinatura

**Status:** Aceita — 26/11/2026
**Relacionadas:** [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]

## Contexto

A auditoria clínica de 26/11/2026 encontrou três defaults inseguros em `apps/api/src/comum/config.ts`: `AUTH_MODE=dev` (o cabeçalho `x-dev-perfil` define o perfil, com padrão `admin`), `ASSINATURA_PROVEDOR=local` e `ASSINATURA_LOCAL_PIN=1234`. A proteção existia só como `refine` de `NODE_ENV=production`, então um staging ou homologação com dado real, rodando com outro `NODE_ENV`, ficava aberto e emitia "assinatura ICP" simulada com PIN conhecido.

## Decisão

1. `AUTH_MODE` tem padrão `oidc`. O modo `dev` só existe com `AUTH_MODE=dev` explícito e continua proibido em produção.
2. `ASSINATURA_PROVEDOR` tem padrão `bloqueado`. `local` e `psc` exigem configuração explícita.
3. `ASSINATURA_LOCAL_PIN` não tem valor padrão e é obrigatório apenas quando `ASSINATURA_PROVEDOR=local`.

## Consequências

- Quem sobe a API sem `.env` precisa declarar `AUTH_MODE=dev` e, para assinar, `ASSINATURA_PROVEDOR=local` com PIN. `apps/api/.env.example` já traz esses valores.
- Compose e CI já fixam `AUTH_MODE=oidc` e `ASSINATURA_PROVEDOR=psc`; produção não muda.
- Fica pendente fechar o caso de homologação com dado real (`NODE_ENV!=production`) também para a exigência de ICP em atestado e pedido de exame.
