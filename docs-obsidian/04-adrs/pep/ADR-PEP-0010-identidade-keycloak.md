---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0010-identidade-keycloak.md
---

# ADR-PEP-0010 — Identidade com Keycloak (OIDC, TOTP, WebAuthn/FIDO2, certificado ICP, SSO)

**Status:** Aceita — 09/11/2026 (proposta em 07/11/2026; já implementada: login OIDC, TOTP e WebAuthn com E2E. Certificado ICP-Brasil, SSO por tenant e códigos de recuperação seguem pendentes na [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]])

## Contexto

As telas 01a/01b exigem: e-mail + senha, MFA por TOTP ou chave FIDO2, códigos de recuperação, login por **certificado ICP-Brasil**, **SSO da organização**, "confiar neste dispositivo por 30 dias", bloqueio temporário após 5 tentativas com aviso ao administrador, sessão encerrada por inatividade e escolha de clínica/unidade. MFA para profissionais é requisito do design de produto. Escrever tudo isso à mão é caro e arriscado.

## Decisão

- **Keycloak** (self-hosted, mesma região, [[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009]]) como provedor OIDC; o backend NestJS só valida tokens (JWT) e resolve autorização.
- **Fluxos:** senha + OTP (TOTP) ou WebAuthn (FIDO2) obrigatório para perfis clínicos e de faturamento; códigos de recuperação; *brute force detection* com bloqueio após 5 falhas por 15 min e evento ao administrador; política de sessão ociosa.
- **Certificado ICP-Brasil:** autenticador X.509 do Keycloak com a cadeia ICP-Brasil confiável **[validar suporte a A3 em token/cartão via navegador]**.
- **SSO:** *identity brokering* (OIDC/SAML) por tenant.
- **Um realm** para o produto; tenant e unidades como claims/grupos. A escolha de unidade (tela 01b) acontece no PEP, depois do login, e vai como contexto de sessão (não um novo token por unidade).
- **Autorização** (perfis, permissões por módulo e unidade) fica no módulo `identidade` do PEP, não no Keycloak ([[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]]).
- Telas de login com o tema do design (Keycloak theme ou *login pages* próprias via API).

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Auth0 / Okta | Custo por usuário ativo e dado de identidade fora do Brasil **[validar]** |
| AWS Cognito / Firebase Auth | Sem login X.509 ICP nativo; amarra a nuvem |
| Código próprio (Passport + bibliotecas) | Cobrir MFA, FIDO2, X.509, SSO e bloqueio com segurança é o projeto inteiro |

## Consequências

- Keycloak vira componente crítico: HA, backup do banco dele, atualização de segurança.
- Customizar as telas para ficarem iguais ao design tem custo; aceitar o tema adaptado no P0 se necessário.

**Resultado do PoC (08/11/2026), atualizado:** TOTP e WebAuthn/FIDO2 validados em E2E (`apps/e2e`), bloqueio de 15 min após 5 falhas e primeiro acesso com cadastro obrigatório do TOTP funcionam. Faltam recuperação, X.509 e brokering (P2). Aceitar quando o Pedro aprovar a operação do Keycloak (custo/HA) e o `amr` nativo substituir o mapper fixo. Histórico do primeiro resultado parcial:

**Resultado parcial do PoC (08/11/2026):** realm importado, PKCE S256, bloqueio por força bruta e política de senha OK; a API valida JWKS/issuer/audience contra o Keycloak 26.0 real. **O Keycloak 26.0 não emite `amr`**; o segundo fator precisa ser provado por `acr` (LoA do fluxo). Faltam: fluxo com LoA, TOTP/WebAuthn/recuperação ponta a ponta, X.509 e brokering. Detalhes na nota de implementação da [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]].

**Para aceitar:** prova de conceito no P0 cobrindo TOTP, WebAuthn, X.509 ICP e brokering; Pedro aprovar operação do Keycloak vs. custo de um SaaS com região Brasil.
