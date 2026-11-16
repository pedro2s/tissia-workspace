---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-003-agenda.md
---

# RFC-PEP-003 — Agenda multiprofissional e recepção

**Fase:** P1 · **Status:** Em implementação (Marco D e onda 2 do MVP entregues; pendentes na nota de 09/11/2026) · **ADRs:** ADR-PEP-0002, ADR-PEP-0003, ADR-PEP-0008
**Telas:** 03 (Agenda multiprofissional), 02 (widgets "Próximos atendimentos")
**Requisitos:** RF-AGE-01 a RF-AGE-09
**Depende de:** RFC-PEP-002 (paciente, beneficiário, elegibilidade), RFC-PEP-001 (profissionais, unidades, salas)

## Objetivo

Agendar e receber pacientes com a elegibilidade e a exigência de autorização prévia visíveis em cada horário, para que o problema de faturamento seja tratado **antes** do atendimento. O check-in abre o atendimento que o prontuário (RFC-PEP-004) usa.

## Escopo

**Dentro (P1)**

- [RF-AGE-01] Visões Dia (colunas por profissional e sala), Semana (um profissional) e Salas. Navegação por data e "Hoje".
- [RF-AGE-02] Grade de horários por profissional (dias, intervalo, duração padrão por tipo de atendimento) e salas compartilhadas (ex.: sala de procedimentos).
- [RF-AGE-03] Agendamento: paciente, profissional, sala, tipo (consulta, retorno, procedimento), convênio ou particular (ADR-PEP-0014), procedimento previsto (TUSS opcional), duração.
- [RF-AGE-04] Encaixe, bloqueio (reunião, higienização, indisponibilidade de sala) e remarcação, com motivo.
- [RF-AGE-05] Selo de elegibilidade por horário (última verificação válida da RFC-PEP-002) e ação "verificar todas de amanhã".
- [RF-AGE-06] Selo de autorização prévia: na P1, cadastro manual da exigência e da senha por agendamento; na P2, derivado das regras do contrato (RFC-PEP-008).
- [RF-AGE-07] Check-in, fila de espera e chamada; "em atendimento" ao abrir o prontuário.
- [RF-AGE-08] Lembretes por WhatsApp (confirmação, remarcação) via módulo `notificacoes`, só com consentimento ativo; painel enviados/confirmados/sem resposta/remarcar.
- [RF-AGE-09] Filtros por profissional, sala e convênio; atalho `N` para novo agendamento.

**Fora**

- Agendamento online pelo paciente e telemedicina: P3 (RFC-PEP-016).
- Overbooking automático e lista de espera inteligente.

## Design

### Modelo de dados

```sql
CREATE TABLE salas (id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL, nome text NOT NULL, compartilhada boolean NOT NULL DEFAULT false);

CREATE TABLE grades_horario (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  profissional_id uuid NOT NULL, unidade_id uuid NOT NULL, sala_id uuid,
  dia_semana smallint NOT NULL, inicio time NOT NULL, fim time NOT NULL,
  duracao_padrao_min smallint NOT NULL DEFAULT 30,
  vigencia_inicio date NOT NULL, vigencia_fim date
);

CREATE TABLE agendamentos (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  unidade_id       uuid NOT NULL,
  profissional_id  uuid,                 -- NULL para bloqueio de sala
  sala_id          uuid,
  paciente_id      uuid REFERENCES pacientes(id),
  beneficiario_id  uuid REFERENCES beneficiarios(id),   -- NULL = particular
  tipo             text NOT NULL,        -- consulta | retorno | procedimento | bloqueio
  procedimento_tuss text,
  inicio           timestamptz NOT NULL,
  fim              timestamptz NOT NULL,
  status           text NOT NULL,        -- ver máquina de estados
  exige_autorizacao boolean NOT NULL DEFAULT false,
  senha_autorizacao text,
  autorizacao_status text,               -- nao_exige | pendente | aprovada | negada
  confirmacao      text,                 -- nao_enviada | enviada | confirmada | sem_resposta | remarcar
  motivo           text,                 -- bloqueio/cancelamento/remarcação
  atendimento_id   uuid,                 -- preenchido no início do atendimento
  EXCLUDE USING gist (profissional_id WITH =, tstzrange(inicio, fim) WITH &&)
    WHERE (status NOT IN ('cancelado','remarcado') AND profissional_id IS NOT NULL)
);
-- mesma EXCLUDE para sala_id; encaixe usa flag `encaixe boolean` que desliga a constraint do profissional
```

`# ponytail:` conflito de horário garantido pela constraint `EXCLUDE` do Postgres, sem lógica de lock na aplicação. Encaixe é a exceção explícita.

### Máquina de estados

```
agendado → confirmado → chegou (check-in) → em_atendimento → atendido
   │            │            └→ faltou (sem check-in até fim + tolerância)
   └────────────┴→ cancelado | remarcado (novo agendamento ligado ao antigo)
```

`em_atendimento` é setado pelo prontuário ao abrir o atendimento; `atendido` pelo evento `AtendimentoFechado`.

### API (módulo `agenda`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/agenda?data=2026-11-06&visao=dia&unidade=&profissionais=&convenio=` | Grade do dia com agendamentos e selos |
| `POST` | `/api/v1/agendamentos` | Cria (409 em conflito, com o horário que conflita) |
| `PATCH` | `/api/v1/agendamentos/{id}` | Remarca/edita/cancela (`motivo` obrigatório) |
| `POST` | `/api/v1/agendamentos/{id}/check-in` | Chegada |
| `POST` | `/api/v1/agendamentos/{id}/iniciar` | Cria `Atendimento` (RFC-PEP-004) e muda para `em_atendimento` |
| `POST` | `/api/v1/agenda/elegibilidade/lote` | `{ "data": "2026-11-07" }` → lista de beneficiários a verificar (P1: worklist manual) |
| `POST` | `/api/v1/agenda/lembretes/reenviar` | Reenvia para `sem_resposta` |

Resposta da grade traz, por agendamento: `elegibilidade` (`elegivel`, `inelegivel`, `verificando`, `vencida`), `autorizacao_status`, `confirmacao` e `atendimento_id`.

### Eventos

- Consome `ElegibilidadeVerificada`, `ConsentimentoRevogado`, `AtendimentoFechado`.
- Publica `AgendamentoCriado`, `AgendamentoAlterado`, `PacienteChegou`, `AtendimentoIniciado` (este último é publicado pelo `prontuario`; a agenda só reage).
- `AgendamentoCriado` e `AgendamentoAlterado` disparam jobs BullMQ de lembrete (`notificacoes`) — D-1 e D-0, configurável por unidade.

### Regras

- Agendamento de convênio sem elegibilidade válida: permitido, com selo âmbar e item na lista "Elegibilidade pendentes".
- Inelegível para procedimentos: ação "oferecer particular" (vira agendamento particular, ADR-PEP-0014).
- `exige_autorizacao` sem senha aprovada até o horário: selo vermelho; o atendimento pode acontecer, a guia fica marcada (RFC-PEP-006).
- Lembretes respeitam consentimento `whatsapp` e janela de horário comercial; texto sem diagnóstico nem procedimento sensível.
- Atendimento realizado na agenda sem guia é insumo da receita esquecida (RFC-PEP-009, P2): o `atendimento_id` + `procedimento_tuss` previsto entram no cruzamento.

## UI

- Tela 03 hoje é feita à mão em `src/pages/Agenda.tsx`, orientada a dados (`APPTS`), pronta para receber a resposta da API. Trocar os arrays estáticos por chamada à rota de grade.
- Linha do "agora" atualizada a cada minuto; card do atendimento corrente com destaque.
- Painel lateral: Elegibilidade (pendentes), Autorização prévia (pendentes), Lembretes por WhatsApp (contadores), Legenda.
- Tablet (P1): a visão Dia precisa funcionar em 1194 px com rolagem horizontal da grade (tela 26d como referência de densidade).

## Segurança e LGPD

- Grade mostra id curto do paciente (`P-7F3A`) e não o nome para perfis sem acesso clínico (como no design). Recepção vê o nome.
- Lembrete por WhatsApp: só com consentimento; o provedor e o template ficam em [[integracoes-externas|integracoes.md]]. Nenhum dado clínico no texto.

## Critérios de aceite

- [ ] Dois agendamentos sobrepostos para o mesmo profissional ou sala → 409 (teste contra o banco real); encaixe explícito passa.
- [ ] Grade do dia com 4 colunas e 40 agendamentos responde em p95 ≤ 300 ms em staging.
- [ ] `iniciar` cria exatamente um atendimento (idempotente por agendamento).
- [ ] Sem consentimento `whatsapp`, nenhum job de lembrete é criado.
- [ ] Tela 03 ligada à API (dia, semana, salas; filtros; novo agendamento; check-in).

## Perguntas abertas

1. Provedor de WhatsApp (API oficial Meta via BSP) e custo por conversa: decidir em integracoes.md.
2. Tolerância para marcar "faltou" e política de cobrança de falta (particular).
3. Agenda por recurso além de sala (equipamento) entra na P1?

## Notas de implementação

**Marco D / D1 (08/11/2026): fatia entregue** ([[release-plan-pep#12-marco-d--mvp-de-demonstração-para-investidores|release plan §12]]).

- Entregue: tabela `agendamentos`, `GET /api/v1/agenda?data=` (visão Dia da unidade ativa, colunas por profissional), `POST /api/v1/agendamentos`, `POST .../check-in`, `POST .../iniciar`, máquina de estados `agendado/confirmado → chegou → em_atendimento` (409 fora dela, repetir é idempotente), eventos `AgendamentoCriado` e `PacienteChegou`, 6 testes. UI: tela 03 ligada à API (dia anterior/próximo/hoje, registrar chegada, iniciar atendimento), com estados de carregando, vazio e erro. Painéis laterais continuam de exemplo, marcados como prévia.
- **Desvios assumidos para a demo, a corrigir na P1:**
  - Sem `EXCLUDE USING gist` (conflito de profissional/sala), sem encaixe, bloqueio, remarcação, salas e grades de horário.
  - `elegibilidade` e `autorizacao_status` ficam **na linha do agendamento**, preenchidos pelo seed. Na P1 passam a ser derivados de `verificacoes_elegibilidade` (RFC-PEP-002) e das regras de contrato.
  - `iniciar` só muda o estado; a criação do `Atendimento` (e a idempotência por agendamento) entra no D2 com a RFC-PEP-004.
  - Sem lembretes por WhatsApp. Fuso fixo UTC-3 (`FUSO_AGENDA`), sem horário de verão desde 2019; revisar se o produto sair do horário de Brasília.
  - Sem FK entre `agendamentos` e `pacientes`/`usuarios` (ADR-PEP-0017); a API valida o paciente no tenant (422).
- Perfis: `agenda:escrever` (recepção) registra a chegada; `prontuario:escrever` (médico) inicia o atendimento. Em dev, `X-Dev-Perfil` aceita vários perfis separados por vírgula (ex.: `medico,recepcao`), como os vínculos múltiplos na mesma unidade.
- Seed: `pnpm --filter @pep/api seed:demo` recria a agenda de **hoje** (Unidade Centro, 2 profissionais, 8 pacientes sintéticos) e é repetível. Para a UI em dev: `VITE_DEV_TENANT_ID`, `VITE_DEV_UNIDADE_ID` e `VITE_DEV_PERFIL` (o seed imprime os ids).

**Onda 2 do MVP (09/11/2026): agenda completa e Início "Meu dia"** ([[release-plan-pep#13-plano-do-mvp-funcional-ondas|release plan §13]]).

- Entregue (RF-AGE-01, 03, 04, 05, 09): migração `0012` com `salas`, `EXCLUDE USING gist` por profissional (exceto encaixe) e por sala, e as colunas `sala_id`, `particular`, `encaixe`, `motivo` e `remarcado_de_id`. Conflito → 409 com o horário que conflita (teste contra o banco real). `GET /agenda?data&visao=dia|semana|salas&profissional&sala&convenio` (Semana = segunda a domingo), `GET /agenda/opcoes` (médicos com vínculo vigente e salas), `POST /agenda/bloqueios`, `PATCH /agendamentos/{id}` (`acao`: `cancelar` e `remarcar` com motivo obrigatório; `particular` para "oferecer particular") e `POST /agenda/elegibilidade/lote` (worklist manual de um dia). Remarcar marca o antigo como `remarcado` e cria o novo ligado a ele; só antes da chegada. Evento `AgendamentoAlterado`.
- **Elegibilidade derivada**: o selo vem da última linha de `verificacoes_elegibilidade` da carteirinha (`elegibilidadeDoHorario`, função pura em `@pep/contracts`): nunca verificada ou indisponível → `verificando`; mais de 30 dias → `vencida`; carência só bloqueia os horários antes do fim dela; carteirinha vencida → `inelegivel`; agendamento sem carteirinha ou marcado como particular → `particular`. A coluna `agendamentos.elegibilidade` ficou como legado (sem leitura nem escrita); sai numa próxima release. `CriarAgendamento` deixou de aceitar `elegibilidade` (422, schema estrito) e passou a aceitar `particular`, `encaixe` e `salaId`.
- UI: tela 03 com Dia (colunas por profissional e por sala compartilhada), Semana (um profissional) e Salas; filtros na URL (`nuqs`); linha do "agora"; novo agendamento (atalho `N`), bloqueio, encaixe, remarcar e cancelar; painéis de Elegibilidade e Autorização prévia derivados da grade, "Oferecer particular" e "Verificar todas de amanhã" (lista que leva à ficha). "Agendar" da ficha do paciente abre o mesmo formulário. Início (tela 02): KPIs, "Próximos atendimentos" e saudação reais; o médico vê a própria agenda, os demais perfis a da unidade.
- **Pendências e desvios:**
  - Sem grades de horário por profissional (RF-AGE-02): a grade da tela mostra 08:00–12:00 ou o que houver, sem horário "livre" nem "+ Encaixar" por slot. Sem `POST /salas`: as salas entram por seed até a RFC-PEP-001 ter a tela de cadastro.
  - RF-AGE-06: só o status de autorização cadastrado por agendamento (sem `exige_autorizacao`/`senha_autorizacao` ainda); `confirmacao`, lembretes por WhatsApp (RF-AGE-08) e o painel de lembretes seguem prévia até o módulo `notificacoes` (onda 6); fila de espera, chamada e "faltou" (RF-AGE-07) ficam sem tela.
  - `profissionalId` não é validado contra os vínculos da unidade (só `agenda:escrever`); a coluna do profissional sem vínculo aparece só se tiver agendamento.
  - Cartão "Evoluções para assinar" do Início fica inerte (RFC-PEP-011); as notificações continuam de exemplo (onda 6).
  - Tablet 1194 px: a grade rola na horizontal, sem a densidade da tela 26d.

- **Visão Semana conforme o canvas atualizado (13/11/2026, tela 03b).** Cartões compactos (62 px por meia hora) em vez dos de 160 px com botões; o clique abre o detalhe com as ações (registrar chegada, remarcar, cancelar, iniciar). Segunda a sábado, com o domingo só se houver horário; cabeçalho de cada dia com o resumo ("4 agend. · 1 falta"), coluna de hoje destacada, dias passados esmaecidos e horários livres tracejados. Painel da semana: números, ocupação por dia, pendências antes dos próximos horários, horários livres e legenda. Botão "Esta semana" e título "09 – 15 nov 2026".
  - **Lacuna:** "livre" é qualquer meia hora sem agendamento dentro da faixa da grade, de agora em diante, porque a jornada do profissional (RF-AGE-02) ainda não é cadastrada; o "Fora do expediente" do design e a "lista de espera" do painel entram com a jornada e com o módulo de lembretes (onda 6). "Oferecer encaixe por WhatsApp" fica visível e inerte.
