---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-013-voz-ambiental.md
---

# RFC-PEP-013 — Voz ambiental e revisão pós-consulta

**Fase:** P2 (fatia antecipada no Marco D) · **Status:** Implementada (fatia do Marco D entregue; restante detalhado em 09/11/2026) · **ADRs:** ADR-PEP-0013, ADR-PEP-0005, ADR-PEP-0007, ADR-PEP-0008, ADR-PEP-0015; Spike ADR-0010
**Telas:** 06 (Captura de voz ambiental), 07 (Revisão pós-consulta), 26b (Mobile — captura de voz); painel Tissia IA da tela 05 (aba "Resumo")
**Requisitos:** RF-VOZ-01 a RF-VOZ-04; RF-ASS-02 (decisão em todo trecho de IA antes de assinar); RF-PLT-02 e RF-PLT-03 (trilha e consentimento); RF-PLT-04 (flag `modulos.voz`); RF-MOD-03 (modelos por especialidade)
**Depende de:** RFC-PEP-002 (`consentimentos` do cadastro, finalidade `escuta_ambiental`), RFC-PEP-004 (modelos por especialidade e evolução versionada), RFC-PEP-011 (assinatura ICP-Brasil, só referência), RFC-PEP-014 (auditoria e consentimentos), RFC-PEP-015 (layout tablet/mobile); Spike RFC-009 (Scribe; pós-consulta em trechos entregue, streaming pendente) e [[SPK-04-requisitos-do-scribe|SPK-04]]

## Objetivo

O médico conversa com o paciente; o PEP captura o áudio com consentimento registrado sobre um termo versionado, o Scribe da Spike transcreve, separa locutores, descarta conversa casual e preenche o modelo da especialidade ao vivo, citando o trecho de origem. Depois, o médico decide trecho a trecho, assina, e a guia nasce em rascunho. O áudio nunca é guardado; a transcrição é material de apoio com prazo de retenção, não prontuário.

## O que já existe e o que falta

A fatia do Marco D (ver [Notas de implementação](#notas-de-implementação)) entregou o fluxo ponta a ponta em **modo trechos**: consentimento por checkbox (`v2`), áudio em ciclos de ~12 s por HTTP, transcrição e estruturação na Spike, revisão trecho a trecho, "Assinar e finalizar" com autoria por trecho, zero retenção de áudio e degradação sem IA. Este documento descreve o **estado-alvo** e marca o que falta:

| # | Lacuna | Seção | Fatia |
| :--- | :--- | :--- | :--- |
| L1 | Termo de consentimento versionado, "Ver termo", canal, revogação ("interromper a pedido do paciente") | [Consentimento](#consentimento-de-gravação) | F1 |
| L2 | Política de retenção da transcrição e expurgo; trilha que sobrevive ao expurgo | [Retenção](#retenção-da-transcrição) | F2 |
| L3 | "Marcar trecho" que influencia a estruturação; "Revisar trecho" descartado | [Regras](#regras) | F3 |
| L4 | Resumo ao paciente na aba "Resumo" do painel (hoje prévia) | [UI](#ui) | F4 |
| L5 | Tablet e mobile (26b) | [UI](#ui) | F5 |
| L6 | Modelos por especialidade (sai a constante `MODELO_ORTOPEDIA_JOELHO`) | [Modelos](#modelos-por-especialidade) | F6 |
| L7 | Streaming por WebSocket (sai o atraso de ~12 s) | [Streaming](#streaming-por-websocket) | F7 |
| L8 | Feedback das decisões à Spike | [Eventos](#eventos-e-integração-com-a-spike) | F8 |
| — | Assinatura ICP-Brasil da evolução com a trilha anexada | RFC-PEP-011 (só referência aqui) | — |

## Escopo

**Dentro (P2)**

- [RF-VOZ-01] Consentimento antes de cada gravação sobre termo versionado (texto, versão e hash), aceito pelo paciente no dispositivo ou atestado verbalmente pelo médico; recusa registrada; revogação a qualquer momento interrompe a sessão e descarta o que foi captado. Pausa e marcação de trecho, com a marcação levada à estruturação.
- [RF-VOZ-02] Transcrição com locutores (médico, paciente, acompanhante, outro) e tempo, ao vivo por streaming; conversa casual descartada com "Revisar trecho" para devolvê-la à estruturação.
- [RF-VOZ-03] Preenchimento do **modelo da especialidade** (versão publicada da RFC-PEP-004) campo a campo com origem; troca para SOAP; revisão trecho a trecho (por campo ou por seção) com aceitar/editar/descartar, aviso de termos acrescentados, atalhos e "aceitar tudo"; nada entra na evolução sem decisão.
- [RF-VOZ-04] Resumo ao paciente sugerido pela Spike na aba "Resumo" do painel Tissia IA, levado ao documento `resumo_paciente` em rascunho para revisão. O envio por WhatsApp segue a RFC-PEP-004/014 (consentimento do canal) e não é desta RFC.
- Captura no navegador desktop, tablet e celular (26b), com microfone da sala ou do dispositivo e alvos de toque ≥ 44 px.
- Zero retenção de áudio no PEP (servidor e dispositivo) e na Spike.
- Retenção da transcrição com expurgo automático; trilha de revisão (trecho de IA, decisão, texto final e **trecho de origem literal**) guardada com o prontuário.
- Degradação em cadeia: streaming → trechos por HTTP (existente) → sem transcrição (o médico escreve). Nunca bloqueia o atendimento.
- Habilitação por clínica pela flag `modulos.voz` (RF-PLT-04, [[release-plan-pep|release plan]]).

**Fora**

- Ditado de prescrição por voz (medir antes; Spike RFC-009: protocolos por clique tendem a ser mais rápidos).
- Assinatura ICP-Brasil (RFC-PEP-011): aqui só o contrato da trilha que ela anexa.
- Envio do resumo por WhatsApp/e-mail (RFC-PEP-004, RFC-PEP-014, integrações).
- Áudio de telemedicina (P3, RFC-PEP-016; SPK-04 pergunta 2).
- Resumo ao paciente gerado da evolução escrita sem escuta (ver Perguntas abertas).
- Layout de revisão (tela 07) dedicado ao celular: no celular a revisão abre em uma coluna, sem desenho próprio.

## Design

### Arquitetura

```
Navegador ──WSS /api/v1/escuta/{sid}/stream (PCM16 16 kHz mono, frames de 100 ms)──► Gateway de voz (PEP, módulo prontuario)
   ▲                                                                                   │ autentica (ticket), confere consentimento,
   │ eventos: parcial, segmento, campo, descartado, estado, degradado                  │ grava só texto, nunca o áudio
   └───────────────────────────────────────────────────────────────────────────────────┤
                                                                                       └─WSS──► Spike Scribe /v1/consultas/stream (SPK-04 etapa 2)
Reserva (existente): POST /api/v1/escuta/{sid}/audio a cada ~12 s ──► Spike /v1/consultas/transcrever + /estruturar
```

**Decisões desta revisão** (eram "Decisões a tomar" no rascunho):

1. **Gateway no PEP**, não navegador direto na Spike. A chave da Spike fica no servidor, o consentimento é conferido antes do primeiro frame, o vínculo paciente ↔ `ref` fica no PEP (ADR-PEP-0005) e o modo trechos já segue esse caminho. Navegador direto só volta à mesa se a latência medida no piloto passar dos limites de [Critérios de aceite](#critérios-de-aceite).
2. **Formato do áudio: PCM 16 bits, 16 kHz, mono**, capturado por `AudioWorklet` (reamostragem no worklet), em frames de 100 ms. É aceito por todos os candidatos de STT da Spike (Chirp, Deepgram, Whisper, Gemini) e funciona igual no Safari do iPad/iPhone, ao contrário do `MediaRecorder` (WebM/Opus no Chrome, MP4/AAC no Safari, que não serve para streaming em vários STT). Custo: ~256 kbit/s por consulta, aceitável em Wi-Fi de consultório e 4G. `# ponytail:` PCM cru; Opus no worklet só se a banda medida no piloto incomodar. O SPK-04 propôs Opus: ajuste vira proposta à Spike (Perguntas abertas).
3. **Sem buffer de áudio persistente no dispositivo.** Durante uma queda de rede o navegador guarda até 120 s de frames **em memória** e reenvia ao reconectar; passou disso, a sessão pausa sozinha. O `apps/ui/src/app/offline-audio-queue.ts` (IndexedDB, hoje sem uso) sai na F7: IndexedDB sobrevive a travamento e fechamento da aba, o que seria retenção de áudio no dispositivo. O offline do prontuário (ADR-PEP-0015) não inclui a escuta: sem rede não há transcrição.
4. **WebSocket no Cloud Run** (`@nestjs/websockets` + `@nestjs/platform-ws`, sem socket.io), no processo `api`, com tempo máximo de requisição de 60 min **[validar limite atual do Cloud Run]**; consulta mais longa reconecta pelo mesmo protocolo de retomada. GKE só se o teste de carga reprovar (ADR-PEP-0009).

### Máquina de estados da sessão de escuta

```
            (consentimento aceito + POST /escuta)
                         │
                         ▼
   ┌──────────────── gravando ◀──(retomar)── pausada ◀─┐
   │                    │  └──(pausar | 10 min sem áudio | rede > 120 s)──┘
   │                    │
   │            (encerrar e revisar)
   │                    ▼
   │               encerrada  ──(decidir trechos; "Assinar e finalizar")──▶ finalizada
   │
   ├──(paciente revoga, de gravando/pausada/encerrada)──▶ interrompida
   └──(atendimento fechado sem finalizar | pausada > 4 h)──▶ abandonada
```

| Estado | Persistido | Aceita áudio | Aceita decisão de trecho | Transcrição |
| :--- | :--- | :--- | :--- | :--- |
| `gravando` | sim | sim | não | cresce |
| `pausada` | sim | não (409) | não | parada |
| `encerrada` | sim | não | sim | completa, em revisão |
| `finalizada` | sim | não | não (já aplicada na evolução) | até o expurgo |
| `interrompida` | sim (novo) | não | não | **apagada na hora**; fica só metadado |
| `abandonada` | sim (novo) | não | não | apagada em 24 h |

- `reconectando` é estado só do cliente (banner "Reconectando… áudio guardado neste aparelho por até 2 min").
- Uma sessão ativa (`gravando`/`pausada`) por atendimento (já garantido por `iniciar`). Uma conexão WebSocket por sessão: a mais nova vence e a anterior recebe `substituida` e fecha.
- `encerrar` é idempotente; `finalizar` é idempotente (como hoje).

### Consentimento de gravação

- **Termo versionado:** texto aprovado pelo jurídico em tabela global `termos` (abaixo), imutável, com `hash` SHA-256 do texto. A versão vigente por finalidade é a de maior `vigente_desde ≤ agora`. O `v2` atual vira a primeira linha (texto do checkbox de hoje); o termo do jurídico entra como `v3` **[validar texto com jurídico/DPO]**.
- **Canal** (`canal`, mesmo vocabulário de `consentimentos.canal` da RFC-PEP-002; `papel_digitalizado` não se aplica a cada gravação):
  - `termo_digital`: o paciente (ou responsável) lê o termo em "Ver termo" no próprio dispositivo da consulta e toca em "Li e concordo"; o médico confirma. É assinatura eletrônica simples (evidências: versão, hash, data/hora, dispositivo, usuário autenticado) **[validar enquadramento na Lei nº 14.063/2020]**.
  - `verbal_registrado`: o paciente não pode ler ou tocar (baixa visão, analfabetismo, mobilidade); o médico lê o termo em voz alta e atesta. Exige o motivo (lista fechada). **[validar aceitação com jurídico]**
- **Assinante** (`assinante`): `paciente` ou `responsavel_legal` (menor ou incapaz). Sem nome do responsável no registro na P2 (ver Perguntas abertas).
- **Validade:** por atendimento. Nova sessão no mesmo atendimento reaproveita o último consentimento `aceito` se não houver `revogado` depois dele. Termo trocado de versão durante o atendimento não invalida o aceite já dado.
- **Revogação ("interromper a pedido do paciente"):** nova linha `decisao = 'revogado'`. Na mesma transação a sessão ativa ou em revisão vai para `interrompida`, os `transcricao_segmentos`, a `estruturacao` e os `trechos_ia` pendentes da sessão são apagados, e a conexão de streaming fecha. O que já foi **aceito e finalizado** na evolução não é tocado (prontuário imutável; correção só por adendo). **[validar com DPO: descarte imediato como regra]**
- O registro é append-only (trigger bloqueia `UPDATE`/`DELETE`, inclusive para o admin), como `evolucao_versoes`.
- **Relação com o consentimento do cadastro** (`consentimentos`, finalidade `escuta_ambiental`, RF-PAC-06 da RFC-PEP-002): são registros diferentes. O do cadastro é a autorização de longo prazo do paciente (tela 04, revogável lá); o de `consentimentos_gravacao` é a confirmação **a cada gravação** (RFC-PEP-014 §1.3) e é o único exigido para gravar. Regra única: se o último registro do paciente em `consentimentos` para `escuta_ambiental` estiver revogado (sem outro vigente), `iniciar` responde 409 "Paciente revogou a escuta ambiental no cadastro"; sem registro nenhum, vale só o da gravação. Revogação no cadastro (`ConsentimentoRevogado`) interrompe as sessões ativas do paciente, como a revogação da tela 06. "Interromper" na tela 06 revoga só a gravação, não o cadastro.
- A tela 25 e o selo da tela 04 listam as duas origens.

### Retenção da transcrição

- **A transcrição não é prontuário.** O prontuário é a evolução assinada e a trilha de revisão. A transcrição completa é apoio à revisão e fica no banco do tenant por **30 dias após `finalizada`** **[validar prazo com jurídico/DPO; Q2]**; `abandonada` por 24 h; `interrompida` é apagada na hora.
- **Trilha sobrevive ao expurgo:** em `finalizar`, cada trecho de IA copia o texto literal dos segmentos de origem para `trechos_ia.origens_texto` (lista `{ordem, inicio_s, fim_s, locutor, texto}`). A trilha (trecho de IA, decisão, texto final, termos acrescentados, origens literais, autor, horário) tem a retenção do prontuário e é o que a RFC-PEP-011 anexa à assinatura.
- **Expurgo:** job diário BullMQ `escuta-expurgo` (processo `worker`) apaga `transcricao_segmentos` e zera `sessoes_escuta.estruturacao` das sessões vencidas, preenche `expurgada_em` e publica `TranscricaoExpurgada`. Lote de 500 sessões por transação; idempotente.
- **Spike:** não guarda áudio nem transcrição (Spike RFC-009). Envio da transcrição para avaliação só com o opt-in `guardar_entrada_para_avaliacao` da clínica ([[SPK-01-api-de-clinicas-e-flags|SPK-01]]) e o termo correspondente; áudio nunca.
- `# ponytail:` prazo único (constante `RETENCAO_TRANSCRICAO_DIAS = 30` em `@pep/contracts`); prazo por clínica só se o jurídico pedir.

### Modelos por especialidade

- A sessão usa a versão publicada do modelo do atendimento (`atendimentos.modelo_versao_id`, RFC-PEP-004); sem modelo, SOAP (comportamento de hoje). O médico pode trocar o modelo na tela 06 enquanto grava (lista da biblioteca da especialidade do profissional); a troca reestrutura com a transcrição acumulada.
- Vai à Spike `modelo: { id, versao, campos: [{ id, rotulo, tipo, opcoes? }] }` (formato do SPK-04). Só tipos `texto`, `selecao`, `lista` e `escala`; escores, cálculos, tabelas e imagem anotada não vão (o PEP calcula; campo fora desses tipos fica para o médico).
- O PEP revalida a resposta: valor de `selecao`/`lista` fora das `opcoes` e `escala` fora da faixa viram `pendente`; campo sem `origens` vira `pendente` (regra R1 do SPK-04, conferida também no PEP).
- `estrutura` da sessão (`modelo` | `soap`): o botão "Trocar para SOAP" muda a estrutura usada no encerramento. Com `modelo`, `encerrar` cria um trecho de IA **por campo preenchido** (`campo_id`); com `soap`, um por seção (hoje). `finalizar` grava em `conteudo.campos[campo_id]` ou `conteudo.soap[secao]` com a autoria por trecho (RF-PRO-07).
- A constante `MODELO_ORTOPEDIA_JOELHO` sai quando o pacote de Ortopedia estiver publicado no catálogo (RFC-PEP-004, RF-MOD-07); até lá ela é o modelo de catálogo de reserva.

### Streaming por WebSocket

**Abertura:** `POST /api/v1/escuta/{sid}/ticket` (autenticado por Bearer) devolve um ticket opaco de uso único, válido por 30 s, guardado no Redis com `tenant_id`, `usuario_id` e `sid`. O navegador abre `wss://…/api/v1/escuta/{sid}/stream?ticket=…` (o token de acesso nunca vai na URL). O gateway consome o ticket, abre a transação `Db.comTenant` com o contexto do ticket e confere sessão `gravando`, consentimento vigente e `modulos.voz`.

**Do navegador ao gateway**

| Mensagem | Tipo | Conteúdo |
| :--- | :--- | :--- |
| áudio | binário | 4 bytes `seq` (uint32 BE) + 3.200 bytes PCM16LE (100 ms) |
| `inicio` | texto JSON | `{ "t": "inicio", "formato": "pcm16le_16k_mono", "ultimoSeqEnviado": n, "tempoS": 312.4 }` |
| `pausar` / `retomar` | texto | Mesma regra do `PATCH /escuta/{sid}/estado` |
| `marcar` | texto | `{ "t": "marcar", "tempoS": 254.1 }` |
| `fim` | texto | Encerra o envio; o gateway chama a estruturação final e responde `estado: encerrada` |

**Do gateway ao navegador**

| Mensagem | Conteúdo |
| :--- | :--- |
| `pronto` | `{ ultimoSeqConfirmado, sessao }` (estado inteiro, para retomar a tela) |
| `ack` | `{ seq }` a cada ~1 s: o navegador libera do buffer em memória |
| `parcial` | `{ locutor, inicioS, texto }` (não persistido) |
| `segmento` | `Segmento` de `@pep/contracts` (persistido antes do envio) |
| `campo` | `CampoEstruturado` (estruturação incremental) |
| `descartado` | `{ inicioS, fimS, origens }` |
| `estado` | `{ estado }` |
| `degradado` | `{ motivo: "tissia_ai_indisponivel" \| "stream_indisponivel" \| "sem_ia" }`: o cliente cai para o modo trechos |
| `substituida` / `erro` | Fecha a conexão (`erro.codigo`: `consentimento`, `sessao_encerrada`, `ticket_invalido`, `limite`) |

- **Retomada:** ao reconectar, `inicio.ultimoSeqEnviado` e o `pronto.ultimoSeqConfirmado` dizem o que reenviar; frames repetidos são ignorados pelo `seq`. Segmentos finais são persistidos antes de ir ao navegador, então nenhum se perde na reconexão.
- **Gateway → Spike:** `POST /v1/consultas/sessoes` (com a API key, `consentimento_ref = "cg_" + consentimento.id`, `modelo`, formato) e WebSocket `/v1/consultas/stream` (SPK-04 etapa 2). Eventos da Spike viram mensagens acima; `transcricao_final` grava `transcricao_segmentos`; `campo_atualizado` atualiza `estruturacao` no máximo a cada 5 s (debounce). `fim` recebe a estruturação completa (formato da etapa 1) e segue o mesmo `encerrar` de hoje.
- **Limites:** frame ≤ 4 KB; até 15 frames/s por conexão; sessão ≤ 2 h (validade do token da Spike); 10 min sem áudio → `pausada`.
- **Degradação:** Spike fora, sem streaming ou `degradado` → `degradado` ao cliente, que passa a mandar ciclos de 12 s por `POST /escuta/{sid}/audio` (existente). Sem transcrição nenhuma, a tela diz e o médico segue escrevendo no prontuário. Nenhuma dessas falhas fecha a sessão nem o atendimento.

### Modelo de dados

Tabelas existentes (migração `0007_escuta_ambiental`), todas com `tenant_id NOT NULL` e, portanto, RLS `tenant_isolation` pela convenção de `drizzle/rls/001_tenant_policies.sql`: `consentimentos_gravacao`, `sessoes_escuta`, `transcricao_segmentos`, `trechos_ia`. Mudanças (migração aditiva `00NN_escuta_p2`):

```sql
-- Global, sem dado de tenant (como `tenants`): não entra na convenção de RLS.
-- Papel da aplicação só com SELECT; texto novo entra por migração, nunca por API.
CREATE TABLE termos (
  id uuid PRIMARY KEY,
  finalidade text NOT NULL,             -- mesmas finalidades de `consentimentos` (RFC-PEP-002); `consentimentos.versao_termo` passa a apontar para cá
  versao text NOT NULL,                 -- v2, v3…
  texto text NOT NULL,                  -- markdown aprovado pelo jurídico
  hash text NOT NULL,                   -- sha256 do texto
  vigente_desde timestamptz NOT NULL,
  UNIQUE (finalidade, versao)
);  -- trigger bloqueia UPDATE/DELETE

ALTER TABLE consentimentos_gravacao
  ADD COLUMN termo_id uuid REFERENCES termos(id),          -- backfill: linha v2
  ADD COLUMN termo_hash text,
  ADD COLUMN canal text NOT NULL DEFAULT 'termo_digital',  -- termo_digital | verbal_registrado (vocabulário da RFC-PEP-002)
  ADD COLUMN motivo_verbal text,                            -- obrigatório se canal = verbal_registrado (lista fechada)
  ADD COLUMN assinante text NOT NULL DEFAULT 'paciente',    -- paciente | responsavel_legal
  ADD COLUMN dispositivo text,                              -- desktop | tablet | celular
  ADD COLUMN sessao_id uuid;                                -- só em 'revogado': a sessão interrompida
-- decisao passa a aceitar 'revogado'; trigger append-only (UPDATE/DELETE bloqueados)

ALTER TABLE sessoes_escuta
  ADD COLUMN consentimento_id uuid REFERENCES consentimentos_gravacao(id),
  ADD COLUMN modo text NOT NULL DEFAULT 'trechos',          -- trechos | stream (o último usado)
  ADD COLUMN estrutura text NOT NULL DEFAULT 'soap',        -- soap | modelo
  ADD COLUMN modelo_versao_id uuid,                         -- RFC-PEP-004 (FK quando modelo_versoes existir)
  ADD COLUMN tissia_ai_estruturacao_id text,                    -- para o feedback (F8)
  ADD COLUMN atividade_em timestamptz NOT NULL DEFAULT now(),
  ADD COLUMN expurgada_em timestamptz;
-- estado passa a aceitar 'interrompida' e 'abandonada'
CREATE INDEX ix_sessao_expurgo ON sessoes_escuta (estado, encerrada_em) WHERE expurgada_em IS NULL;

ALTER TABLE transcricao_segmentos
  ADD COLUMN marcado boolean NOT NULL DEFAULT false;        -- marcação do médico (F3)

ALTER TABLE trechos_ia
  ALTER COLUMN secao DROP NOT NULL,
  ADD COLUMN campo_id text,                                 -- trecho de campo do modelo (F6)
  ADD COLUMN marcado boolean NOT NULL DEFAULT false,        -- alguma origem marcada pelo médico
  ADD COLUMN origens_texto jsonb;                           -- cópia literal das origens, gravada em finalizar (F2)
-- CHECK (secao IS NOT NULL OR campo_id IS NOT NULL)
```

- `sessoes_escuta.unidade_id` e `atendimento_id` já existem; `marcas` (jsonb de segundos) continua como registro bruto dos toques.
- Nenhuma coluna guarda áudio, nem em base64 nem em referência a objeto (teste existente continua valendo).
- `@pep/contracts/escuta.ts` ganha: `TermoSchema`, `ConsentimentoSchema` estendido (`termoVersao`, `termoHash`, `canal`, `motivoVerbal`, `assinante`), estados novos em `SessaoEscutaSchema`, `campoId`/`marcado`/`origensTexto` em `TrechoIASchema`, mensagens do WebSocket como união discriminada por `t`, `RETENCAO_TRANSCRICAO_DIAS` e `TrilhaRevisaoSchema`.

### API (módulo `prontuario`, controller `escuta`)

Rotas existentes continuam (`GET /atendimentos/{id}/escuta`, `POST /atendimentos/{id}/escuta`, `POST /escuta/{sid}/audio`, `POST /escuta/{sid}/segmentos`, `PATCH /escuta/{sid}/estado`, `POST /escuta/{sid}/marcas`, `POST /escuta/{sid}/encerrar`, `POST /escuta/{sid}/trechos[/{tid}]/decisao`, `POST /escuta/{sid}/finalizar`). Novas ou alteradas:

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `GET` | `/api/v1/termos/{finalidade}/vigente` | Termo vigente (`versao`, `texto`, `hash`) para "Ver termo" |
| `POST` | `/api/v1/atendimentos/{id}/escuta/consentimento` | **Alterada:** `{ "decisao": "aceito" \| "recusado", "termoVersao": "v3", "termoHash": "…", "canal": "termo_digital", "motivoVerbal": null, "assinante": "paciente" }` → 409 se versão não vigente ou hash diferente; 422 se `verbal_registrado` sem motivo; 409 se o cadastro tiver a escuta revogada |
| `POST` | `/api/v1/atendimentos/{id}/escuta/consentimento/revogar` | Revoga, interrompe a sessão e apaga a transcrição dela (idempotente) |
| `POST` | `/api/v1/escuta/{sid}/ticket` | Ticket de uso único (30 s) para o WebSocket |
| `GET` (upgrade) | `/api/v1/escuta/{sid}/stream?ticket=` | WebSocket de streaming (protocolo acima) |
| `PATCH` | `/api/v1/escuta/{sid}/segmentos/{ordem}` | `{ "clinico": true }`: "Revisar trecho" devolve conversa descartada à estruturação (reestrutura) |
| `PATCH` | `/api/v1/escuta/{sid}/estrutura` | `{ "estrutura": "modelo" \| "soap", "modeloVersaoId": "…" }` (só em `gravando`/`pausada`; reestrutura) |
| `GET` | `/api/v1/escuta/{sid}/trilha` | Trilha de revisão (`TrilhaRevisaoSchema`), consumida pela RFC-PEP-011 e pela impressão |

- Permissão nova `escuta:usar` (perfil `medico`) para iniciar, consentir, revogar, decidir e finalizar; `GET` da escuta segue com `prontuario:ler`. Enfermagem não grava nem finaliza na P2. Finalizar exige ser o profissional do atendimento.
- Todas as rotas respondem 404 se `modulos.voz` estiver desligado para a clínica.
- `GET /atendimentos/{id}/escuta` e `/trilha` geram log de acesso "quem viu" em `auditoria_eventos` (RFC-PEP-014).

### Regras

- **Sem consentimento vigente, nada é captado:** `iniciar`, `audio`, `segmentos` e o WebSocket respondem 409 sem consentimento `aceito` posterior ao último `revogado`.
- **Marcar trecho (F3):** a marca no instante `t` marca o segmento que contém `t` e o imediatamente anterior (o médico costuma apertar depois de ouvir). Segmento marcado vai à Spike com `clinico: true`; se a Spike ainda assim o devolver em `trechos_descartados`, o PEP ignora o descarte para ele. Trecho de IA com origem marcada leva o selo "Marcado" na revisão e aparece primeiro na navegação `J/K`. Marca feita antes de o segmento chegar (streaming) é aplicada quando ele chegar.
- **Revisar trecho descartado (F3):** devolver um segmento à estruturação marca `clinico = true`, refaz a estruturação e registra a decisão (autor e horário) na trilha. Em `encerrada`, gera novos trechos pendentes para o que mudou.
- **Origem obrigatória:** campo ou trecho sem origem válida (ordem existente na sessão) nunca vira trecho de IA; vira `pendente` (R1 do SPK-04, conferido no PEP).
- **Termos acrescentados:** `termosNovos` (mesma função no cliente e no servidor) continua; a trilha guarda a lista.
- **Assinatura só sem pendentes** (RF-ASS-02, já entregue); com a RFC-PEP-011, "Assinar e finalizar" passa a chamar o fluxo ICP-Brasil com a trilha anexada; antes dela, assinatura eletrônica simples.
- **Revogação depois de `finalizada`:** só registra a revogação (a sessão já não tem nada pendente); a transcrição é apagada no mesmo ato, antes do prazo de retenção.
- **Atendimento fechado sem finalizar a escuta:** sessão vai a `abandonada`; os trechos pendentes não entram na evolução.

### Eventos e integração com a Spike

Pelo outbox (ADR-PEP-0008), só com ids e contagens, nunca texto:

- `ConsentimentoGravacaoRegistrado { consentimento_id, atendimento_id, decisao, termo_versao, canal }` → auditoria (RFC-PEP-014).
- Consome `ConsentimentoRevogado { finalidade: "escuta_ambiental" }` (RFC-PEP-002): interrompe as sessões ativas ou em revisão do paciente.
- `EscutaFinalizada { sessao_id, atendimento_id, modo, estrutura, duracao_s, aceitos, editados, descartados, com_termos_novos, provedor }` → BI (aceite da IA, RF-BI-03) e feedback à Spike (F8). Segue publicando `EvolucaoVersionada` e `AtendimentoFechado` como hoje.
- `EscutaInterrompida { sessao_id, atendimento_id }` e `TranscricaoExpurgada { sessao_id, motivo: "prazo" | "abandonada" | "revogacao" }` → auditoria.
- **Feedback (F8):** consumidor de `EscutaFinalizada` envia `POST /v1/consultas/estruturacoes/{tissia_ai_estruturacao_id}/feedback` com `item_id = "campo:<id>"` ou `"soap:<S|O|A|P>"` e a ação; best-effort, nunca bloqueia (SPK-04).
- Do paciente vão à Spike só `ref`, sexo e idade (invariante; teste existente estendido ao `inicio` do streaming e à sessão da Spike).

## UI

- **Tela 06 (desktop):** como hoje, mais: "Ver termo" na faixa de consentimento abre o termo vigente em `Sheet` (versão, data de aceite, canal); no pré-gravação, o painel de consentimento mostra o termo, "Li e concordo" do paciente, opção "Consentimento verbal" (com motivo) e "Responsável legal"; "Paciente pediu para interromper" (confirmação em duas etapas) chama a revogação. Transcrição com a linha parcial (cursor) ao vivo; "Revisar trecho" no bloco de conversa descartada; "Marcar trecho" (`M`) com feedback visual no segmento marcado; seletor de modelo (badge "Ortopedia · Joelho v3" vira menu) e "Trocar para SOAP"; rótulo do microfone pelo `enumerateDevices` e qualidade do sinal pelo nível RMS; banner "Reconectando…" e "Transcrição em trechos (mais lenta)" no modo reserva.
- **Tela 07:** trechos por campo do modelo (agrupados pelas seções do modelo) ou por seção SOAP; selo "Marcado"; origem literal no painel da transcrição. Depois do expurgo, a revisão de uma sessão finalizada mostra só a trilha (origens literais), com o aviso "Transcrição completa removida em dd/mm (política de retenção)".
- **Painel Tissia IA, aba "Resumo" (tela 05):** com sessão encerrada/finalizada, mostra o `resumoPaciente` sugerido (selo "Sugerido pela IA · gerado da conversa, antes da sua revisão"), contador de caracteres (≤ 1.000, R5 do SPK-04) e "Levar aos documentos", que salva o documento `resumo_paciente` em rascunho com `origem: ia_sugerida` pelo `PUT /atendimentos/{id}/documentos/resumo_paciente` existente e abre a tela 08 no cartão. Sem escuta, o texto atual ("o médico escreve o resumo nos documentos") continua. O botão "Usar resumo sugerido pela IA" da tela 08 permanece.
- **26b (celular) e tablet:** a mesma rota `/atendimento/:id/escuta`, responsiva abaixo de `md` (834): cabeçalho escuro com tempo grande e forma de onda, selo "Consentimento ok" (toque abre o termo), transcrição em lista, cartão do modelo com "N de M campos preenchidos pela IA" e selo "Ao vivo", rodapé fixo com "Pausar" e "Encerrar e revisar" (48 px). "Marcar trecho" entra como botão de ícone de 44 px no cabeçalho (o 26b não o desenha; registrar o ajuste no canvas). `navigator.wakeLock` mantém a tela acesa enquanto grava; ao ir para segundo plano ou bloquear a tela (`visibilitychange`), a sessão pausa e mostra "Pausada: o aparelho interrompeu o microfone". A referência estática `apps/ui/src/modulos/dispositivos/MobileVoz.tsx` sai quando a tela real cobrir o 26b.
- Aviso legal fixo em todas as variantes: "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."

## Segurança e LGPD

- **Base legal:** consentimento específico para a gravação, além da tutela da saúde para o atendimento (conformidade §1) **[validar com jurídico]**. Voz como dado biométrico em potencial **[validar enquadramento]**: zero retenção de áudio é requisito.
- **Zero retenção de áudio:** nem coluna, nem objeto, nem log, nem fila, nem IndexedDB. No servidor, frames vivem só no buffer do gateway até irem à Spike; no navegador, só no buffer em memória de até 120 s. Teste automatizado no banco + revisão de buckets e logs antes do piloto.
- **Transferência internacional:** o áudio vai da Spike ao Vertex AI no endpoint `global` sem scrubber (Spike ADR-0010, decisão 4); a transcrição passa pelo scrubber da Spike antes do LLM. O item "nenhuma transferência internacional no PEP" da [[conformidade-regulatoria|conformidade]] continua verdadeiro para o PEP, mas o RIPD da voz precisa descrever o suboperador no exterior e o mecanismo (cláusulas-padrão ANPD) **[validar com DPO]**. Alternativa contratual: `VERTEX_LOCATION` regional quando um cliente exigir.
- **RIPD específico da voz** (revisão do RIPD do PEP prevista para a P2): finalidades, fluxo áudio → Spike → Vertex, retenção da transcrição, revogação, riscos de alucinação e mitigação (origem obrigatória, revisão humana). Condição para ligar `modulos.voz` em `prod`.
- **Logs e métricas:** nunca texto, áudio, nome falado ou tamanho de frame por paciente; só ids, estados, contagens e latências. `ticket` nunca logado (a URL do WebSocket passa por filtro no logger e no proxy).
- **RLS:** tabelas de escuta sob `tenant_isolation`; o gateway abre `Db.comTenant` com o contexto do ticket a cada escrita (o WebSocket não carrega o `AsyncLocalStorage` da requisição HTTP original). `termos` é global e só de leitura para a aplicação.
- **Acesso:** só o profissional do atendimento grava e finaliza; faturista não vê transcrição nem trilha (só a evolução no recorte da guia).

## Critérios de aceite

- [ ] Consentimento com `termoHash` diferente do vigente → 409; `verbal_registrado` sem motivo → 422; escuta revogada no cadastro do paciente → 409 em `iniciar`; `UPDATE`/`DELETE` em `consentimentos_gravacao` e `termos` falham no banco, inclusive para o admin.
- [ ] Revogação durante a gravação: a sessão vai a `interrompida`, o WebSocket fecha em ≤ 1 s, `transcricao_segmentos` e `trechos_ia` pendentes da sessão somem e a evolução não muda (teste de integração).
- [ ] Nenhuma tabela, objeto ou chave Redis guarda áudio depois de uma sessão de streaming de 5 min (teste no banco + varredura do Redis); `offline-audio-queue.ts` removido e nenhum `indexedDB.open` com áudio no bundle.
- [ ] Reconexão no meio da consulta (rede derrubada por 30 s no E2E) não perde segmento final nem duplica (`seq` e `ordem` contínuos).
- [ ] Spike desligada durante o streaming: o cliente passa ao modo trechos sem perder a sessão; sem transcrição nenhuma, o atendimento fecha normalmente.
- [ ] Latência medida em ambiente `hml` com áudio consentido de 15 min: parcial p95 ≤ 1,5 s, segmento final p95 ≤ 3 s, campo atualizado p95 ≤ 8 s após o segmento **[validar metas com a Spike]**.
- [ ] Segmento marcado nunca aparece como descartado; trecho com origem marcada traz o selo na revisão.
- [ ] "Revisar trecho" devolve o segmento e a estruturação seguinte pode usá-lo; a ação fica na trilha.
- [ ] Com modelo publicado: valor de `selecao` fora das opções vira `pendente`; finalizar grava `conteudo.campos` com autoria por campo; sem modelo, SOAP como hoje.
- [ ] Expurgo: sessão finalizada há 31 dias perde a transcrição, mantém `trechos_ia.origens_texto`; `GET /trilha` continua completo; `TranscricaoExpurgada` publicado uma vez (job reexecutado não duplica).
- [ ] Aba "Resumo" leva o resumo ao documento em rascunho com `origem: ia_sugerida`; envio continua exigindo `revisado`.
- [ ] 26b: E2E em viewport 390×844 (Playwright, WebKit) grava, pausa, marca e encerra; alvos ≥ 44 px (axe).
- [ ] Teste de propriedade do SpikeAdapter cobre `sessoes` e `inicio` do streaming: só `ref`, sexo e idade do paciente.
- [ ] `modulos.voz` desligado: rotas 404 e botão "Iniciar escuta" oculto.

## Riscos

- Qualidade do ASR em consultório (ruído, sobreposição, termos médicos pt-BR); WER medido em áudio real consentido (Spike test plan F4).
- Latência e custo por minuto do streaming; ~256 kbit/s por consulta com PCM.
- WebSocket no Cloud Run (limite de tempo e afinidade de sessão) **[validar]**; plano B é GKE (ADR-PEP-0009).
- Consentimento mal registrado = risco LGPD alto (dado sensível, áudio). RIPD específico antes de `prod`.
- Microfone em tablets e iOS: interrupção em segundo plano, PWA instalada no iOS com `getUserMedia` **[validar versões mínimas]**.
- Alucinação: campo preenchido sem trecho de origem não pode existir (regra igual à Spike ADR-0008, conferida no PEP).
- Transferência internacional do áudio (Spike ADR-0010) pode ser vetada por contrato de cliente.

## Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | _(Resolvida: ASR na Spike como Scribe, vendável à parte — [[ADR-PEP-0013-asr-na-spike\|ADR-PEP-0013]], aceita em 09/11/2026.)_ | — | — |
| Q2 | Prazo de retenção da transcrição (default 30 dias após finalizar) e descarte imediato na revogação | Jurídico/DPO | Ligar F2 em `prod` (o código usa o default) |
| Q3 | Microfone de sala dedicado ou dispositivo do médico no piloto? A RFC suporta os dois pelo seletor | Produto | Não |
| Q4 | Texto do termo `v3`; aceitação do canal `verbal_registrado` por gravação; termo por clínica ou só o nosso | Jurídico/DPO | F1 em `prod` |
| Q5 | Responsável legal: guardar nome e documento do responsável no consentimento? (hoje só `assinante = responsavel_legal`) | Jurídico/DPO | Não (aditivo) |
| Q6 | **Proposta de SPK a abrir** (emenda à etapa 2 do SPK-04): formato `pcm16le_16k_mono` no `inicio`; `transcricao[].marcado` para a Spike priorizar segmentos marcados como evidência; `estruturacao_id` também na resposta do modo trechos | Time da Spike | F7 (formato); F3 funciona sem |
| Q7 | **Proposta de SPK a abrir:** o [[manual-integracao-tissia-ia-v1\|manual de integração]] da Spike não documenta `/v1/consultas/transcrever` e `/v1/consultas/estruturar`; documentar antes de vender o Scribe a terceiros | Time da Spike | Não |
| Q8 | **Proposta de SPK a abrir:** resumo ao paciente a partir da evolução escrita (sem escuta), hoje fora do escopo | Produto + Spike | Não |
| Q9 | Transferência internacional do áudio (Vertex `global`) no RIPD e no contrato com a clínica | DPO | `modulos.voz` em `prod` |
| Q10 | Ajuste do 26b no canvas (botão "Marcar trecho" no cabeçalho) | Design | Não |

## Fatias de entrega

Ordenadas para entregar primeiro o que é exigência legal para dado real e o que não depende da Spike:

1. **F1 — Consentimento versionado:** `termos` (linha `v2` + backfill), colunas novas e trigger append-only em `consentimentos_gravacao`, `GET /termos/{finalidade}/vigente`, consentimento estendido, revogação com interrupção e descarte, consumidor de `ConsentimentoRevogado` do cadastro, estado `interrompida`, "Ver termo" e painel de consentimento na tela 06, permissão `escuta:usar`, flag `modulos.voz`. Sem dependência externa.
2. **F2 — Retenção e trilha:** `origens_texto` em `finalizar`, `GET /escuta/{sid}/trilha`, estado `abandonada`, job `escuta-expurgo`, eventos `EscutaFinalizada`, `EscutaInterrompida`, `TranscricaoExpurgada`, aviso de expurgo na tela 07. Prazo default até Q2.
3. **F3 — Marcação e revisar descartado:** `marcado` em segmentos e trechos, regra de não descarte, selo na revisão, `PATCH /escuta/{sid}/segmentos/{ordem}`. Sem dependência da Spike (Q6 só melhora).
4. **F4 — Aba "Resumo":** painel Tissia IA com o resumo sugerido e "Levar aos documentos". Só UI.
5. **F5 — Tablet e celular (26b):** layout responsivo, wake lock, pausa em segundo plano, botão de marcar no cabeçalho; remove `MobileVoz.tsx`.
6. **F6 — Modelos por especialidade:** depende de `modelos`/`modelo_versoes` publicados (RFC-PEP-004, P1). `estrutura`, `modelo_versao_id`, trechos por campo, `PATCH /escuta/{sid}/estrutura`, seletor de modelo; remove `MODELO_ORTOPEDIA_JOELHO`.
7. **F7 — Streaming por WebSocket:** depende da etapa 2 do Scribe na Spike (SPK-04) e de Q6 (formato). Ticket, gateway `ws`, `AudioWorklet` PCM16, buffer em memória de 120 s, retomada por `seq`, degradação para o modo trechos; remove `offline-audio-queue.ts`.
8. **F8 — Feedback à Spike:** consumidor de `EscutaFinalizada` → `/v1/consultas/estruturacoes/{id}/feedback`. Depende do endpoint na Spike.

A RFC-PEP-011 (ICP-Brasil) consome a trilha da F2; nada aqui depende dela.

## Notas de implementação

**Marco D / telas 06 e 07 (08/11/2026): fatia entregue**, a pedido do produto, antes da P2.

- Telas `/atendimento/:id/escuta` (design `C06-Voz`) e `/atendimento/:id/revisao` (`C07-Revisao`). O botão "Iniciar escuta" e a tecla `V` do prontuário abrem a escuta.
- Consentimento antes de cada gravação (`consentimentos_gravacao`, termo `v2`): sem consentimento aceito a API responde 409; a recusa também fica registrada.
- Captura no navegador (`MediaRecorder`, Opus) em **ciclos de ~12 s, cada um um arquivo independente**, enviados a `POST /api/v1/escuta/{sid}/audio` e repassados à Spike (`/v1/consultas/transcrever`). O PEP não grava o áudio (nenhuma coluna o guarda; teste confirma) e a Spike também não. Só ficam `transcricao_segmentos` (texto, locutor, tempo), `sessoes_escuta` e `trechos_ia`.
- A cada dois trechos (e ao encerrar) a transcrição acumulada vai a `/v1/consultas/estruturar`: campos do modelo "Ortopedia · Joelho" com chip de origem (`mm:ss`), conversa casual descartada, SOAP proposto, resumo ao paciente e **CID e procedimento sugeridos das bases vigentes**. Do paciente vão só `ref`, sexo e idade. Teste confirma.
- Revisão trecho a trecho (`trechos_ia`: pendente, aceito, editado, descartado), com a transcrição ao lado e o trecho de origem destacado, atalhos `J K A E D` e `Shift A`, aviso de **termos acrescentados** pela edição que não estão no trecho nem na transcrição (`termosNovos`, mesma função no cliente e no servidor). "Assinar e finalizar" só habilita sem pendentes; grava uma nova versão da evolução com autoria `ia_sugerida` ou `ia_editada` por trecho, fecha o atendimento (guia só se marcado) e, com pedido de convênio, dispara a análise anti-glosa. A trilha de revisão fica em `trechos_ia`.
- A transcrição aceita também texto pronto (`POST /escuta/{sid}/segmentos`): é o que roda o botão "Usar consulta de exemplo" (reserva da demo e dos testes).
- **Desvios:** transcrição em **trechos por HTTP**, não streaming por WebSocket (a transcrição aparece com atraso de alguns segundos); sem consentimento por termo versionado assinado pelo paciente nem "Ver termo"; sem aba de modelos (o modelo é uma constante, `MODELO_ORTOPEDIA_JOELHO`); sem marcação de trecho que influencie a estruturação (`Marcar trecho` só registra o instante); sem tablet/mobile (26b); assinatura eletrônica simples em vez de ICP-Brasil; a transcrição fica gravada no banco do tenant até a política de retenção do RIPD (Q2); o resumo ao paciente sugerido aparece na tela 08 ("Usar resumo sugerido pela IA"); sem diarização entre trechos além do contexto dos últimos 6 segmentos.
- **Sem IA, nada quebra, e a tela diz:** sem credencial do Vertex a transcrição fica indisponível (aviso na tela) e a estruturação cai no classificador local por palavras-chave da Spike, marcado "Sem IA (modo local)". Com credencial, o Gemini faz as duas etapas.

- **Ajustes após teste com o Gemini (09/11/2026):** a IA devolve uma frase por item do SOAP; o encerramento agora junta os itens de cada seção num único trecho (um bloco por seção, como no design C07), com a união das origens. O pedido sugerido abre com a indicação clínica proposta (avaliação, queixa e exame físico não descartados, até 500 caracteres), que o médico revisa antes de adicionar; a revisão também lista os pedidos do atendimento e deixa adicionar outros do catálogo ou remover. Atendimento particular não mostra a opção de guia/anti-glosa. Depois de "Assinar e finalizar" a tela segue para os documentos (C08).

**Demo de dados fictícios (19/11/2026, PR #30 do `tissia`, em rascunho).** Ver [[demo-dados-ficticios]].

- O seed `clinico` cria 4 sessões de escuta sintéticas (pendente de revisão, revisada, pausada), com texto pronto, sem áudio (o PEP não grava áudio). O "Usar consulta de exemplo" segue como reserva da demo.
- `Voz.tsx`: em sessão retomada o cronômetro parte do tempo já transcrito (antes reiniciava).
- Os textos visíveis de `/revisao` foram reescritos em linguagem de produto (sem RFC ou fase).
