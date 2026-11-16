---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-018-backoffice-clinicas-e-onboarding.md
---

# RFC-PEP-018 — Backoffice: clínicas, onboarding, acesso de suporte e visão geral

**Fase:** B0 (P1, antes do piloto em produção) · **Status:** Em implementação (B0 entregue em homologação em 15/11/2026; ver Notas de implementação) · **ADRs:** ADR-PEP-0006, 0021, 0022
**Telas do backoffice:** 02 (visão geral, parcial), 03 (clínicas), 04 (detalhe da clínica: resumo, unidades e usuários, configuração somente leitura, acesso de suporte)
**Telas do PEP afetadas:** 25 (aprovação do acesso de suporte)
**Requisitos:** RF-BKO-04…09 ([[PRD-003-backoffice|PRD-003]])
**Depende de:** RFC-PEP-017; RFC-PEP-014 (trilha do tenant e tela 25) para o acesso de suporte

## Objetivo

Colocar uma clínica do piloto em produção sem SQL manual: criar o tenant, a clínica e as unidades, convidar o primeiro administrador, acompanhar o uso e, quando preciso, pedir acesso de suporte que a clínica aprova.

## Design

### Modelo (tabelas de plataforma, módulo `organizacao`)

Acréscimos às tabelas existentes (RFC-PEP-000/001):

```sql
ALTER TABLE tenants  ADD situacao text NOT NULL DEFAULT 'ativa' CHECK (situacao IN ('ativa','teste','suspensa','encerrada')),
                     ADD plano_rotulo text,                -- texto livre na B0; vira FK em assinaturas na B1
                     ADD cs_responsavel text,              -- sub do operador interno
                     ADD criado_por text NOT NULL;
ALTER TABLE clinicas ADD cnpj text, ADD registro_cnes text;  -- dados da PJ, não de paciente

CREATE TABLE convites (                -- plataforma
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL REFERENCES tenants(id),
  email text NOT NULL, perfil text NOT NULL CHECK (perfil = 'administrador'),
  unidade_ids uuid[] NOT NULL, expira_em timestamptz NOT NULL,
  aceito_em timestamptz, criado_por text NOT NULL);

CREATE TABLE acessos_suporte (         -- plataforma
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL REFERENCES tenants(id),
  solicitante_sub text NOT NULL, escopo text NOT NULL CHECK (escopo IN ('configuracoes','faturamento_metadados')),
  duracao interval NOT NULL CHECK (duracao <= interval '4 hours'), motivo text NOT NULL,
  estado text NOT NULL CHECK (estado IN ('pendente','aprovado','recusado','expirado','revogado')),
  decidido_por uuid, decidido_em timestamptz, expira_em timestamptz, criado_em timestamptz NOT NULL DEFAULT now());

CREATE TABLE uso_diario (              -- plataforma, alimentada pelo worker
  tenant_id uuid NOT NULL, unidade_id uuid NOT NULL, dia date NOT NULL,
  medicos_ativos int NOT NULL, analises int NOT NULL, aceitos int NOT NULL, editados int NOT NULL, rejeitados int NOT NULL,
  PRIMARY KEY (tenant_id, unidade_id, dia));
```

- `convites`, `acessos_suporte` e `uso_diario` têm `tenant_id` e, por convenção, RLS de tenant. Ficam também em `plataforma.txt`, que acrescenta a política `plataforma_leitura` (ADR-PEP-0022). O PEP lê e atualiza `acessos_suporte` do próprio tenant (tela 25) pelo `app_rw`, como qualquer tabela.
- O job diário `uso.consolidar` (worker, `app_admin`, auditado) agrega por unidade e por dia, a partir de `analise_tissia_ai` (RFC-PEP-005) e das sessões, só com contagens. Antes da RFC-PEP-005, só `medicos_ativos`.

### Onboarding

```
POST /bo/v1/clinicas           { tenant: { nome, situacao: 'teste'|'ativa', plano_rotulo? },
                                 clinica: { nome, cnpj?, registro_cnes? },
                                 unidades: [{ nome }] (1..20),
                                 administrador: { email } }      Idempotency-Key obrigatório
→ 201 { tenant_id, clinica_id, unidade_ids, convite_id }
```

1. Numa transação `Db.comPlataforma`, cria tenant, clínica, unidades (com `spike.modo = 'desligado'`, padrão do release plan) e o convite; publica `TenantProvisionado` no outbox.
2. O consumidor `identidade` cria o usuário no realm do PEP (Admin API do Keycloak), sem senha, com a ação obrigatória de definir senha e cadastrar segundo fator, e envia o e-mail do convite (link de 7 dias). O vínculo `administrador` com as unidades é criado no primeiro login, quando o convite é aceito.
3. Até a SPK-01, o provisionamento da unidade na Spike segue por script (`configurar_clinica.py` com `modo=visivel` explícito, ADR-PEP-0006). O backoffice mostra o lembrete "unidade sem provisionamento na Spike" até a flag sair de `desligado`.

`POST /bo/v1/clinicas/{tenant_id}/convites` reenvia o convite ou convida outro administrador.

### Carteira e detalhe

```
GET /bo/v1/clinicas?situacao=&modo=&busca=&cursor=      (tela 03)
GET /bo/v1/clinicas/{tenant_id}                         (tela 04: resumo, unidades, administradores vindos de `convites` aceitos, config vigente, uso 30 d)
GET /bo/v1/clinicas.csv                                 (exportar; mesma consulta)
POST /bo/v1/clinicas/{tenant_id}/suspensao   { motivo }   → 2ª aprovação em produção (estado 'pendente' até outro operador aprovar)
DELETE /bo/v1/clinicas/{tenant_id}/suspensao { motivo }
```

- A coluna **Saúde** da B0 é uma regra simples e documentada na tela: "Atenção" se não houve análises em 7 dias com o modo diferente de `desligado`, ou se o convite expirou sem aceite; senão, "Boa". `# ponytail:` regra fixa; score de saúde quando houver cobrança e desfechos (B1/B2).
- Glosa base → atual, aceite da IA e cobrança aparecem como "—" até as RFCs 012 e 019.
- Configuração (modo da IA por unidade, módulos, especialidades): **somente leitura** (RF-BKO-08).
- Suspensão: `situacao='suspensa'` bloqueia a aceitação de convites e a criação de novos vínculos. **Não** bloqueia login clínico, atendimento, prontuário nem IA.

### Acesso de suporte

```
POST /bo/v1/clinicas/{tenant_id}/acessos-suporte   { escopo, duracao_min (15..240), motivo }  → 201 { id, estado: 'pendente' }
GET  /bo/v1/clinicas/{tenant_id}/acessos-suporte
DELETE /bo/v1/acessos-suporte/{id}                 → revogado
```

No PEP (tela 25, perfil administrador), via `app_rw` do tenant:

```
GET  /api/v1/admin/acessos-suporte?estado=pendente
POST /api/v1/admin/acessos-suporte/{id}/decisao   { aprovar: boolean }
```

- Aprovar cria o vínculo temporário do usuário interno (conta espelho no realm do PEP, só WebAuthn) com o perfil `suporte_<escopo>` e `expira_em`. O guard do PEP rejeita o vínculo vencido. O job `acessos.expirar` marca `expirado`.
- Os perfis `suporte_configuracoes` e `suporte_faturamento_metadados` entram na matriz da RFC-PEP-001 **sem** nenhuma permissão `prontuario:*`, `paciente:*` ou de leitura de PII.
- Eventos `AcessoSuporteSolicitado`, `AcessoSuporteDecidido` e `AcessoSuporteExpirado` vão para as duas trilhas.

### Visão geral (tela 02, parcial)

`GET /bo/v1/visao-geral` → clínicas por situação, médicos ativos (7 d), unidades por modo da IA e lista "precisa de ação" (acessos pendentes há mais de 24 h, convites expirados, unidades sem provisionamento na Spike, suspensões pendentes de segunda aprovação). Os blocos de saúde da Spike e de negócio ficam com "Em breve" até a B1/B2.

## Segurança e LGPD

- O CNPJ, o nome e o e-mail do administrador são dados de PJ e de contato contratual e entram no RIPD (ADR-PEP-0022). Nenhum dado de paciente transita.
- O e-mail do convite não carrega dado clínico; o link é de uso único.
- Toda leitura de clínica gera `clinica.lida` na trilha interna.

## Critérios de aceite

- [ ] Onboarding de ponta a ponta em E2E: operador cria clínica com 2 unidades → e-mail capturado (Mailpit) → administrador define senha e cadastra TOTP → entra no PEP na unidade.
- [ ] `POST /bo/v1/clinicas` repetido com a mesma `Idempotency-Key` → mesmo resultado, sem duplicar.
- [ ] Unidade nova nasce com `spike.modo='desligado'`.
- [ ] `app_plataforma` não lê `vinculos`, pacientes nem prontuário (teste de permissão).
- [ ] Acesso de suporte: pendente → aprovado pela clínica → operador entra no PEP com perfil `suporte_configuracoes` → nenhuma rota de prontuário ou paciente responde 200 → vence e o próximo request dá 401.
- [ ] Suspensão em produção sem segunda aprovação não muda a situação; clínica suspensa continua atendendo.
- [ ] O CSV exportado não tem coluna de dado de paciente.

## Perguntas abertas

1. O `uso_diario` pode depender da RFC-PEP-005 (análises) ou a B0 sai só com médicos ativos?
2. É preciso servidor de e-mail transacional na P1 (convites)? Proposta: o mesmo provedor das notificações do PEP **[validar]**.
3. O administrador convidado deve ter perfil Administrador + Gestor por padrão (para operar a tela 15)?

## Notas de implementação

- `unidades.modo_ia` mantém `default('assistido')` na definição da coluna para compatibilidade com os fluxos existentes do PEP, e o onboarding (`POST /bo/v1/clinicas`) grava explicitamente `modoIa = 'desligado'`.
- `acessos_suporte` armazena `duracao_min` (15..240) como inteiro, e `vinculos` ganhou `expira_em timestamptz` para que vínculos temporários de suporte (`suporte_configuracoes`, `suporte_faturamento_metadados`) expirem no minuto exato sem depender apenas de `valido_ate` (`date`).
- O onboarding grava `TenantProvisionado` no `outbox` transacional e a aceitação automática do convite ocorre no primeiro login (`IdentidadeService.resolverUsuario`); o consumidor de envio de e-mail SMTP/Mailpit e criação automática no Keycloak Admin API não foi ligado ao container externo neste ambiente e o E2E (`apps/e2e/specs-backoffice/onboarding.spec.ts`) valida o fluxo completo em `AUTH_MODE=dev`.
