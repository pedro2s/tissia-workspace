---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0002-backend-nestjs-monolito-modular.md
---

# ADR-PEP-0002 — Backend NestJS, monólito modular

**Status:** Aceita — 07/11/2026

## Contexto

O PEP cobre domínios com regras próprias (clínico, agenda, faturamento TISS, financeiro, plataforma) e precisa ir ao piloto em 90 dias com uma equipe pequena trabalhando com agentes. Microsserviços desde o início trariam deploy, rede e consistência distribuída sem ganho mensurável. O design de produto já define NestJS e "um módulo por domínio".

## Decisão

- **NestJS 12 + TypeScript** (`apps/api` no monorepo, ver [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]]), um processo de API e um processo de workers (mesma imagem, entrypoints diferentes), com **Drizzle ORM** sobre contexto transacional RLS ([[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]]).
- **Um módulo por domínio:** `identidade`, `organizacao`, `pacientes`, `agenda`, `prontuario`, `documentos`, `modelos`, `faturamento`, `spike`, `retorno`, `contratos`, `financeiro`, `assinatura`, `notificacoes`, `auditoria`, `integracoes`, `bi`.
- **Fronteiras:** cada módulo expõe um serviço público (`<modulo>/public.ts`) e eventos de domínio ([[ADR-PEP-0008-eventos-outbox-bullmq|ADR-PEP-0008]]). Nenhum módulo importa entidade, repositório ou tabela de outro.
- **Lint de dependência** na CI (`dependency-cruiser` ou `eslint-plugin-boundaries`) falha o PR que cruzar fronteira.
- **Banco único** (PostgreSQL, [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]]), com schema por módulo; joins entre módulos só via leitura de projeções/visões publicadas.
- **Extrair serviço** só com motivo medido (escala, isolamento de falha, time dedicado), registrado em nova ADR. Candidatos conhecidos: geração de lote TISS e `bi`.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Microsserviços por domínio | Custo operacional e consistência eventual em tudo antes de haver carga |
| Python/FastAPI (igual à Spike) | Possível; o design escolheu NestJS e a UI é TS, o que permite compartilhar tipos gerados |
| Monólito sem fronteiras | Vira acoplamento total; a extração futura fica impraticável |

## Consequências

- Uma transação pode tocar dois módulos só através do serviço público do outro; o resto é evento.
- Testes por módulo com banco real ([[test-plan-pep|test plan]]); o lint de fronteira é gate de PR.
- Detalhes de estrutura de pastas e convenções na [[RFC-PEP-000-fundacoes|RFC-PEP-000]].
