---
projeto: PEP
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-014-plataforma-lgpd-auditoria.md
---

# RFC-PEP-014 — Plataforma: perfis, auditoria, LGPD e integrações

**Fase:** P1 (núcleo) · P3 (API pública, webhooks, marketplace) · **Status:** Em implementação (núcleo P1 — Ondas 1 e 6 entregues em 09/11/2026 e 14/11/2026; pendente particionamento mensal e âncora em bucket, ver Notas de implementação) · Rascunho (P3)
**ADRs:** ADR-PEP-0003, ADR-PEP-0005, ADR-PEP-0007, ADR-PEP-0008, ADR-PEP-0009, ADR-PEP-0010
**Telas:** 25 (Perfis, auditoria, LGPD e integrações); 04 (revelar dado mascarado)
**Requisitos:** RF-PLT-*, RNF de segurança e LGPD do PRD-001
**Depende de:** RFC-PEP-000 (tenancy, RLS, infra), RFC-PEP-001 (identidade e MFA)

## Objetivo

Dar ao PEP, desde a P1, o que é transversal e caro de adicionar depois: autorização por perfil, trilha de auditoria imutável com "quem viu", consentimentos, direitos do titular, retenção e chaves por tenant. A API pública e o marketplace ficam para a P3.

## Parte 1 — Núcleo (P1)

### 1.1 Perfis e permissões

- Perfis padrão: **Médico, Recepção, Faturista, Gestor, Administrador**; perfis customizados por clínica copiando um padrão.
- Permissões com escopo, como na tela 25: `prontuario.ver` (`proprios | todos | resumo | nao`), `evolucao.editar_assinar`, `agenda` (`propria | todas | ver`), `guias.editar` (`revisar | sim | ver | nao`), `lotes.gerar_enviar`, `financeiro` (`proprio | ver | sim`), `dado_mascarado.revelar` (`com_motivo | nao`).
- Avaliação no backend (guard NestJS por rota + filtro de escopo na query); a UI só esconde. RLS garante o tenant (ADR-PEP-0003); escopo "próprios" é filtro de aplicação por `profissional_id`.
- Administrador exige MFA com chave física (WebAuthn, ADR-PEP-0010).
- Faturista: o backend acessa a evolução para mandar à Spike (`evolucao_clinica`), mas o perfil vê só o resumo necessário à guia (Spike `validacao_tissia_ai.md` Q2).

### 1.2 Trilha de auditoria imutável

- Tabela `auditoria_eventos` append-only (sem UPDATE/DELETE por grant + trigger), particionada por mês: `id`, `tenant_id`, `ator` (usuário, sistema, integração), `acao`, `recurso_tipo`, `recurso_id`, `paciente_id` (quando houver), `motivo`, `ip`, `user_agent`, `ocorrido_em`, `hash_anterior`, `hash`.
- **Encadeamento por hash sem contenção de leitura:** a gravação do evento na requisição é um `INSERT` síncrono append-only sem lock serial (`hash` inicialmente `NULL`), garantindo latência < 5 ms em leituras simultâneas de prontuário; um worker sela os blocos pendentes por tenant a cada minuto (`hash = SHA-256(hash_anterior ‖ evento canônico)`), e um job diário verifica toda a cadeia ("Integridade verificada" na tela 25) gravando âncora diária do último hash fora do banco (Object Storage com retenção bloqueada) para detectar reescrita total.
- **"Quem viu":** toda leitura de prontuário, documento ou dado mascarado gera evento (`prontuario.visualizado`, `cpf.revelado` com motivo obrigatório). Gravado por interceptor no backend, na mesma requisição; falha na gravação da auditoria = falha da leitura (`503`, não se lê prontuário sem rastro).
- Eventos de IA: análise recebida, sugestão exibida, aceita/editada/descartada (agregado por sessão, como "Aceitou 9, editou 1, descartou 1"), com autoria humano × IA.
- Nunca conteúdo clínico no evento: só ids, ação e motivo.
- Consulta por paciente, usuário e período; exportação assinada (CSV + hash).

### 1.3 Consentimentos

- `Consentimento` (paciente, finalidade, termo/versão, canal, decisão, data, revogado_em). Finalidades na P1: comunicação por WhatsApp (opt-in no cadastro), pesquisa com dados anonimizados (opcional, revogável). Escuta ambiental com IA entra com a RFC-PEP-013 (termo pedido antes de cada gravação).
- Tratamento para tutela da saúde e para a Spike não depende de consentimento (base legal LGPD art. 11, II, f **[validar com jurídico]**), mas a pseudonimização é sempre ativa (ADR-PEP-0005) e mostrada como "Ativa" na tela 25.

### 1.4 Direitos do titular e encarregado

- Cadastro do encarregado (DPO) por clínica; fila de solicitações (acesso, correção, portabilidade, informação de compartilhamento) com prazo **[validar prazos LGPD/ANPD]**.
- Relatório de acesso do titular gerado da trilha (quem viu, quando).
- Eliminação: prontuário tem guarda legal mínima **[validar: Lei 13.787/2018 — 20 anos]**; pedido de eliminação de prontuário vira bloqueio de uso secundário, não exclusão.

### 1.5 Retenção

| Dado | Retenção | Mecanismo |
| :--- | :--- | :--- |
| Prontuário e documentos assinados | Prazo legal **[validar]** | Sem exclusão; versionado |
| Trilha de auditoria | ≥ prazo do prontuário | Partições antigas em armazenamento frio |
| Áudio de escuta | Zero (descartado após transcrição) | RFC-PEP-013 |
| Arquivos XML TISS | Prazo fiscal/contratual **[validar]** | Object storage com lifecycle |
| Tabela de vínculo Spike (`paciente_ref`) | Enquanto o paciente existir | Cifrada (1.6) |

### 1.6 Chaves por tenant

- Envelope encryption: KEK por tenant no Cloud KMS (ADR-PEP-0009), DEKs para colunas sensíveis (CPF, CNS, carteirinha, telefone, sal do HMAC da Spike, certificados A1) cifradas na aplicação.
- Rotação anual da KEK; re-cifragem preguiçosa das DEKs. Exclusão do tenant = destruição da KEK (crypto-shredding) após o período de guarda.
- Mascaramento por padrão na UI (tela 04: `•••.•••.•••-41`); revelar = permissão + motivo + evento de auditoria.

### 1.7 RIPD

Relatório de Impacto produzido antes do piloto com dado real, cobrindo PEP e o compartilhamento pseudonimizado com a Spike (papéis: clínica controladora; PEP operadora; Spike suboperadora quando vendida separadamente **[validar com jurídico]**). Ver [[conformidade-regulatoria|conformidade]].

### 1.8 Painel de integrações (somente leitura na P1)

Status de Spike (ativa/degradada, `versao_bases`, latência média), webservice TISS por operadora, WhatsApp, certificados ICP (vencimento). Configuração da Spike por unidade (modo silencioso/assistido) fica na tela 15 (RFC-PEP-005).

### Critérios de aceite (núcleo)

- [ ] Toda rota com dado clínico tem guard de permissão; teste de matriz perfil × rota gerado a partir da tabela de permissões.
- [ ] Abrir prontuário sem conseguir gravar auditoria → 503, e nada é retornado.
- [ ] Verificador de cadeia detecta alteração de qualquer linha (teste que adultera um evento).
- [ ] Nenhum evento de auditoria nem log contém texto clínico ou PII (teste com PII sintética, como a Spike faz).
- [ ] Revelar CPF sem motivo → 422; com motivo → evento `cpf.revelado`.
- [ ] Usuário de um tenant não lê nada de outro, mesmo com bug na aplicação (teste de RLS com conexão sem `tenant_id` setado → zero linhas).

## Parte 2 — API pública, webhooks e marketplace (P3, Rascunho)

- **API pública:** OAuth2 client credentials, chaves por clínica com escopos (ex.: `agenda:leitura`, `bi:leitura`), rate limit, versionamento no path, documentação OpenAPI (tela 25: "pk_live_…", revogar).
- **Webhooks:** eventos públicos derivados dos de domínio (`guia.validada`, `lote.protocolado`, `glosa.recebida`), assinatura HMAC, retry com backoff, painel de falhas ("1 falhando").
- **Marketplace:** catálogo de integrações por clínica (laboratórios, imagem, farmácia, ERP). Ver RFC-PEP-016.
- Perguntas: a API expõe dado clínico (exige escopo e base legal por integração)? Cobrança por uso?

## Riscos

- Auditoria síncrona aumenta latência de leitura; medir (alvo < 10 ms por evento) e usar insert em lote só se necessário.
- Volume da trilha (cada leitura gera linha): particionamento desde o início.
- Permissões granulares demais viram suporte; manter tipos fechados.

## Perguntas abertas

| # | Pergunta | Dono |
| :--- | :--- | :--- |
| Q1 | Papéis LGPD e base legal do compartilhamento com a Spike | Jurídico |
| Q2 | Prazos de guarda de prontuário e XML | Jurídico **[validar]** |
| Q3 | Escopo "próprios" inclui pacientes atendidos por outro profissional da mesma equipe? | CMO |

## Notas de implementação

**Onda 1 do MVP (09/11/2026): núcleo mínimo da trilha antecipado com a RFC-PEP-002.**

- Tabela `auditoria_eventos` (módulo `auditoria`, RLS por tenant) append-only por trigger, inclusive para o admin; `AuditoriaService.registrar` grava na transação do chamador, então falha da auditoria falha a leitura. Ações em uso: `paciente.visualizado`, `paciente.revelado` (campos + motivo), `paciente.editado` (só nomes de campo). Nunca valores nem texto clínico.
- **Pendente (onda 6):** `hash_anterior`/`hash` ficam nulos até o selador por tenant e o verificador diário; sem `ip`/`user_agent`, sem particionamento mensal, sem consulta nem exportação (tela 25), sem "quem viu" do prontuário e dos documentos.

**Onda 6, fatia 2 (14/11/2026): selagem da trilha.**

- `sequencia`, `hash_anterior` e `hash` (`SHA-256(hash_anterior ‖ evento canônico)`, `selo.ts`) são preenchidos pelo worker (`selagem.ts`): `selarPendentes` a cada minuto, um lock consultivo por tenant, na ordem de gravação. A ordem da cadeia é a `sequencia` (ordem de selagem), não `ocorrido_em`: uma transação longa que confirma depois não entra no meio de eventos já selados. O evento canônico usa `ocorrido_em` em texto UTC com microssegundos (o `Date` do JS os perderia).
- O trigger de `auditoria_eventos` passou a permitir uma única mudança: selar uma linha ainda não selada, com todo o resto idêntico; selado é para sempre e DELETE segue bloqueado. `app_rw` perde UPDATE/DELETE por grant (`scripts/db-rls.ts`, que reafirma os grants depois das migrações); `app_admin` continua dependendo só do trigger.
- `verificarCadeia` confere buracos, encadeamento, hash recalculado e âncoras; `verificarTodos` (job diário do worker) sela, verifica, grava o resultado em `auditoria_verificacoes` (é o que a tela 25 vai mostrar como "Integridade verificada") e ancora o último hash.
- **Desvio:** a âncora vai para arquivo (`AUDITORIA_ANCORA_DIR`, `.ancoras-auditoria/`) atrás da interface `AncoraStore`; o Object Storage com retenção bloqueada entra no portão do piloto (ADR-PEP-0009). Teste: `apps/api/test/auditoria-selagem.spec.ts` (encadeia, selagem concorrente, trigger, conteúdo alterado, evento removido, cadeia refeita e truncada pegas pela âncora).
- **Pendente (fatia 3):** `ip`/`user_agent`, particionamento mensal, consulta e exportação, "quem viu" do prontuário e dos documentos.

**Onda 6, fatia 3 (14/11/2026): consulta, exportação e "quem viu".**

- `GET /api/v1/auditoria/eventos` (filtros paciente, usuário, ação e período; cursor opaco) devolve os eventos com o nome de quem agiu e a última verificação da cadeia (`integridade`); `GET /api/v1/auditoria/eventos/exportar` entrega o CSV (até 10.000 linhas) com o SHA-256 do conteúdo em `X-Conteudo-SHA256`, neutraliza fórmula de planilha e grava `auditoria.exportada`. Exigem `auditoria:ler`, que o perfil **admin** passou a ter (a RFC diz que o Administrador tem todas as permissões); DPO já tinha.
- `AuditoriaService.registrar` falhando → **503** e a operação inteira desfeita, sem corpo (teste: ator que não é uuid na leitura do paciente). `ip` e `user_agent` entram no evento (do `Contexto`) e no hash da selagem.
- "Quem viu": `documento.visualizado` ao listar os documentos do atendimento e do paciente, na mesma transação. Os demais já existiam (`prontuario.visualizado`, `paciente.visualizado`, `paciente.revelado`).
- Tela 25: o painel "Trilha de auditoria" vira real (`TrilhaAuditoria.tsx`: filtro por período, selo de integridade, "Carregar mais", Exportar) para quem tem `auditoria:ler`; sem a permissão segue o painel ilustrativo do design.
- **Desvios:** gravação por serviço (`registrar` nos pontos de leitura) e não por interceptor global: o interceptor não sabe qual recurso e qual paciente a rota leu. **Pendente:** particionamento mensal (recriar a tabela fere "migrações só aditivas"; entra com o portão do piloto, antes de haver volume), eventos de IA agregados por sessão e "quem viu" de outras leituras clínicas conforme as RFCs de cada domínio.
