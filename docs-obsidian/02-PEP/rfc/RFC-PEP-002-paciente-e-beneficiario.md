---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-002-paciente-e-beneficiario.md
---

# RFC-PEP-002 — Paciente, beneficiário e elegibilidade

**Fase:** P1 · **Status:** Implementada (fatia do Marco D e onda 1 do MVP entregues; pendentes na nota de 09/11/2026) · **ADRs:** ADR-PEP-0002, ADR-PEP-0003, ADR-PEP-0005, ADR-PEP-0012
**Telas:** 04 (Cadastro de paciente e carteirinhas)
**Requisitos:** RF-PAC-01 a RF-PAC-08
**Depende de:** RFC-PEP-000 (tenancy, RLS, outbox), RFC-PEP-001 (perfis), RFC-PEP-014 (auditoria de acesso e consentimentos)

## Objetivo

Cadastro único do paciente por tenant, com seus vínculos a operadoras (beneficiário), verificação de elegibilidade e histórico de guias. É a fonte de PII do PEP: tudo que sai para a Spike sai daqui pseudonimizado (ADR-PEP-0005).

## Escopo

**Dentro (P1)**

- [RF-PAC-01] Cadastro de paciente: nome civil, nome social, CPF, data de nascimento, sexo TISS (`M`/`F`/`I`), CNS, contato (celular/WhatsApp, e-mail), responsável legal, alergias, preferência de canal.
- [RF-PAC-02] Mascaramento por padrão de CPF, CNS, nascimento, telefone e número de carteirinha. "Revelar" exige justificativa e gera registro de auditoria.
- [RF-PAC-03] Beneficiário: paciente × operadora × carteirinha × plano × acomodação × titular/dependente × validade. Uma carteirinha principal por paciente.
- [RF-PAC-04] Elegibilidade **manual** na P1: a recepção registra o resultado (elegível / inelegível com motivo / carência até) com data, autor e fonte (`manual`, `portal`, `telefone`).
- [RF-PAC-05] Alertas de cadastro: validade vencida, última verificação antiga (> 30 dias, configurável), carência registrada.
- [RF-PAC-06] Consentimentos LGPD do paciente (tratamento, escuta ambiental, envio por WhatsApp, CID em atestado), com versão do termo e revogação.
- [RF-PAC-07] Histórico de guias do paciente (lista paginada, status, valor, link para a guia).
- [RF-PAC-08] Busca por nome, nome social, data de nascimento, últimos dígitos do CPF ou da carteirinha, dentro da unidade/tenant.

**Fora**

- Elegibilidade por webservice TISS da operadora: P2, via parceiro ou motor próprio (ADR-PEP-0012).
- Portal do paciente e app do paciente: P3.
- Importação de cadastro de concorrentes: ADR-PEP-0016 / RFC-PEP-016.

## Design

### Modelo de dados

```sql
-- todas as tabelas: tenant_id uuid NOT NULL (RLS forçada e política aplicadas por convenção em drizzle/rls, ADR-PEP-0019)
CREATE TABLE pacientes (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  nome_civil       text NOT NULL,             -- cifrado na aplicação (ver Segurança)
  nome_social      text,
  cpf              text,                      -- cifrado; cpf_hash para busca exata
  cpf_hash         bytea,
  cpf_final        char(2),                   -- busca por últimos dígitos sem decifrar
  nascimento       date NOT NULL,
  sexo_tiss        char(1) NOT NULL CHECK (sexo_tiss IN ('M','F','I')),
  cns              text,                      -- cifrado
  celular          text,                      -- cifrado
  email            text,                      -- cifrado
  responsavel_id   uuid REFERENCES pacientes(id),
  alergias         jsonb NOT NULL DEFAULT '[]',
  canal_preferido  text,                      -- whatsapp | email | telefone
  criado_em        timestamptz NOT NULL DEFAULT now(),
  atualizado_em    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_pacientes_cpf ON pacientes (tenant_id, cpf_hash) WHERE cpf_hash IS NOT NULL;

CREATE TABLE beneficiarios (
  id                uuid PRIMARY KEY,
  tenant_id         uuid NOT NULL,
  paciente_id       uuid NOT NULL REFERENCES pacientes(id),
  operadora_id      uuid NOT NULL,   -- id da operadora (módulo contratos; sem FK cross-módulo, ADR-PEP-0017)
  numero_carteira   text NOT NULL,           -- cifrado; carteira_final char(4)
  carteira_final    char(4),
  plano             text,
  acomodacao        text,                    -- enfermaria | apartamento | ambulatorial
  titularidade      text NOT NULL,           -- titular | dependente
  validade          date,
  principal         boolean NOT NULL DEFAULT false,
  ativo             boolean NOT NULL DEFAULT true
);
CREATE UNIQUE INDEX ux_benef_principal ON beneficiarios (paciente_id) WHERE principal AND ativo;

CREATE TABLE verificacoes_elegibilidade (
  id               uuid PRIMARY KEY,
  tenant_id        uuid NOT NULL,
  beneficiario_id  uuid NOT NULL REFERENCES beneficiarios(id),
  resultado        text NOT NULL,            -- elegivel | inelegivel | carencia | indisponivel
  escopo           text,                     -- consulta | procedimentos | todos
  carencia_ate     date,
  motivo           text,
  fonte            text NOT NULL,            -- manual | portal | telefone | webservice (P2)
  protocolo        text,
  verificado_por   uuid,                    -- usuario_id (módulo identidade; sem FK cross-módulo)
  verificado_em    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE consentimentos (
  id            uuid PRIMARY KEY,
  tenant_id     uuid NOT NULL,
  paciente_id   uuid NOT NULL REFERENCES pacientes(id),
  finalidade    text NOT NULL,   -- tratamento | escuta_ambiental | whatsapp | cid_em_atestado
  versao_termo  text NOT NULL,
  canal         text NOT NULL,   -- verbal_registrado | termo_digital | papel_digitalizado
  concedido_em  timestamptz NOT NULL,
  revogado_em   timestamptz,
  registrado_por uuid                       -- usuario_id (módulo identidade; sem FK cross-módulo)
);
```

`operadoras` (registro ANS, nome, versão TISS exigida) é cadastro do módulo `contratos`, compartilhado já na P1 em forma mínima.

### API (módulo `pacientes`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/pacientes?q=` | Busca; retorna sempre campos mascarados |
| `POST` | `/api/v1/pacientes` | Cria |
| `PATCH` | `/api/v1/pacientes/{id}` | Edita (registra diff na auditoria, sem valores de PII) |
| `POST` | `/api/v1/pacientes/{id}/revelar` | `{ "campos": ["cpf"], "justificativa": "Conferência com documento" }` → valores em claro, uma vez |
| `GET/POST` | `/api/v1/pacientes/{id}/beneficiarios` | Lista/adiciona carteirinha |
| `POST` | `/api/v1/beneficiarios/{id}/elegibilidade` | `{ "resultado": "carencia", "escopo": "procedimentos", "carencia_ate": "2026-12-15", "fonte": "portal" }` |
| `GET/POST/DELETE` | `/api/v1/pacientes/{id}/consentimentos` | Concede/revoga (revogar = `revogado_em`, nunca delete) |
| `GET` | `/api/v1/pacientes/{id}/guias` | Histórico de guias (lê do módulo `faturamento`) |

### Eventos

- `PacienteCriado`, `BeneficiarioAtualizado`, `ElegibilidadeVerificada` (consumidos por `agenda` para o selo do horário).
- `ConsentimentoRevogado` (consumido por `notificacoes` e pelo fluxo de voz da RFC-PEP-013).

### Regras

- Sexo TISS e data de nascimento são obrigatórios: a Spike precisa de `sexo` e `idade` (calculada na data do atendimento, RFC-PEP-005).
- Elegibilidade "vencida" após `elegibilidade_validade_dias` (default 30, por tenant) gera alerta na agenda e na guia.
- Carência com escopo `procedimentos` não bloqueia consulta (como na tela 04: "Consultas liberadas").
- Nenhuma operação apaga paciente. Exclusão a pedido do titular segue a RFC-PEP-014 (anonimização respeitando a guarda mínima de prontuário **[validar prazo — Lei 13.787/2018]**).

## UI

- Tela 04 era estática (`apps/ui/src/modulos/agenda/Paciente.tsx`). Componentes a extrair: cabeçalho do paciente (reaproveitado nas telas 05, 06, 12), cartão de carteirinha com selo de elegibilidade, tabela de histórico de guias.
- Abas: Cadastro e carteirinhas · Histórico de guias · Documentos · Consentimentos.
- Aviso fixo no bloco de dados pessoais: "Dados sensíveis mascarados. Revelar exige justificativa e fica na trilha de auditoria."
- Cartão de orientação ("Para agendar procedimento hoje, use a Operadora A…") é derivado das verificações e das regras de autorização (RFC-PEP-008 na P2; na P1 só elegibilidade).

## Segurança e LGPD

- PII cifrada em nível de campo (AES-256-GCM, chave por tenant via KMS — ADR-PEP-0009) além da criptografia em repouso do banco. Hashes com sal por tenant para busca exata.
- Toda leitura de ficha de paciente gera evento de acesso ("quem viu") no módulo `auditoria` (RFC-PEP-014). "Revelar" gera evento separado com a justificativa.
- Perfil faturista vê carteirinha completa (necessária para a guia) mas não CPF/contato sem "revelar".
- Nenhum campo deste módulo vai à Spike; só `paciente.ref` (HMAC), `sexo` e `idade` (RFC-PEP-005).

## Critérios de aceite

- [ ] Teste de RLS: usuário do tenant A não lê paciente do tenant B, mesmo com id válido (teste de integração com dois tenants).
- [ ] Resposta de busca e de ficha nunca contém CPF/CNS/telefone em claro; snapshot de resposta verificado em teste.
- [ ] `revelar` sem justificativa → 422; com justificativa → registro de auditoria com usuário, campos, justificativa e horário.
- [ ] Só uma carteirinha principal ativa por paciente (constraint).
- [ ] Revogar consentimento de WhatsApp impede envio na RFC-PEP-003 e na RFC-PEP-004 (teste de integração).
- [ ] Tela 04 ligada à API: cadastro, carteirinhas, verificação manual e histórico.

## Perguntas abertas

1. Quais campos de cadastro são obrigatórios por operadora no TISS 4.03.00 (ex.: CNS) **[validar]**?
2. Guardar foto/documento do paciente na P1 (armazenamento de objetos) ou só na P3?
3. Política de deduplicação quando não há CPF (recém-nascido, estrangeiro).

## Notas de implementação

**Marco D / D1 (08/11/2026): fatia entregue** ([[release-plan-pep#12-marco-d--mvp-de-demonstração-para-investidores|release plan §12]]).

- Entregue: `pacientes` e `beneficiarios` (uma carteirinha principal por paciente, índice parcial único), `GET/POST /api/v1/pacientes`, `GET /api/v1/pacientes/{id}`, busca por nome e nome social, evento `PacienteCriado`, RLS por convenção, 3 testes (mascaramento, RLS entre tenants, campo extra e sexo TISS inválido).
- **Desvios assumidos para a demo, a corrigir na P1:**
  - Sem CPF, CNS, contato, responsável, alergias nem consentimentos: o cadastro mínimo não guarda PII direta. `numero_carteira` fica **em claro** no banco (dado sintético na demo); a API só devolve `••••` + 4 últimos dígitos. A cifragem por campo com KMS entra antes de qualquer dado real (release plan §4).
  - Sem `revelar`, sem elegibilidade manual e sem histórico de guias; `operadora` é só o nome (e o registro ANS opcional), sem cadastro em `contratos`.
  - Sem trilha "quem viu" (RFC-PEP-014). Os logs não trazem corpo de requisição.
- O identificador curto (`P-7F3A`) é derivado dos 4 últimos hex do id; colisões são possíveis em bases grandes. Não usar como chave.

**Onda 1 do MVP (09/11/2026): cadastro completo, carteirinhas, elegibilidade manual, consentimentos e tela 04 ligada à API** ([[release-plan-pep#13-plano-do-mvp-funcional-ondas|release plan §13]]).

- Entregue (RF-PAC-01 a 08): CPF, CNS, celular, e-mail, canal preferido, responsável (`responsavel_id`) e alergias; `PATCH /pacientes/{id}`; `POST /pacientes/{id}/revelar` (permissão nova `pacientes:revelar`: médico, recepção e faturista; justificativa ≥ 10 caracteres, senão 422); `GET/POST /pacientes/{id}/beneficiarios`, `PATCH /beneficiarios/{id}` (principal, inativar, plano, acomodação, validade); `POST /beneficiarios/{id}/elegibilidade` (manual, insert-only em `verificacoes_elegibilidade`; carência exige data, inelegível exige motivo); `GET/POST /pacientes/{id}/consentimentos` e `DELETE …/consentimentos/{cid}` (revoga com `revogado_em`, um vigente por finalidade); `GET /pacientes/{id}/guias` (lido do `faturamento`). Busca por nome, CPF exato (hash), 2 últimos do CPF, 4 últimos da carteirinha e nascimento. Eventos `BeneficiarioAtualizado`, `ElegibilidadeVerificada`, `ConsentimentoRevogado`.
- **Cifragem por campo:** CPF, CNS, celular, e-mail e número da carteirinha em AES-256-GCM com chave por tenant (`apps/api/src/comum/cifra.ts`); `cpf_hash` (HMAC com sal por tenant) para busca exata. A API nunca devolve esses valores nem a data de nascimento: só máscaras (`•••.•••.•••-41`, `••/••/1972`) e a idade. As carteirinhas sintéticas antigas, em claro, continuam legíveis (valor sem o prefixo `v1.`); o `seed:demo` já as grava cifradas.
- **Auditoria:** abrir a ficha grava `paciente.visualizado`; revelar grava `paciente.revelado` com campos e justificativa; editar grava `paciente.editado` só com os nomes dos campos. Na mesma transação da leitura: sem rastro, sem resposta (núcleo da RFC-PEP-014 antecipado).
- UI: `/pacientes` (busca com `?q=` na URL via `nuqs`, novo paciente) e `/pacientes/:id` (tela 04 fiel ao design: cabeçalho com selos de alergia, consentimento e canal; dados pessoais com Revelar e Editar; carteirinhas com selo de elegibilidade e alertas de RF-PAC-05; orientação derivada da elegibilidade; histórico de guias; aba Consentimentos). Ações escondidas sem a permissão (`usePode`). "Abrir prontuário" leva ao atendimento mais recente; o menu "Prontuário" leva à busca de pacientes.
- **Desvios e pendências:**
  - `ponytail:` a chave do tenant é derivada (HKDF) de `PII_CHAVE_MESTRA` (variável de ambiente, obrigatória em produção). A DEK por tenant envelopada por KEK no Cloud KMS (RFC-PEP-014 §1.6) entra antes de qualquer dado real; o prefixo `v1.` permite re-cifrar.
  - `cpf_hash` é `text` (hex), não `bytea`. Sem `operadoras`/`operadora_id` (segue `operadora_nome`, RFC-PEP-008 fatia 0). Sem deduplicação sem CPF (pergunta 3).
  - Histórico de guias sem paginação (limite 200) e sem pago/glosado (RFC-PEP-007). "Exportar", "Verificar todas" (webservice, P2), "Coberturas" e "Agendar" (onda 2) ficam inertes com o motivo. Aba Documentos é prévia (documentos por paciente entram com o resto da RFC-PEP-004).
  - "Revogar WhatsApp impede envio" fica para quando houver envio (RFC-PEP-003/004); o evento já é publicado. A agenda ainda usa a elegibilidade da linha do agendamento (onda 2).
  - Prazo de verificação antiga fixo em 30 dias (`ELEGIBILIDADE_VALIDADE_DIAS`); por tenant numa próxima fatia.
