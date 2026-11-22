---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-011-assinatura-icp-brasil.md
---

# RFC-PEP-011 — Assinatura digital ICP-Brasil

**Fase:** P2 · **Status:** Implementada (detalhada em 09/11/2026; a [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]] ainda está **Proposta** e precisa ser aceita antes da fatia 2, a do PSC real) · **ADRs:** ADR-PEP-0011 (Proposta), ADR-PEP-0007, ADR-PEP-0008, ADR-PEP-0009, ADR-PEP-0010, ADR-PEP-0017
**Telas:** 07 (Revisão pós-consulta e assinatura), 08 (Documentos do atendimento), 25 (cartão "Assinatura ICP-Brasil" em Integrações e API); pendência na 02
**Requisitos:** RF-ASS-01, RF-ASS-02; completa RF-DOC-01 (receita de controle especial e antimicrobiano) e RF-DOC-06 (ICP-Brasil na P2)
**Depende de:** RFC-PEP-004 (evolução versionada com `hash`, adendos, documentos e PDF dos documentos, RF-DOC-04), RFC-PEP-013 (`trechos_ia`, a trilha de revisão da voz), RFC-PEP-005 (decisões sobre sugestões de IA), RFC-PEP-014 (auditoria, envelope encryption com KMS por tenant, painel de integrações), RFC-PEP-001 (perfis, MFA); externas: contrato com um PSC credenciado (Q1) e uma ACT para o carimbo do tempo (Q4)

## Objetivo

Assinar a evolução definitiva, os adendos e os documentos clínicos (receita, pedido de exame, atestado, encaminhamento) com certificado ICP-Brasil, só depois que cada trecho gerado por IA tiver decisão humana, com a trilha de revisão anexada ao PDF assinado. Depois de assinado, nada muda; correção só por adendo, que também é assinado. A assinatura nunca bloqueia o fechamento do atendimento nem o faturamento: se o PSC falhar, o atendimento fecha e a assinatura fica pendente.

## Escopo

**Dentro (P2)**

- [RF-ASS-01] Assinatura ICP-Brasil em **PAdES** de PDF gerado pelo PEP, com carimbo do tempo de ACT credenciada (política AD-RT por padrão **[validar política exigida por tipo de documento, Q4]**), para: evolução definitiva do atendimento, adendo, receita (simples, controle especial, antimicrobiano), pedido de exame, atestado e encaminhamento.
- [RF-ASS-01] Certificado **em nuvem** (A3 em HSM de PSC credenciado pelo ITI) como caminho principal, integrado por OAuth do PSC; o domínio conhece só a porta `ServicoAssinatura` (padrão de [[integracoes-externas|integracoes]]). A1 guardado cifrado fica como fatia condicionada à Q5.
- [RF-ASS-01] Trilha de revisão anexada ao PDF da evolução (cada trecho de IA, decisão aceito/editado/descartado, autor e horário) ou, se o médico desmarcar "Anexar trilha", referenciada por hash.
- [RF-ASS-01] Assinatura de vários itens do mesmo atendimento em uma solicitação ("Assinar pendentes (N)"), com uma autorização do titular quando o PSC oferecer sessão de assinatura (Q2).
- [RF-ASS-02] Bloqueio no servidor: nenhum item com trecho de IA sem decisão entra em solicitação (409, mesma mensagem da tela 07).
- Verificação: código de verificação e QR impressos no PDF, com página pública que confirma validade e metadados **sem expor conteúdo clínico** (Q8).
- Convivência com a assinatura eletrônica simples que já existe (Marco D): o que foi assinado em modo simples continua válido e imutável; a unidade pode exigir ICP.
- Cadastro do certificado pelo profissional, aviso de vencimento (30 dias) e cartão na tela 25.

**Fora**

- Assinatura do lote XML TISS (`GuiaAssinada`): não exigida no piloto **[validar, RFC-PEP-006 Q3]**; o evento fica reservado.
- Reassinar com ICP documento já assinado em modo simples: não se faz (seria um segundo ato sobre documento fechado; se for preciso, adendo).
- Notificação de receita A e B (talonário físico), fora de qualquer fluxo digital **[validar Portaria SVS/MS nº 344/1998 e normas posteriores]**.
- Plataforma de prescrição de terceiros (bulário, validação em farmácia): decisão da Q3; esta RFC entrega o PDF assinado próprio, que é o fallback previsto em [[integracoes-externas|integracoes]].
- Login com certificado ICP (tela 01a): ADR-PEP-0010 / RFC-PEP-001.
- Recurso de glosa como documento assinado: decisão da RFC-PEP-007; se for, usa a mesma solicitação desta RFC com `alvo_tipo = documento`.
- CAdES sobre o JSON canônico da versão (citado como opção na ADR-PEP-0011): um formato só (PAdES) na P2.

## Design

### Estado atual (Marco D) e como migra

| Hoje | Onde | Depois desta RFC |
| :--- | :--- | :--- |
| Documento `rascunho → revisado → assinado`, assinatura simples (`assinado_em`, `assinado_por`), trigger `documento_assinado_imutavel` bloqueia `UPDATE`/`DELETE` de assinado | `apps/api/src/modulos/documentos/`, migração `0006` | Ganha o estado `aguardando_assinatura` e as colunas `assinatura_padrao` (`simples` \| `icp`) e `assinatura_id`. A rota atual de assinatura continua para o modo simples |
| "Assinar e finalizar" grava a versão da evolução e fecha o atendimento; nenhum registro de assinatura | `EscutaService.finalizar`, `ProntuarioService.fechar` | Fecha igual (nunca bloqueado pelo PSC) e, se o profissional tem certificado ativo, cria a solicitação ICP da versão definitiva |
| Trilha de revisão em `trechos_ia` + `sessoes_escuta.trilha_anexada` | `modulos/prontuario` | Vira apêndice do PDF da evolução (ou hash dele) |
| Módulo `assinatura` vazio | `modulos/assinatura/assinatura.module.ts` | Recebe tabelas, porta, adaptadores, API e job |

Migração de `documentos` (o trigger dispara em `UPDATE`, não em DDL; por isso a ordem importa):

```sql
ALTER TABLE documentos ADD COLUMN assinatura_padrao text DEFAULT 'simples';  -- DDL: preenche todas as linhas sem disparar o trigger
ALTER TABLE documentos ALTER COLUMN assinatura_padrao DROP DEFAULT;
UPDATE documentos SET assinatura_padrao = NULL WHERE status <> 'assinado';  -- o trigger permite: linhas não assinadas
ALTER TABLE documentos ADD COLUMN assinatura_id uuid;                        -- módulo assinatura; sem FK entre módulos
ALTER TABLE documentos DROP CONSTRAINT doc_status, ADD CONSTRAINT doc_status CHECK (status IN ('rascunho','revisado','aguardando_assinatura','assinado'));
ALTER TABLE documentos ADD CONSTRAINT doc_assinatura_padrao CHECK ((status = 'assinado') = (assinatura_padrao IS NOT NULL) AND assinatura_padrao IN ('simples','icp'));
ALTER TABLE documentos ADD CONSTRAINT doc_icp_tem_assinatura CHECK (assinatura_padrao IS DISTINCT FROM 'icp' OR assinatura_id IS NOT NULL);
```

O trigger passa a rejeitar também, em `aguardando_assinatura`, qualquer mudança que não seja a transição de `status` (para `assinado` ou de volta para `revisado`) e as colunas de assinatura: o PDF assinado tem que corresponder ao conteúdo congelado. Atendimentos fechados antes desta RFC aparecem como "Assinatura eletrônica simples" (derivado: fechado e sem linha em `assinaturas`).

### Máquina de estados

Documento (`documentos.status`):

```
rascunho ──(salvar revisado)──▶ revisado
rascunho | revisado ──(assinatura simples, unidade sem exigir_icp)──▶ assinado [simples]
rascunho | revisado ──(entra em solicitação ICP; conteúdo congelado)──▶ aguardando_assinatura
aguardando_assinatura ──(item assinado)──▶ assinado [icp]
aguardando_assinatura ──(cancelar, ou solicitação expirada/falhou e o médico pede "editar")──▶ revisado
```

Evolução e adendo não têm coluna de estado (tabelas append-only); o estado é derivado: `nao_assinada` → `aguardando_assinatura` (item ativo em solicitação) → `assinada_icp` (linha em `assinaturas`). Atendimento fechado sem item ativo nem assinatura = `assinada_simples` (login + MFA, ADR-PEP-0011).

Solicitação (`solicitacoes_assinatura.status`):

```
aguardando_autorizacao ──(callback OAuth ok, ou já havia sessão válida)──▶ assinando
aguardando_autorizacao ──(10 min sem resposta)──▶ expirada
aguardando_autorizacao | assinando ──(médico cancela)──▶ cancelada
assinando ──(todos os itens com resultado final: assinado ou falhou)──▶ concluida
assinando ──(PSC/ACT indisponível após 3 tentativas com backoff)──▶ falhou
```

Itens de solicitação `expirada`, `cancelada` ou `falhou` voltam a ficar pendentes; "Tentar de novo" cria uma solicitação nova com os mesmos alvos (o PDF é regerado se o documento tiver voltado a `revisado`, senão reaproveitado).

### Modelo de dados (módulo `assinatura`)

Todas as tabelas com `tenant_id uuid NOT NULL` e RLS forçada por convenção em `drizzle/rls` (ADR-PEP-0019); acesso só por `Db.comTenant`. Referências a outros módulos são `uuid` sem FK (ADR-PEP-0017).

```sql
CREATE TABLE certificados_profissionais (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  usuario_id uuid NOT NULL,                  -- titular (módulo identidade)
  tipo text NOT NULL CHECK (tipo IN ('nuvem','a1')),
  psc text,                                  -- id do adaptador do PSC; NULL para A1
  titular_nome text NOT NULL,                -- do certificado (dado profissional)
  titular_cpf_hash bytea NOT NULL,           -- HMAC por tenant do CPF do certificado: confere renovação sem guardar o CPF
  emissor text NOT NULL,                     -- AC emissora
  serial text NOT NULL,
  impressao_digital bytea NOT NULL,          -- SHA-256 do certificado
  cadeia_pem text NOT NULL,                  -- certificados públicos (não é segredo)
  valido_de timestamptz NOT NULL, valido_ate timestamptz NOT NULL,
  sessao_psc_cifrada bytea,                  -- token de sessão de assinatura do PSC, cifrado com DEK do tenant (RFC-PEP-014)
  sessao_expira_em timestamptz,
  a1_pfx_cifrado bytea,                      -- só tipo a1 (fatia 6, Q5); a senha nunca é gravada
  ativo boolean NOT NULL DEFAULT true,
  ultimo_uso_em timestamptz,
  criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (tenant_id, usuario_id, impressao_digital),
  CHECK ((tipo = 'nuvem') = (psc IS NOT NULL)),
  CHECK (tipo = 'a1' OR a1_pfx_cifrado IS NULL)
);
CREATE INDEX ix_cert_usuario ON certificados_profissionais (usuario_id) WHERE ativo;
CREATE INDEX ix_cert_vencimento ON certificados_profissionais (valido_ate) WHERE ativo;

CREATE TABLE solicitacoes_assinatura (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  atendimento_id uuid NOT NULL,              -- uma solicitação = itens de um atendimento
  signatario_id uuid NOT NULL,
  certificado_id uuid NOT NULL REFERENCES certificados_profissionais(id),
  status text NOT NULL CHECK (status IN ('aguardando_autorizacao','assinando','concluida','falhou','expirada','cancelada')),
  oauth_state_hash bytea,                    -- state + PKCE do fluxo de autorização (só o hash)
  autorizacao_expira_em timestamptz,
  psc_transacao text,                        -- id opaco do PSC, para suporte
  tentativas smallint NOT NULL DEFAULT 0,
  erro_codigo text,                          -- código interno (psc_timeout, psc_recusado, act_indisponivel, cert_revogado…); nunca mensagem com conteúdo
  criado_em timestamptz NOT NULL DEFAULT now(), concluida_em timestamptz
);
CREATE INDEX ix_solic_atendimento ON solicitacoes_assinatura (atendimento_id, criado_em DESC);
CREATE INDEX ix_solic_pendentes ON solicitacoes_assinatura (signatario_id) WHERE status IN ('aguardando_autorizacao','assinando','falhou');

CREATE TABLE solicitacao_itens (
  tenant_id uuid NOT NULL,
  solicitacao_id uuid NOT NULL REFERENCES solicitacoes_assinatura(id),
  ordem smallint NOT NULL,
  alvo_tipo text NOT NULL CHECK (alvo_tipo IN ('evolucao_versao','adendo','documento','pedido_exame')),
  alvo_id uuid NOT NULL,
  hash_conteudo bytea NOT NULL,              -- hash do conteúdo canônico do alvo (evolucao_versoes.hash etc.)
  pdf_preparado_objeto text NOT NULL,        -- PDF com placeholder de assinatura (GCS)
  status text NOT NULL CHECK (status IN ('pendente','assinado','falhou','cancelado')),
  assinatura_id uuid,
  PRIMARY KEY (solicitacao_id, ordem)
);
-- Um alvo só pode estar em uma solicitação ativa por vez:
CREATE UNIQUE INDEX ux_item_alvo_ativo ON solicitacao_itens (alvo_tipo, alvo_id) WHERE status = 'pendente';

-- append-only: trigger rejeita UPDATE/DELETE (mesmo padrão de evolucao_versoes), REVOKE no app_rw
CREATE TABLE assinaturas (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  atendimento_id uuid NOT NULL,
  alvo_tipo text NOT NULL, alvo_id uuid NOT NULL,
  solicitacao_id uuid NOT NULL,
  signatario_id uuid NOT NULL, certificado_id uuid NOT NULL,
  certificado_serial text NOT NULL, certificado_emissor text NOT NULL,
  padrao text NOT NULL DEFAULT 'pades',
  politica text NOT NULL,                    -- 'AD-RT' | 'AD-RB' [validar OIDs das políticas ICP-Brasil]
  hash_conteudo bytea NOT NULL,
  pdf_assinado_objeto text NOT NULL,         -- GCS, bucket com retenção bloqueada (prazo do prontuário)
  pdf_assinado_sha256 bytea NOT NULL,
  carimbo_tempo_em timestamptz,              -- do token RFC 3161 da ACT; NULL só se a política for AD-RB
  trilha_anexada boolean NOT NULL,           -- apêndice no PDF (true) ou só hash (false)
  trilha_sha256 bytea,                       -- hash da trilha de revisão (evolução)
  trilha_resumo jsonb,                       -- {"aceitos":8,"editados":1,"descartados":1} para a UI, sem texto
  codigo_verificacao text NOT NULL UNIQUE,   -- 26 caracteres base32 (128 bits aleatórios)
  verificacao_inicial jsonb NOT NULL,        -- resultado da validação local logo após assinar (cadeia, política, revogação)
  assinado_em timestamptz NOT NULL,          -- = carimbo do tempo quando houver
  UNIQUE (alvo_tipo, alvo_id)                -- uma assinatura ICP por alvo; correção = adendo
);
CREATE INDEX ix_assin_atendimento ON assinaturas (atendimento_id);

CREATE TABLE config_assinatura_unidade (
  unidade_id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  exigir_icp boolean NOT NULL DEFAULT false,  -- true: rota de assinatura simples responde 409
  assinar_evolucao text NOT NULL DEFAULT 'sempre' CHECK (assinar_evolucao IN ('sempre','com_documento_ou_convenio'))  -- Q6 (custo por assinatura)
);
```

`assinaturas` é a única tabela consultada sem tenant na sessão (página pública, pelo `codigo_verificacao`): a busca passa por uma função `verificar_assinatura(codigo text)` `SECURITY DEFINER`, dona `app_owner`, que devolve só os campos públicos da verificação (abaixo, com nome e conselho do signatário de `usuarios`) e nada mais; o `app_rw` só tem `EXECUTE` nela. Atualizar o [[modelo-de-dados-pep|modelo de dados]] ao implementar (hoje lista `assinaturas` com `documento_id` ou `evolucao_versao_id`; aqui virou `alvo_tipo`/`alvo_id`).

### Porta e adaptadores

```ts
// apps/api/src/modulos/assinatura/servico-assinatura.ts
interface ServicoAssinatura {
  urlAutorizacao(cert: CertificadoRef, state: string, codeChallenge: string): string   // OAuth do PSC (nuvem)
  trocarCodigo(code: string, codeVerifier: string): Promise<SessaoPsc>                  // token de sessão + certificado
  assinarHashes(sessao: SessaoPsc, hashes: Buffer[]): Promise<Buffer[]>                // PKCS#1 v1.5 / SHA-256 sobre signedAttrs
}
```

| Adaptador | Uso |
| :--- | :--- |
| `psc-<fornecedor>` | Produção, um por PSC contratado (Q1). Só hashes saem do PEP |
| `a1` | Fatia 6 (Q5): decifra o PFX em memória no worker, com a senha digitada na própria solicitação |
| `simulado` | Dev, CI e demo: AC de teste própria e ACT simulada. Nunca habilitado em `prod` (a API recusa subir com ele) |

A montagem do PAdES fica no PEP, igual para todos os adaptadores: o PDF (gerado pela RFC-PEP-004, PDF/A-2b **[validar exigência de PDF/A]**, com a página de assinatura visível: nome, CRM/UF, data, código de verificação e QR) recebe um placeholder; o PEP monta o CMS `SignedData` com os atributos assinados da política ICP-Brasil (`signing-certificate-v2`, `signature-policy-identifier` **[validar OIDs e versão das políticas no DOC-ICP-15]**), envia ao PSC só o hash dos atributos, pede o carimbo do tempo (RFC 3161) à ACT sobre a assinatura, embute o CMS e valida o resultado (cadeia até a raiz ICP-Brasil, política, revogação por LCR/OCSP) antes de gravar. Bibliotecas: `pkijs` (CMS) + `@signpdf/*` (placeholder e embutir), JS puro no worker, sujeito à fatia 0.

### Fluxo

1. **Pedido.** `POST /api/v1/assinaturas/solicitacoes` com os alvos de um atendimento (ou automático no "Assinar e finalizar", passo 6). O serviço valida, na mesma transação: signatário é o profissional do atendimento (ou autor do adendo); atendimento `fechado` para evolução; nenhum trecho de IA pendente (regra RF-ASS-02, abaixo); certificado ativo, dentro da validade; alvo sem assinatura ICP nem item ativo. Documentos vão para `aguardando_assinatura`.
2. **PDF.** Para cada alvo, gera o PDF (evolução: última versão + apêndice da trilha, ou o hash da trilha se `anexar_trilha = false`), grava no GCS e calcula `hash_conteudo`. Síncrono: são poucos itens por atendimento (meta ≤ 1,5 s para 5 itens).
3. **Autorização.** Sessão PSC válida no certificado → `assinando` e enfileira o job. Sem sessão → `aguardando_autorizacao` e devolve `autorizacao_url` (OAuth com PKCE e `state`); a UI abre a página/app do PSC e consulta a solicitação a cada 2 s. O PSC redireciona para a rota da UI `/assinatura/retorno`, que repassa `code` e `state` a `POST /api/v1/assinaturas/psc/retorno` com a sessão do PEP; a API troca o código, grava a sessão cifrada com a validade concedida pelo PSC **[validar duração máxima da sessão, Q2]** e enfileira o job.
4. **Execução.** Job BullMQ `assinatura-executar` (chave = id da solicitação): para cada item pendente, monta o CMS, `assinarHashes` (em um lote, se o PSC aceitar), carimbo do tempo, embute, valida e, em uma transação por item: grava `assinaturas`, marca o item `assinado`, leva o documento a `assinado` (`assinatura_padrao = 'icp'`, `assinatura_id`) e grava `DocumentoAssinado` no outbox. A `UNIQUE (alvo_tipo, alvo_id)` torna a reentrega idempotente. Falha de rede: 3 tentativas com backoff (5 s, 30 s, 2 min), depois `falhou`.
5. **Falha.** Nada se perde: o atendimento continua fechado, a guia segue para o faturista, o documento fica "Aguardando assinatura" e aparece na pendência da tela 02 do signatário. Assinatura nunca bloqueia faturamento (princípio 5).
6. **"Assinar e finalizar" (tela 07) e "Finalizar" (Ctrl+Enter, tela 05).** Primeiro fecha o atendimento exatamente como hoje (grava a versão, `AtendimentoFechado`, guia em rascunho se marcado); só depois, fora dessa transação, cria a solicitação da evolução (e dos documentos `revisado`, se houver). A resposta do `finalizar` ganha `solicitacaoAssinaturaId` (ou `null` sem certificado ou com `assinar_evolucao` não aplicável).

### API (módulo `assinatura`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/certificados` | Certificados do usuário logado: tipo, PSC, titular, emissor, `validoAte`, `sessaoAtiva` (nunca o token) |
| `POST` | `/api/v1/certificados/conectar` | `{ "psc": "psc_x" }` → `{ "autorizacaoUrl": "…" }`. No retorno, o PEP lê o certificado, mostra titular e validade e grava `certificados_profissionais` |
| `POST` | `/api/v1/certificados/a1` | Fatia 6 (Q5): upload do PFX (multipart) + senha só para validar; grava o PFX cifrado |
| `DELETE` | `/api/v1/certificados/{id}` | `ativo = false` e descarta a sessão (nunca apaga a linha: assinaturas antigas apontam para ela) |
| `POST` | `/api/v1/assinaturas/psc/retorno` | `{ "code": "…", "state": "…" }`, chamado pela página `/assinatura/retorno` da UI (o redirect do PSC não leva o token do PEP); exige sessão do PEP e `state` igual a `oauth_state_hash`. Serve à conexão do certificado e à autorização de solicitação |
| `POST` | `/api/v1/assinaturas/solicitacoes` | `{ "atendimentoId": "…", "itens": [{ "alvoTipo": "evolucao_versao", "alvoId": "…" }, { "alvoTipo": "documento", "alvoId": "…" }], "anexarTrilha": true }` → `201 { "id": "…", "status": "aguardando_autorizacao", "autorizacaoUrl": "…" }`. `409` com trecho de IA pendente ("Decida os 2 trechos pendentes para liberar a assinatura") ou alvo já assinado/em solicitação; `422` sem certificado ativo ou vencido |
| `GET` | `/api/v1/assinaturas/solicitacoes/{id}` | Status, itens e `erroCodigo` (a UI consulta a cada 2 s até status final) |
| `POST` | `/api/v1/assinaturas/solicitacoes/{id}/cancelar` | Itens voltam a pendentes; documentos voltam a `revisado` |
| `POST` | `/api/v1/assinaturas/solicitacoes/{id}/repetir` | Nova solicitação com os itens não assinados |
| `GET` | `/api/v1/atendimentos/{id}/assinaturas` | Estado de assinatura por alvo do atendimento (telas 07 e 08) |
| `GET` | `/api/v1/assinaturas/{id}/pdf` | Download do PDF assinado (gera evento de auditoria, como leitura de prontuário) |
| `GET` | `/api/v1/assinaturas/pendencias` | Itens do usuário em `aguardando_assinatura`/falha (tela 02) |
| `GET` | `/api/v1/integracoes/assinatura` | Tela 25: tipos em uso, total de certificados ativos da unidade, quantos vencem em 30 dias (`admin:configurar`) |
| `PUT` | `/api/v1/unidades/atual/assinatura` | `{ "exigirIcp": true, "assinarEvolucao": "sempre" }` (`admin:configurar`, auditado) |
| `GET` | `/v/{codigo}` (público) | Página de verificação; `GET /api/v1/verificacao/{codigo}` devolve `{ "valida": true, "tipoDocumento": "receita", "signatario": "Dra. Exemplo A", "conselho": "CRM/UF", "assinadoEm": "…", "politica": "AD-RT", "sha256": "…" }`. Sem paciente, sem conteúdo. O visitante pode arrastar o PDF que tem em mãos: o hash é calculado no navegador e comparado com `sha256`; o arquivo não sai do dispositivo. Rate limit por IP |

Permissões: criar/acompanhar solicitação e gerir o próprio certificado exigem `prontuario:escrever` (e a regra "signatário = profissional do atendimento"); tela 25 e configuração exigem `admin:configurar`. Nenhuma permissão nova. A rota existente `POST /api/v1/atendimentos/{id}/documentos/assinatura` continua sendo a assinatura simples; com `exigir_icp` ela responde `409` e a UI usa a de solicitações.

### Eventos

- **Publica** `DocumentoAssinado` (canônico, já emitido hoje pela assinatura simples), payload ampliado e só com ids: `{ assinaturaId, alvoTipo, alvoId, atendimentoId, tipoDocumento, padrao: "simples" | "icp" }`. Na assinatura simples `assinaturaId` é `null`. Consumidores: `notificacoes` (envio ao paciente, RFC-PEP-004), `retorno` (evidência para recurso, RFC-PEP-007), `auditoria`.
- **Não publica** `GuiaAssinada` (reservado para XML assinado, fora do escopo) nem evento de falha: pendências e vencimentos são lidos por consulta (tela 02) e o job diário `certificados-vencimento` cria a notificação pelo serviço do módulo `notificacoes`.
- **Não consome** eventos: a solicitação nasce de ação do usuário, que precisa do id na hora para autorizar.

### Regras

- **RF-ASS-02, no servidor:** a solicitação é recusada se existir, no atendimento, `trechos_ia.status = 'pendente'` (RFC-PEP-013), ou se o alvo for documento cujo conteúdo traz texto sugerido por IA ainda sem decisão (resumo ao paciente não `revisado`, indicação clínica sugerida e não aceita). Decisões pendentes do painel anti-glosa (RFC-PEP-005) são sobre a **guia** e não bloqueiam a assinatura da evolução; bloqueiam só o que vai à guia.
- Só o profissional responsável assina evolução e documentos do atendimento; adendo, só o autor. Ninguém assina por outro (sem delegação na P2).
- Receita de controle especial e antimicrobiano só existem com ICP: o tipo só é selecionável com certificado ativo e a solicitação é recusada (`422`) se o signatário não tiver ICP **[validar requisitos ANVISA/CFM de receita digital]**.
- Um alvo tem no máximo uma assinatura ICP; correção é adendo, também assinado.
- Conteúdo congelado: documento em `aguardando_assinatura` não aceita `PUT` (409) e o trigger rejeita mudança de conteúdo.
- Certificado vencido, revogado (LCR/OCSP no momento da assinatura) ou de cadeia fora da ICP-Brasil: `422` ou item `falhou` com `cert_revogado`.
- ACT indisponível: o item falha e tenta de novo; não há rebaixamento silencioso para AD-RB (Q4 decide se isso muda).
- `assinado_em` vem do carimbo do tempo, não do relógio do servidor.
- Retenção: PDFs assinados no bucket com retenção bloqueada pelo prazo do prontuário **[validar 20 anos, Lei nº 13.787/2018]**; nunca apagados por exclusão de titular (ADR-PEP-0007).
- Unidade com `assinar_evolucao = 'com_documento_ou_convenio'`: atendimento particular sem documento não cria solicitação da evolução (fica com assinatura simples). Padrão `sempre` até a Q6.

## UI

Rotas canônicas da RFC-PEP-015: `/atendimentos/:id/revisao` (07), `/atendimentos/:id/documentos` (08), `/configuracoes/*` (25). Hoje as telas reais estão em `/atendimento/:id/revisao` e `/atendimento/:id/documentos`.

**Tela 07** (`apps/ui/src/pages/Revisao.tsx`, design C07):

| Bloco | Fonte | Estado |
| :--- | :--- | :--- |
| Contadores de trechos, barra, atalhos J/K/A/E/D, Aceitar/Descartar tudo, aviso de termos acrescentados | `trechos_ia` (RFC-PEP-013) | Real (já entregue) |
| Cartão do certificado ("Certificado ICP-Brasil A3 · Dra. Exemplo A · CRM ••••• · válido até 08/2027") | `GET /certificados` | Real nesta RFC. Sem certificado: "Assinatura eletrônica simples" + ação "Conectar certificado ICP-Brasil" |
| "Gerar guia SP/SADT em rascunho…" | `finalizar.gerarGuia` | Real (já entregue) |
| "Anexar trilha de revisão…" | `anexarTrilha` → apêndice ou hash no PDF | Real nesta RFC |
| "Decida os N trechos pendentes para liberar a assinatura" | contagem local + 409 do servidor | Real |
| "Assinar e finalizar" | `finalizar` + solicitação | Real. Depois do clique: "Aguardando autorização no app do certificado…" com "Abrir autorização" e "Cancelar"; sucesso: "Atendimento assinado com ICP-Brasil"; falha: "Atendimento finalizado. Assinatura ICP pendente" + "Tentar de novo" (não volta a travar nada) |
| "Após assinar, o documento fica imutável. Correções geram adendo versionado." | texto fixo | Real |

**Tela 08** (`apps/ui/src/pages/Documentos.tsx`, design C08):

| Bloco | Fonte | Estado |
| :--- | :--- | :--- |
| Selo do cartão | `documentos.status` + `assinatura_padrao` | "Assinada 09:58 · ICP-Brasil", "Assinada 09:58 · simples", "Aguardando assinatura" (âmbar) |
| "Assinar pendentes (N)" | uma solicitação com todos os pendentes (e a evolução, se pendente) | Real nesta RFC |
| Receita: "Controle especial", "Antimicrobiano" | certificado ativo | Habilitados com ICP; sem certificado continuam inertes com `title` "Exige certificado ICP-Brasil (RFC-PEP-011)" |
| "Imprimir" / anexos do resumo ("receita (PDF)") | `GET /assinaturas/{id}/pdf` | Depende do PDF (RF-DOC-04, RFC-PEP-004); até lá, Prévia |
| "Enviar ao paciente" / WhatsApp | RFC-PEP-004 + consentimento | Fora desta RFC (continua Prévia) |

**Tela 25** (`apps/ui/src/modulos/plataforma/Plataforma.tsx`, rota `/configuracoes/*`): só o cartão "Assinatura ICP-Brasil" (selo "A1 e A3" a partir dos tipos em uso; "6 certificados · 1 vence em 30 dias") vem de `GET /integracoes/assinatura`, mais o interruptor "Exigir ICP-Brasil nesta unidade". O resto da tela é da RFC-PEP-014.

**Tela 02:** item de pendência "N documentos aguardando assinatura ICP" e "Seu certificado vence em N dias" (componente de notificações da RFC-PEP-015).

**Página pública** `/v/{codigo}`: fora do shell do PEP, sem login; mostra validade, tipo, signatário, data e a área "Confira o arquivo" (hash no navegador).

## Segurança e LGPD

- Ao PSC só vão hashes dos atributos assinados; nunca o PDF, texto clínico ou dado do paciente. Teste do adaptador confere o corpo das requisições (mesmo padrão do teste do SpikeAdapter).
- Tokens de sessão do PSC e PFX de A1 cifrados com DEK do tenant (envelope com Cloud KMS, RFC-PEP-014); decifrados só no worker, em memória, durante a operação. Senha de A1 nunca persistida nem logada.
- Logs com ids, `erro_codigo` e latência; nunca hash do conteúdo junto com id de paciente, nunca token, nunca corpo de requisição.
- Auditoria (`auditoria_eventos`): `certificado.conectado`, `certificado.removido`, `assinatura.solicitada`, `assinatura.concluida`, `assinatura.falhou`, `assinatura.pdf_baixado`, `assinatura.config_alterada`.
- Página pública: código de 128 bits, rate limit, resposta sem PII do paciente; nome e conselho do signatário são os mesmos impressos no documento.
- PDFs assinados com CMEK e retenção bloqueada (ADR-PEP-0009); download só por quem lê o prontuário daquele atendimento, com registro "quem viu".
- O PSC é suboperador (identidade do signatário); entra na lista pública de suboperadores e no RIPD **[validar com jurídico]**.

## Critérios de aceite

- [ ] Solicitação com trecho de IA pendente → `409` com a contagem; depois de decidir tudo → `201`. Mesmo comportamento chamando a API direto (sem a UI).
- [ ] Documento em `aguardando_assinatura`: `PUT` do conteúdo → `409`; `UPDATE` direto no banco do `conteudo` falha no trigger, inclusive para o owner.
- [ ] `UPDATE`/`DELETE` em `assinaturas` falha (trigger + `REVOKE`).
- [ ] Migração aplicada sobre o seed da demo: documentos já assinados ficam `assinatura_padrao = 'simples'` sem erro do trigger; não assinados ficam `NULL`.
- [ ] Com o adaptador `simulado`, o PDF assinado valida em teste automatizado (cadeia da AC de teste, política, carimbo, ByteRange cobrindo o arquivo todo). Na homologação com o PSC real, o PDF valida no verificador oficial do ITI **[validar ferramenta]** (evidência anexada ao PR).
- [ ] Teste do adaptador: nenhuma requisição ao PSC ou à ACT contém bytes do PDF, texto da evolução ou nome do paciente.
- [ ] PSC fora do ar (timeout simulado): "Assinar e finalizar" fecha o atendimento (`200`), cria a guia, a solicitação termina `falhou` e a pendência aparece na tela 02; "Tentar de novo" assina sem regerar o conteúdo.
- [ ] Reentrega do job `assinatura-executar` não cria segunda assinatura nem segundo `DocumentoAssinado`.
- [ ] Receita de controle especial sem certificado ativo → `422`; com certificado → assinada ICP.
- [ ] Unidade com `exigir_icp`: rota de assinatura simples → `409`.
- [ ] Verificação pública: código válido devolve `valida: true` e os metadados; resposta não contém nome do paciente nem conteúdo (snapshot); código inexistente → `404`; 30 requisições/min por IP → `429`.
- [ ] RLS: usuário do tenant A não lê certificado, solicitação nem assinatura do tenant B (teste com dois tenants).
- [ ] Certificado com `valido_ate` em 29 dias aparece no aviso; vencido → `422` ao solicitar.
- [ ] Telas 07 e 08 e o cartão da tela 25 ligados à API, com E2E do caminho feliz no adaptador `simulado`.

## Riscos

- Dependência de PSC externo (latência, indisponibilidade) no momento de fechar a consulta: mitigada por fechar antes e assinar depois.
- Custo por assinatura do PSC e da ACT; volume alto em receitas. Medir no piloto; a Q6 reduz a evolução assinada se o custo pesar.
- Montar PAdES ICP-Brasil em Node é menos comum que em Java: a fatia 0 prova a biblioteca antes de qualquer outra entrega.
- Requisitos de prontuário sem papel (SBIS NGS2) podem mudar o nível de assinatura exigido **[validar]**; alinhar com a RFC-PEP-016.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | Qual PSC (preço por assinatura e por médico, API, SLA, suporte a OAuth com PKCE) e quantos médicos do piloto já têm certificado em nuvem? | Pedro + piloto | Fatia 2 |
| Q2 | O PSC oferece sessão de assinatura (várias assinaturas com uma autorização) e por quanto tempo? Se não, cada item pede autorização | Pedro (com o PSC) | Fatia 2 (só UX) |
| Q3 | Receita: integrar plataforma de prescrição (validação em farmácia) ou só o PDF assinado próprio? | Produto **[validar]** | Fatia 4 (controle especial) |
| Q4 | Política por tipo de documento (AD-RB × AD-RT), carimbo do tempo obrigatório e qual ACT | Jurídico + CMO **[validar]** | Fatia 2 (o padrão AD-RT segue até a resposta) |
| Q5 | A1 do profissional guardado cifrado no servidor é aceitável jurídica e operacionalmente? | Jurídico + segurança | Só a fatia 6 |
| Q6 | Assinar a evolução de todo atendimento ou só com documento/convênio (custo)? | Produto + CMO | Não (configurável; padrão `sempre`) |
| Q7 | A página pública de verificação basta para a farmácia ou é preciso integrar o validador de prescrição do ITI/CFM **[validar]**? | Produto | Fatia 4 |

## Fatias de entrega

| # | Entrega | Pronto quando |
| :--- | :--- | :--- |
| 0 | Prova técnica (2–3 dias): PAdES com `pkijs` + `@signpdf` sobre AC de teste, carimbo RFC 3161 simulado, validação | PDF de exemplo valida no teste automatizado e em um validador externo; se não validar, decidir a biblioteca antes da fatia 1 |
| 1 | Tabelas, migração de `documentos`, porta + adaptador `simulado`, API de solicitações, job, `DocumentoAssinado` ampliado; tela 08 com selos e "Assinar pendentes" | Critérios de RLS, imutabilidade, migração e reentrega verdes no `simulado` |
| 2 | Adaptador do PSC escolhido (Q1), conexão do certificado (OAuth), ACT real; ADR-PEP-0011 aceita | PDF assinado no PSC de homologação valida no verificador do ITI |
| 3 | Evolução e adendo: PDF da evolução com trilha, "Assinar e finalizar" (07) e "Finalizar" (05) criam a solicitação; pendências na 02 | Teste de PSC fora do ar e E2E da tela 07 verdes |
| 4 | Receita de controle especial e antimicrobiano; página pública `/v/{codigo}` e QR | Critérios de receita e verificação verdes; Q3/Q7 respondidas |
| 5 | Tela 25 (cartão e "Exigir ICP"), aviso de vencimento | Cartão com dados reais; `exigir_icp` testado |
| 6 | A1 cifrado (só se a Q5 aprovar) | PFX nunca aparece em claro fora da memória do worker (teste) |

## Notas de implementação (22/11/2026 — PR `pedro2s/tissia#40`)

- Migração `0057_onda_e2_f1_assinatura_icp.sql`: persistência server-side de `hash_conteudo_sha256`, `codigo_verificacao` (`VER-XXXX-XXXX-XXXX`), `nivel_assinatura` (`simples` | `icp_brasil`), `provedor_assinatura` e resumo do certificado.
- `AssinaturaService` + `ProvedorAssinaturaPsc` (PSC em nuvem Vidaas/BirdID + sandbox determinístico com step-up PIN/OTP), rota pública sem PII `GET /api/v1/publico/documentos/verificar/:codigo`, QR Code SVG inline (`QrCodeVerificacao.tsx`) e desbloqueio de antimicrobianos/controle especial com assinatura qualificada.
