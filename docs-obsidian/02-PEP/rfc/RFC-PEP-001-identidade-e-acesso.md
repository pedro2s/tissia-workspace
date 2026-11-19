---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-001-identidade-e-acesso.md
---

# RFC-PEP-001 — Identidade, acesso, MFA, unidades e perfis

**Fase:** P0 · **Status:** Em implementação (API, fluxo na UI e login com segundo fator validados em E2E; tema da marca no Keycloak e realms de produção entregues (PR #25, #26 e #29); recuperação, auditoria de login e quebra-vidro pendentes) · **ADRs:** ADR-PEP-0003, 0010
**Telas:** 01a (credenciais), 01b (MFA e unidade), 25 (perfis e permissões, parcial)
**Requisitos:** RF-ACE-01…RF-ACE-12

## Objetivo

Autenticar profissionais com MFA obrigatório, escolher a clínica/unidade de trabalho e autorizar cada ação por perfil e unidade, com tudo registrado na trilha de auditoria.

## Escopo

| Dentro | Fora |
| :--- | :--- |
| Keycloak (OIDC) com TOTP, WebAuthn/FIDO2, códigos de recuperação | Login de paciente (portal do paciente) |
| Login com certificado ICP-Brasil (X.509) e SSO da organização | Assinatura de documentos (RFC-PEP-011) |
| Seleção de clínica/unidade e troca na Topbar | Provisionamento SCIM |
| RBAC com escopo por unidade; quebra-vidro | Billing por usuário |
| Bloqueio, sessão, confiança de dispositivo, auditoria de login | |

## Requisitos

| ID | Requisito |
| :--- | :--- |
| RF-ACE-01 | Login por e-mail e senha seguido de MFA obrigatório para todo perfil com acesso a dado clínico ou financeiro. |
| RF-ACE-02 | Segundo fator: TOTP, chave FIDO2/WebAuthn ou código de recuperação (10 códigos de uso único). |
| RF-ACE-03 | Alternativas de entrada: certificado ICP-Brasil (A1/A3) e SSO da organização (OIDC/SAML), configuráveis por tenant. |
| RF-ACE-04 | "Confiar neste dispositivo por 30 dias" dispensa o segundo fator nesse navegador; revogável pelo usuário e pelo admin. |
| RF-ACE-05 | 5 tentativas de senha erradas → bloqueio de 15 min e aviso ao admin da clínica. |
| RF-ACE-06 | Sessão encerrada após inatividade (padrão 30 min, configurável por tenant entre 15 e 120). |
| RF-ACE-07 | Após autenticar, o usuário escolhe clínica/unidade entre as que o perfil permite; a unidade padrão é pré-selecionada. |
| RF-ACE-08 | Troca de unidade sem novo login (Topbar), com novo token e registro de auditoria. |
| RF-ACE-09 | Autorização por permissão (RBAC) com escopo por unidade; a API nega por padrão. |
| RF-ACE-10 | Quebra-vidro: acesso emergencial a prontuário fora do escopo, com justificativa obrigatória, prazo e notificação ao DPO. |
| RF-ACE-11 | Login, falha, bloqueio, MFA, troca de unidade, quebra-vidro e mudança de permissão vão para a trilha de auditoria. |
| RF-ACE-12 | Estados da tela 01a: credencial inválida com tentativas restantes, Caps Lock, bloqueio, sessão expirada, senha redefinida. |

## Design

### Keycloak

- Um realm por ambiente (`pep-staging`, `pep-prod`). Tenant é atributo do usuário/grupo, não realm: evita centenas de realms e mantém SSO entre clínicas do mesmo profissional.
- Cliente `pep-web` (público, Authorization Code + PKCE) para a UI; cliente `pep-api` (bearer-only) para validar tokens.
- Fluxo de autenticação customizado: `Username Password Form` → `Conditional OTP` / `WebAuthn Authenticator` (obrigatório, exceto cookie de dispositivo confiável) → `X.509 Client Certificate` (alternativo) → Identity Provider por tenant (SSO).
- Políticas de senha: mínimo 12 caracteres, checagem contra senhas vazadas, histórico de 5.
- Brute force detection: 5 falhas → 15 min (RF-ACE-05). Evento enviado ao PEP por event listener (webhook) para notificar o admin.
- Tema do Keycloak com os tokens do design (telas 01a/01b) gerado via **Keycloakify** a partir dos mesmos tokens e primitivos React de `@pep/ui`, evitando duplicar CSS/FTL e mantendo fidelidade visual sem manipular senha diretamente na SPA; a seleção de unidade (01b, etapa 3) é tela do próprio `@pep/ui`, depois do redirect.

### Unidade ativa e token

1. Keycloak emite o token com `sub`, `tenant_ids` e `amr` (métodos usados).
2. A UI chama `GET /api/v1/eu/unidades` e mostra a etapa 3 da tela 01b.
3. `POST /api/v1/sessao/unidade { unidadeId }` valida o vínculo e devolve um token de sessão do PEP (JWT curto, 15 min, renovado por refresh) com `tenant_id`, `unidade_id`, `perfis` e `permissoes` resolvidos.
4. A API usa esse token: `tenant_id` alimenta o `SET LOCAL app.tenant_id` (RFC-PEP-000); `unidade_id` limita o escopo.

`# ponytail:` token de sessão próprio em vez de `token exchange` do Keycloak; trocar se surgir um segundo consumidor da identidade.

### Modelo

```sql
CREATE TABLE usuarios (
  id uuid PRIMARY KEY, keycloak_sub text UNIQUE NOT NULL,
  nome text NOT NULL, email text NOT NULL,
  conselho text, numero_conselho text, uf_conselho text,   -- CRM/COREN etc.
  ativo boolean NOT NULL DEFAULT true, criado_em timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE vinculos (                -- usuário × unidade × perfil (RLS por tenant)
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  usuario_id uuid NOT NULL REFERENCES usuarios(id),
  unidade_id uuid NOT NULL, perfil text NOT NULL,
  padrao boolean NOT NULL DEFAULT false,
  valido_ate date, criado_em timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE acessos_emergenciais (    -- quebra-vidro
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, usuario_id uuid NOT NULL,
  paciente_id uuid NOT NULL, justificativa text NOT NULL,
  expira_em timestamptz NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);
```

`usuarios` é global (um profissional atende em vários tenants) e não tem RLS; contém só dados profissionais. Todo o resto tem RLS.

### Perfis e permissões

Permissões finas (`recurso:ação`), perfis como conjuntos fixos por padrão e editáveis por tenant (tela 25).

| Perfil | Prontuário | Agenda | Guias/lotes | Financeiro | Admin/LGPD |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Médico | Lê e escreve (próprios atendimentos + pacientes da unidade) | Lê a própria | Revisa sugestões da Spike, assina | — | — |
| Enfermagem | Lê; escreve evolução de enfermagem | Lê | — | — | — |
| Recepção | Não lê evolução; lê dados cadastrais | Lê e escreve | Vê status de elegibilidade/autorização | — | — |
| Faturista | Lê só trechos vinculados à guia (evolução do atendimento faturado), somente leitura | Lê | Lê e escreve; gera lote | Lê | — |
| Gestor | Não lê evolução | Lê | Lê | Lê e escreve | Painéis |
| Admin da clínica | Não lê evolução | Configura | Configura | Configura | Usuários, perfis, integrações |
| DPO | Lê trilha de auditoria, não lê evolução | — | — | — | Consentimentos, solicitações de titular |
| Somente leitura | Conforme vínculo | Lê | Lê | — | — |

- Guard do NestJS `@Permissao('prontuario:ler')` + verificação de escopo de unidade no serviço. Negado por padrão.
- O faturista lê evolução apenas pelo endpoint de guia, que devolve o trecho do atendimento vinculado (alinhado à Q2 do roteiro de validação da Spike).

### API

| Rota | Uso |
| :--- | :--- |
| `GET /api/v1/eu` | Usuário, vínculos, unidade ativa |
| `GET /api/v1/eu/unidades` | Unidades disponíveis agrupadas por clínica (tela 01b, etapa 3) |
| `POST /api/v1/sessao/unidade` | Ativa unidade → token de sessão |
| `POST /api/v1/sessao/encerrar` | Logout (revoga refresh no Keycloak) |
| `GET/POST/DELETE /api/v1/eu/dispositivos-confiaveis` | Lista e revoga dispositivos de 30 dias |
| `GET/PUT /api/v1/admin/perfis` | Perfis e permissões do tenant (tela 25) |
| `GET/POST/PATCH /api/v1/admin/vinculos` | Usuário × unidade × perfil |
| `POST /api/v1/pacientes/{id}/acesso-emergencial` | Quebra-vidro (`{ justificativa }`) |

## UI

- **01a:** tema do Keycloak com os componentes do design: e-mail, senha com mostrar/ocultar, "Continuar (Enter)", certificado ICP-Brasil, SSO. Mensagens da coluna "Estados da etapa 1" mapeadas aos erros do Keycloak (RF-ACE-12). Caps Lock detectado no cliente.
- **01b etapa 2:** MFA no Keycloak (6 campos de dígito com auto-avanço e colar, "confiar 30 dias", FIDO2, recuperação, contagem de reenvio).
- **01b etapa 3:** tela do PEP com radio por unidade agrupado por clínica, selo "Padrão", botão "Entrar na <unidade>".
- **Topbar:** botão "Trocar unidade" reabre a etapa 3 em diálogo.
- Indicador de etapas (1 · Credenciais, 2 · Verificação, 3 · Clínica e unidade) igual ao design em todas as etapas.

## Segurança/LGPD

- MFA obrigatório; `amr` do token conferido pela API para rotas clínicas.
- Cookies `Secure`, `HttpOnly`, `SameSite=Lax`; refresh token rotativo.
- Sem enumeração de usuário: mensagem única "E-mail ou senha incorretos".
- Toda abertura de prontuário (inclusive quebra-vidro) gera registro "quem viu" (RFC-PEP-014).
- Certificado ICP-Brasil: validação de cadeia e revogação (OCSP/LCR) pelo Keycloak **[validar]** configuração da cadeia AC Raiz v5+.

## Critérios de aceite

- [x] Login sem segundo fator é impossível para perfis clínicos/financeiros (E2E `login.spec.ts`: só a senha leva à tela de cadastro do TOTP; nos acessos seguintes pede o código).
- [ ] TOTP e FIDO2 (autenticador virtual no Playwright) funcionam — **feito** (`login.spec.ts`, `webauthn.spec.ts`); código de recuperação usado não volta a valer — **pendente**.
- [ ] 5 senhas erradas → bloqueio de 15 min — **feito** (E2E confirma o bloqueio no Keycloak); evento de auditoria + notificação ao admin — **pendente**.
- [ ] Inatividade além do limite encerra a sessão e mostra o estado "Sessão encerrada por inatividade".
- [x] Usuário com vínculo em 2 clínicas vê as duas na etapa 3; sem vínculo em uma unidade, a API devolve 403 (teste de API `identidade.spec.ts`).
- [x] Teste de matriz de permissões: cada perfil × cada rota protegida devolve 2xx/403 conforme a tabela (hoje 3 rotas × 8 perfis; cresce com cada RFC).
- [ ] Quebra-vidro exige justificativa, expira no prazo e notifica o DPO.
- [ ] Todos os eventos do RF-ACE-11 aparecem na trilha de auditoria com usuário, IP, data e resultado.

## Perguntas abertas

| # | Pergunta | Bloqueia |
| :--- | :--- | :--- |
| Q1 | Keycloak autogerenciado (Cloud Run/GKE) ou serviço gerenciado (ex.: Cloud Identity Platform) se a ADR-PEP-0010 não for aceita? | Infra de identidade |
| Q2 | Faturista pode ler evolução só do atendimento faturado, ou nenhuma? (impacta a justificativa do Campo 23) | Permissões |
| Q3 | Tempo de inatividade padrão aceito pelas clínicas-piloto (30 min)? | RF-ACE-06 |
| Q4 | Login por certificado ICP precisa existir na P0 ou pode ir para a P2 junto com a assinatura? | Escopo P0 |

## Notas de implementação

08/11/2026 — primeira fatia (`apps/api`, `apps/ui`, `packages/contracts`):

**Entregue e testado (API, 36 testes com Postgres/Redis reais; UI, 7):**
- Módulo `identidade`: `usuarios` (global, sem RLS) e `vinculos` (RLS), migração `1730000000001_identidade.sql`.
- `GET /api/v1/eu`, `GET /api/v1/eu/unidades` (agrupado por clínica, vínculo expirado some), `POST /api/v1/sessao/unidade` (valida vínculo, emite o token de sessão de 15 min, HS256) e `GET/POST /api/v1/admin/vinculos` (`admin:usuarios`).
- Guard global `AcessoGuard`, nega por padrão: cada rota declara `@Publico`, `@Identidade` ou `@RequerPermissao`; um teste varre todos os controllers e falha se alguma rota não declarar.
- Matriz perfil × rota testada para os 8 perfis; permissões em `@pep/contracts` (`PERMISSOES_POR_PERFIL`). Token do Keycloak só vale em `/eu*` e `/sessao/unidade`; o de sessão só nas rotas de domínio. Tenant e unidade vêm só do token.
- Segundo fator exigido no token do Keycloak (`amr` ou `acr`), com perfil `dev` por cabeçalho apenas fora de produção.
- UI: `oidc-client-ts` (PKCE, tokens do Keycloak em memória), `/entrar`, `/entrar/callback`, `/entrar/unidade` (etapa 3 da tela 01b), guarda de rota, renovação do token de sessão a 80% do prazo, 401 volta ao login com o estado "Sessão encerrada por inatividade". `VITE_AUTH_MODE=dev` (padrão) mantém as telas abertas.
- Keycloak 26.0 no `docker compose` (perfil `identidade`, porta 8180) com realm `pep-dev` importado: 5 falhas → bloqueio de 15 min, senha ≥ 12 com histórico de 5, ociosidade de 30 min, cliente público `pep-web` com PKCE S256 e audience `pep-api`. `pnpm --filter @pep/api seed:dev <sub>` cria tenant, clínica, 2 unidades e vínculos.

**Achados do PoC contra o Keycloak real:**
- JWKS, `iss` e `aud` validados: token real aceito, adulterado = 401.
- **O Keycloak 26.0 não emite `amr`** (token só de senha traz `acr: "1"`). A API aceita `amr` OU `acr` ∈ `MFA_ACR_ACEITOS` (padrão `2`). **Pendente:** configurar no realm o fluxo com nível de autenticação (LoA 1 = senha, LoA 2 = OTP/WebAuthn) para que `acr=2` só apareça depois do segundo fator. Sem isso, em oidc o login só passa se um fluxo emitir `acr=2`.
- A ação padrão `CONFIGURE_TOTP` do realm também é aplicada a usuários criados pela API admin, mesmo com OTP importado.
- Não foi possível validar OTP ponta a ponta sem navegador (credencial OTP por import/admin API não foi aceita no grant direto); **TOTP, WebAuthn e recuperação continuam por verificar** (critérios de aceite abaixo).

**Pendente nesta RFC:** fluxo do realm com LoA e política de recuperação (10 códigos); tema do Keycloak (Keycloakify); bloqueio → evento ao admin (precisa de event listener no Keycloak); dispositivo confiável de 30 dias (`/eu/dispositivos-confiaveis`); `POST /sessao/encerrar` revogando o refresh; `GET/PUT /admin/perfis` (perfis editáveis); quebra-vidro (depende de pacientes, RFC-002); auditoria de login na trilha (RFC-014; hoje só o evento `UnidadeAtivada`/`VinculoCriado` no outbox); troca de unidade com diálogo na Topbar (hoje navega para `/entrar/unidade`); inatividade no cliente (a do servidor vem do Keycloak); login por certificado ICP e SSO, adiados para a P2 (decisão assumida para a Q4; confirmar).

**Decisões assumidas (a confirmar):** Q4 certificado ICP na P2; Q2 faturista lê só o trecho vinculado à guia (permissão `prontuario:ler_trecho_guia`); Keycloak autogerenciado.

08/11/2026 — 2ª fatia (login com segundo fator validado de ponta a ponta):

- **E2E real** (`apps/e2e`, Playwright; `pnpm e2e`, exige `pnpm infra:up` e o Keycloak do perfil `identidade`): sobe API (porta 3100, `AUTH_MODE=oidc`) e UI (porta 5273) contra o Keycloak. 5 cenários passam, repetíveis a partir de um realm recém-importado: (1) primeiro acesso: senha → cadastro do TOTP → escolha de unidade → `/home`; (2) rota protegida sem sessão volta ao login; (3) novo acesso: senha certa não basta, código errado é recusado, código certo entra; (4) 5 senhas erradas bloqueiam a conta; (5) chave FIDO2 com autenticador virtual: cadastro no primeiro acesso e exigência da chave no seguinte. Fora do `pnpm test` e da CI por depender de Keycloak e navegador.
- **Fluxo do realm** (`apps/api/infra/keycloak/pep-dev-realm.json`, fluxo `pep-browser`): senha → subfluxo condicional (`conditional-user-configured`) com TOTP **ou** FIDO2 como alternativas. Usuário sem fator recebe a ação obrigatória `CONFIGURE_TOTP` (padrão do realm) e só obtém token depois de cadastrá-lo; `webauthn-register` fica disponível como ação.
- **Prova do segundo fator no token:** como o fluxo do realm é a garantia (nenhum login conclui sem fator), o cliente `pep-web` emite `amr=["pwd","mfa"]` por mapper fixo. Vale só enquanto o realm tiver este único fluxo e nenhum IdP externo sem MFA. **Substituir pelo `amr` nativo** quando atualizar o Keycloak para uma versão que o emita **[validar: 26.1+]**. A API aceita `amr` ou `acr` ∈ `MFA_ACR_ACEITOS`; o caminho por `acr` (LoA) ficou sem uso.
- **Armadilhas do Keycloak 26.0 (custaram tempo, ficam registradas):**
  - Com LoA (`conditional-level-of-authentication`) e TOTP/FIDO2 como `ALTERNATIVE`, o login falha com "usuário ou senha inválida" (`AuthenticationFlowException`, log `org.keycloak.authentication` em DEBUG): a configuração de fator pelo próprio fluxo só dispara em execução `REQUIRED` e alternativas dentro de subfluxo `REQUIRED` são puladas. O padrão que funciona é o do fluxo nativo: condição "usuário configurado" + ação obrigatória para o primeiro cadastro.
  - Na importação do realm a chave certa é `authenticatorFlow` (a grafia antiga `autheticatorFlow` é ignorada).
  - Usuários criados pela API admin recebem `CONFIGURE_TOTP` por ser ação padrão, mesmo se a criação pedir outras.
- **Pendentes** (inalterados, ver bloco de 1ª fatia): código de recuperação (`recovery-authn-codes` do Keycloak, a validar nesta versão), tema com os componentes do design (Keycloakify), evento de bloqueio ao admin, dispositivo confiável de 30 dias, logout que revoga o refresh, perfis editáveis, auditoria de login (RFC-PEP-014), quebra-vidro (RFC-PEP-002), diálogo de troca de unidade, X.509/SSO (P2).

**Logout, Início e usuário no menu (09/11/2026).**

- **Sair** (RF-ACE-12): botão no rodapé da barra lateral (só com login OIDC). `sair()` em `apps/ui/src/auth/sessao.ts` zera o token de sessão do PEP (só em memória), cancela a renovação agendada e chama `signoutRedirect()` do Keycloak, encerrando também o SSO; sem isso o cookie do Keycloak faria o próximo acesso entrar sozinho. Coberto pelo E2E `shell` (`pnpm e2e:demo`). **Limite conhecido:** o token de sessão do PEP é sem estado e expira em até 15 min; não há lista de revogação no servidor (a P1 pode adicionar, junto da auditoria de sessão da RFC-PEP-014).
- O bloco do usuário na barra lateral mostra o nome (`GET /api/v1/eu`), os perfis da sessão e a unidade ativa, em vez do texto fixo do design; sem login (modo dev) continua o usuário de exemplo.
- A barra lateral passou a ter a altura da janela (`sticky`, `h-screen` a partir de 800 px): só os itens rolam; usuário e "Sair" ficam sempre visíveis, mesmo em páginas longas.
- Novo item **Início** no menu (e o logo leva a `/home`): antes a Home só aparecia logo após o login. É um acréscimo ao design.

- **Tema da marca no Keycloak (escopo mínimo):** o tema da marca foi feito como tema CSS sobre `keycloak.v2` em vez de Keycloakify, montando diretamente `/opt/keycloak/themes/tissia` no Compose e configurado em `pep-dev-realm.json`.

**Realms de produção `pep-prod` e `tissia-equipe-prod` (18/11/2026, PR #29 do `tissia`).** Importados no primeiro boot do Keycloak 26 da stack OCI (`infra/oci/keycloak/*.json`, runbook em `infra/oci/keycloak/README.md`, verificador `infra/oci/verificar-realms.mjs`). Sem usuário, senha ou secret nos JSONs; um realm não vê a equipe do outro. Divergências em relação ao realm de desenvolvimento e ao desenho desta RFC:

- **Segundo fator obrigatório no fluxo, `amr` fixo por mapper.** `pep-prod` exige senha **e** `auth-otp-form` (ambos `REQUIRED`); `tissia-equipe-prod` exige senha e `webauthn-authenticator`. O mapper emite `amr=["pwd","mfa"]` (`pep-web`) e `["pwd","hwk"]` (`bo-web`), e a API aceita por `MFA_AMR_ACEITOS`. O caminho por `acr`/LoA continua não funcionando no Keycloak 26.0 (armadilha registrada acima); `MFA_ACR_ACEITOS=2` não é o que libera o login neste desenho. O mapper fixo só é honesto enquanto cada fluxo exigir o fator: não acrescentar IdP externo, fluxo alternativo ou cliente novo com esse mapper sem o mesmo fluxo. No dev, quem não tinha fator cadastrado pulava o segundo fator (reproduzido); em produção isso foi fechado.
- **Desvio de RF-ACE-02:** nos clínicos, FIDO2 deixa de ser alternativa ao TOTP (a RF prevê TOTP **ou** FIDO2). Reabrir com um fluxo equivalente testado.
- **FIDO2 `cross-platform` na equipe:** a política WebAuthn só aceita chave física (USB, NFC, BLE); Touch ID, Windows Hello e passkey do aparelho não registram, para que `amr=hwk` signifique chave física. O Backoffice só aceita `amr` com `hwk` ou `webauthn`. Recomenda-se uma segunda chave de reserva.
- **Lockout (RF-ACE-05):** 5 falhas bloqueiam por 15 min (`waitIncrementSeconds=900`, medido); `minimumQuickLoginWaitSeconds=60` para que 2 falhas rápidas não travem a conta por 15 min (evita negação de serviço contra quem só tem o e-mail da vítima). O aviso ao admin da clínica continua pendente.
- **`offline_access` removido** dos escopos dos clientes (daria refresh token que sobrevive ao logout; `invalid_scope` testado) e `fullScopeAllowed=false`; `admin-cli` sem direct grant.
- **`resetPasswordAllowed=false`** nos dois realms: o Keycloak do Compose não tem SMTP e o fluxo de redefinição por e-mail não passa pelo segundo fator (o `amr` fixo marcaria o token como `mfa`). A redefinição é feita pelo administrador do Keycloak, com senha temporária.
- Outros: `sslRequired=external`, redirects com os caminhos exatos da UI (sem curinga), e-mail editável só pelo administrador (a API vincula convites pelo claim `email`), eventos de login e administrativos ligados por 90 dias.
- **Lacunas registradas no runbook (não resolvidas):** a API vincula o convite sem checar `email_verified` (mitigado pelos realms; virar proposta SPK), sem códigos de recuperação, sem alerta de bloqueio ao administrador, e o `master` não vem do JSON (proteção contra força bruta desligada por padrão, recomendada à mão). Os realms só são importados se ainda não existirem: mudar o JSON depois do primeiro boot não altera o realm em produção.
