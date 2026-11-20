---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
  - agenda
  - enfermagem
  - financeiro
aliases:
  - RFC-PEP-022
  - Triagem e check-out na recepção
---

# RFC-PEP-022 — Triagem (pré-consulta), Check-out da Recepção e fronteira Médico × Faturista

**Fase:** P2 (evolução operacional pós-P1) · **Status:** Rascunho · **ADRs:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0014-convenio-e-particular-no-mvp|ADR-PEP-0014]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0031-separacao-papeis-checkout-recepcao-e-triagem|ADR-PEP-0031]]
**Telas:** 03 (Agenda — fila de triagem e painel de check-out), 05 (Prontuário — faixa de sinais vitais da triagem), 17 (Editor de guia — restrição ao faturista)
**Requisitos:** RF-AGE-10..12, RF-PRO-11, RF-FIN-07 (propostos ao PRD-001)
**Depende de:** [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-003-agenda|RFC-PEP-003]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-010-financeiro-e-repasse|RFC-PEP-010]] (Fatia 1)

## Objetivo

Alinhar o fluxo diário do PEP à dinâmica real de uma clínica multiprofissional:
1. Permitir que a **Enfermagem** registre triagem/pré-consulta (sinais vitais, antropometria, alergias e medicações em uso) antes de o médico abrir a evolução.
2. Centralizar o **Check-out** (cobrança particular, recibo PDF, entrega de documentos e agendamento de retorno) na bancada da **Recepção**, retirando a cobrança do consultório médico.
3. Ajustar o RBAC padrão para que o perfil `medico` não edite campos administrativos de guias TISS (`guias:escrever`), mantendo total visibilidade clínica (`guias:ler`).

## Escopo

**Dentro**
- **Fatia 1 — Fronteira RBAC Médico × Faturista e Check-out na Recepção:**
  - Atualizar `PERMISSOES_POR_PERFIL` em `packages/contracts/src/identidade.ts`: remover `guias:escrever` do perfil `medico` (preservar `guias:ler`); adicionar permissão específica `triagem:escrever` para `enfermagem` e `medico`.
  - Criar painel/gaveta de **Check-out** acessível a partir de `Agenda.tsx` para agendamentos em estado `atendido` (exibindo status financeiro particular via `RecebimentoParticular`, documentos prontos/assinados para impressão sem exibir conteúdo clínico sigiloso e atalho para agendar retorno).
  - Simplificar `Atendimento.tsx` (quando fechado e particular): substituir o bloco interativo de recebimento por indicador de encaminhamento à recepção (mantendo atalho apenas se o usuário logado tiver `financeiro:receber_particular`).
- **Fatia 2 — Triagem / Pré-consulta (`triagens`):**
  - Tabela `triagens` no módulo `prontuario` (com `tenant_id`, RLS forçada, `agendamento_id` único, vínculo opcional com `atendimento_id`).
  - Endpoints `PUT /api/v1/agendamentos/:id/triagem` e `GET /api/v1/atendimentos/:id/triagem` (ou embutido em `AtendimentoSchema`).
  - Cartão compacto de sinais vitais (PA, FC, FR, Tax, SpO₂, peso, altura, IMC calculado, dor) no topo de `Atendimento.tsx`, alimentando também o contexto exibido ao médico.

**Fora**
- Classificação de risco de pronto-socorro (Manchester / protocolos de urgência hospitalar).
- Painel de chamada em TV / senha de painel de espera (P3).

## Design

### Modelo de dados (Fatia 2)

```sql
CREATE TABLE triagens (
  id              uuid PRIMARY KEY,
  tenant_id       uuid NOT NULL,
  unidade_id      uuid NOT NULL,
  agendamento_id  uuid NOT NULL UNIQUE,      -- 1 triagem vigente por agendamento (sem FK entre módulos, ADR-PEP-0017)
  atendimento_id  uuid,                      -- preenchido ao iniciar o atendimento
  paciente_id     uuid NOT NULL,
  sinais_vitais   jsonb NOT NULL,            -- validado por Zod .strict() em @pep/contracts
  queixa_breve    text,
  alergias_revistas boolean NOT NULL DEFAULT false,
  autor_id        uuid NOT NULL,
  criado_em       timestamptz NOT NULL DEFAULT now(),
  atualizado_em   timestamptz NOT NULL DEFAULT now()
);
```

### Máquina de estados operacional na Agenda

Mantém a máquina de estados principal da [[RFC-PEP-003-agenda|RFC-PEP-003]] (`agendado → confirmado → chegou → em_atendimento → atendido`) para compatibilidade aditiva, acrescentando indicadores derivados de pré e pós-consulta:
- `triado: boolean` (derivado da existência de registro em `triagens` para o agendamento após `chegou`).
- `checkoutStatus`: `'nao_aplicavel' | 'pendente_recebimento' | 'pendente_documentos' | 'concluido'` (derivado de `recebiveis` particulares + `documentos` para entrega na recepção quando `status === 'atendido'`).

### Endpoints propostos

| Método | Rota | Permissão | Uso |
| :--- | :--- | :--- | :--- |
| `PUT` | `/api/v1/agendamentos/{id}/triagem` | `triagem:escrever` | Registra ou atualiza pré-consulta antes/durante atendimento |
| `GET` | `/api/v1/agendamentos/{id}/triagem` | `prontuario:ler` ou `triagem:escrever` | Lê dados da triagem |
| `GET` | `/api/v1/agenda/checkout?data=AAAA-MM-DD` | `agenda:ler` | Lista atendimentos finalizados do dia com pendência de recebimento particular, documentos ou retorno |

## Segurança e LGPD

- A rota `/api/v1/agenda/checkout` **nunca** expõe evolução clínica, CID ou conteúdo de receita; retorna apenas metadados operacionais (paciente, horário, status do recebível particular, quantidade de documentos assinados disponíveis para impressão e sugestão de retorno).
- Sinais vitais e queixa breve da triagem são protegidos pela mesma política de sigilo do módulo `prontuario` (inacessíveis ao `faturista` e à `recepcao`).

## Critérios de aceite

- [ ] Usuário apenas com perfil `medico` não consegue chamar `POST /api/v1/guias/:id/revisoes` ou `transicoes` (`403 Forbidden`), mas visualiza guias na Mesa (se tiver `guias:ler`) e no Histórico de Guias do paciente (`GET /api/v1/pacientes/:id/guias`).
- [ ] Recepcionista (`recepcao`) visualiza atendimento particular fechado na fila de Check-out da Agenda, registra recebimento, emite recibo PDF e agenda retorno sem precisar entrar em `/atendimento/:id`.
- [ ] Enfermeira (`enfermagem`) registra sinais vitais para paciente em estado `chegou`; ao médico iniciar o atendimento, os sinais vitais aparecem automaticamente no topo de `Atendimento.tsx`.
- [ ] RLS isolando `triagens` por `tenant_id` (teste com dois tenants).
