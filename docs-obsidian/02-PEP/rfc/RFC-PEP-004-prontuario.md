---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-004-prontuario.md
---

# RFC-PEP-004 — Prontuário, documentos e modelos por especialidade

**Fase:** P1 · **Status:** Implementada · **ADRs:** ADR-PEP-0002, ADR-PEP-0003, ADR-PEP-0007, ADR-PEP-0008, ADR-PEP-0015
**Telas:** 05 (Prontuário em atendimento), 07 (revisão da evolução, parcial), 08 (Documentos), 09 (Construtor de modelos), 10 (Biblioteca), 11 (Modelo preenchido — Ortopedia)
**Requisitos:** RF-PRO-01 a RF-PRO-10, RF-DOC-01 a RF-DOC-06, RF-MOD-01 a RF-MOD-07
**Depende de:** RFC-PEP-002, RFC-PEP-003, RFC-PEP-014 (auditoria "quem viu"); alimenta RFC-PEP-005 e RFC-PEP-006

## Objetivo

Registro clínico estruturado, imutável e rápido, que já nasce com o que o faturamento precisa (TUSS, CID, evidência DUT) sem que o médico preencha campo de faturamento. O fechamento do atendimento é o gatilho que cria a guia em rascunho e chama a Spike.

## Escopo

**Dentro (P1)**

Prontuário

- [RF-PRO-01] Atendimento ligado ao agendamento (ou avulso), com profissional, unidade, beneficiário/particular e cronômetro.
- [RF-PRO-02] Evolução em SOAP ou por modelo de especialidade; autosave a cada 5 s ou ao sair do campo.
- [RF-PRO-03] Versionamento imutável: cada salvamento gera `evolucao_versoes`; após fechar, mudança só por **adendo** (autor, data, justificativa) — ADR-PEP-0007.
- [RF-PRO-04] Linha do tempo do paciente (consultas, exames, guias, anexos) com filtros; problemas ativos e medicações em uso.
- [RF-PRO-05] Barra de comandos `/` dentro do editor: `/receita`, `/exame`, `/atestado`, `/encaminhamento`.
- [RF-PRO-06] Anexos (PDF/imagem) em armazenamento de objetos, com vínculo ao atendimento.
- [RF-PRO-07] Autoria por trecho: cada bloco de texto marca origem `humano`, `ia_sugerida`, `ia_editada` (Res. CFM sobre IA **[validar nº 2.454/2026]**).
- [RF-PRO-08] Fechar atendimento (`Ctrl+Enter`): valida obrigatórios do modelo, publica `AtendimentoFechado`; se houver pedido de exame/procedimento de convênio, cria guia em rascunho (`GuiaRascunhoCriada`, RFC-PEP-006).
- [RF-PRO-09] Log de acesso a cada prontuário aberto (RFC-PEP-014).
- [RF-PRO-10] Offline (ADR-PEP-0015): o editor da evolução e os documentos em rascunho funcionam sem rede; sincronização com fila local.

Documentos

- [RF-DOC-01] Receita simples (P1), controle especial e antimicrobiano (P2, dependem de ICP-Brasil — RFC-PEP-011), com checagem de alergias.
- [RF-DOC-02] Pedido de exame com TUSS, quantidade e **indicação clínica ≤ 500 caracteres por atendimento** (vira o Campo 23 da guia).
- [RF-DOC-03] Atestado (comparecimento, afastamento, acompanhante); CID só com consentimento `cid_em_atestado`.
- [RF-DOC-04] Encaminhamento para especialidade.
- [RF-DOC-05] Resumo ao paciente em linguagem simples: na P1, escrito pelo médico (ou a partir de modelo); geração por IA na P2 (SPK-04). Envio por WhatsApp só com revisão marcada e consentimento.
- [RF-DOC-06] PDF de cada documento; assinatura eletrônica simples na P1 e ICP-Brasil na P2 (RFC-PEP-011).

Modelos

- [RF-MOD-01] Construtor: seções e campos (texto, seleção, escala 0–10, escore com fórmula, cálculo, tabela, imagem anotada, data/período) e blocos prontos (SOAP, sinais vitais).
- [RF-MOD-02] Condição de exibição por campo (`se McMurray = Positivo`).
- [RF-MOD-03] Vínculos de faturamento por campo/opção: sugerir CID, marcar como evidência DUT de um TUSS. Sempre sugestão revisada pelo médico.
- [RF-MOD-04] Versionamento de modelo: rascunho → publicada; atendimentos guardam a versão usada.
- [RF-MOD-05] Biblioteca por especialidade: duplicar, compartilhar com profissionais da clínica, desativar.
- [RF-MOD-06] Pré-visualização ao vivo no construtor.
- [RF-MOD-07] Dois pacotes do MVP: **Ortopedia** (joelho, coluna) e **Cardiologia** **[validar especialidades com o piloto]**, curados pelo CMO.

**Fora**

- Voz ambiental e estruturação por IA da conversa: RFC-PEP-013 (P2).
- Prescrição integrada a plataformas de receita digital: avaliar na P2 (integracoes.md).
- Laudos e internação: P3.

## Design

### Modelo de dados

```sql
CREATE TABLE atendimentos (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, unidade_id uuid NOT NULL,
  paciente_id uuid NOT NULL, profissional_id uuid NOT NULL,
  agendamento_id uuid, beneficiario_id uuid,
  modelo_versao_id uuid,
  status text NOT NULL,                -- aberto | fechado | cancelado
  iniciado_em timestamptz NOT NULL, fechado_em timestamptz
);

-- append-only (trigger bloqueia UPDATE/DELETE)
CREATE TABLE evolucao_versoes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  atendimento_id uuid NOT NULL REFERENCES atendimentos(id),
  versao int NOT NULL,
  conteudo jsonb NOT NULL,             -- { soap: {...} } ou { campos: {campo_id: valor} }
  autoria jsonb NOT NULL,              -- [{ caminho, origem: humano|ia_sugerida|ia_editada, analise_id? }]
  texto_plano text NOT NULL,           -- render para busca e para a Spike
  autor_id uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now(),
  origem_sync text,                    -- online | offline:<device_id>
  UNIQUE (atendimento_id, versao)
);

CREATE TABLE adendos (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, atendimento_id uuid NOT NULL,
  texto text NOT NULL, justificativa text NOT NULL,
  autor_id uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE documentos (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, atendimento_id uuid NOT NULL,
  tipo text NOT NULL,                  -- receita | pedido_exame | atestado | encaminhamento | resumo_paciente
  conteudo jsonb NOT NULL,             -- itens, posologia, TUSS/qtd, indicação etc.
  status text NOT NULL,                -- rascunho | revisado | assinado | enviado | cancelado
  pdf_objeto text, assinatura_id uuid,
  criado_por uuid NOT NULL, criado_em timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE modelos (id uuid PRIMARY KEY, tenant_id uuid, especialidade text NOT NULL, nome text NOT NULL, ativo boolean NOT NULL DEFAULT true);
-- tenant_id NULL = modelo de catálogo (curado): policy RLS SELECT permite (tenant_id IS NULL OR tenant_id = nullif(current_setting('app.tenant_id', true), '')::uuid); INSERT/UPDATE/DELETE exige tenant_id = sessão
CREATE TABLE modelo_versoes (
  id uuid PRIMARY KEY, tenant_id uuid, modelo_id uuid NOT NULL REFERENCES modelos(id), versao int NOT NULL,
  status text NOT NULL,                -- rascunho | publicada | arquivada
  definicao jsonb NOT NULL,            -- seções, campos, condições, vínculos (schema abaixo em @pep/contracts)
  publicada_em timestamptz, publicada_por uuid,
  UNIQUE (modelo_id, versao)
);
CREATE TABLE modelo_compartilhamentos (
  tenant_id uuid NOT NULL, modelo_id uuid NOT NULL REFERENCES modelos(id), profissional_id uuid NOT NULL,
  PRIMARY KEY (tenant_id, modelo_id, profissional_id)
);
```

Definição de campo (JSON validado por schema na publicação):

```json
{
  "id": "mcmurray", "rotulo": "Teste de McMurray", "tipo": "selecao",
  "opcoes": ["Positivo", "Negativo", "Não realizado"], "obrigatorio_ao_finalizar": true,
  "exibir_se": null,
  "vinculos": [
    { "quando": "Positivo", "sugerir_cid": "<CID_…>" },
    { "quando": "Positivo", "evidencia_dut": { "tuss": "<TUSS_…>", "criterio": "exame físico compatível" } }
  ]
}
```

`# ponytail:` escores e cálculos com uma linguagem de expressão mínima (soma, média, faixas) avaliada no cliente e revalidada no servidor; sem motor de fórmulas genérico.

### API (módulos `prontuario`, `documentos`, `modelos`)

| Método | Rota | Uso |
| :--- | :--- | :--- |
| `POST` | `/api/v1/atendimentos` | Avulso (o da agenda vem de `/agendamentos/{id}/iniciar`) |
| `GET` | `/api/v1/atendimentos/{id}` | Atendimento + última versão + documentos (gera log de acesso) |
| `PUT` | `/api/v1/atendimentos/{id}/evolucao` | `{ "versao_base": 3, "conteudo": {...}, "autoria": [...] }` → 409 se `versao_base` desatualizada |
| `POST` | `/api/v1/atendimentos/{id}/fechar` | Valida e publica `AtendimentoFechado` |
| `POST` | `/api/v1/atendimentos/{id}/adendos` | Só após fechado |
| `GET` | `/api/v1/pacientes/{id}/linha-do-tempo?tipo=` | Linha do tempo |
| `POST/PATCH` | `/api/v1/atendimentos/{id}/documentos` | Cria/edita rascunho |
| `POST` | `/api/v1/documentos/{id}/enviar` | WhatsApp/e-mail (exige `revisado` + consentimento) |
| `GET/POST` | `/api/v1/modelos`, `/api/v1/modelos/{id}/versoes` | Biblioteca e construtor |
| `POST` | `/api/v1/modelos/{id}/versoes/{v}/publicar` | Publica |

### Eventos

- `EvolucaoVersionada` (a cada versão; a UI usa para "Salvo automaticamente 09:42 · versão 3").
- `AtendimentoFechado { atendimento_id, pedidos: [{documento_id, tuss, qtd}], convenio: bool }` → `faturamento` cria a guia (`GuiaRascunhoCriada`) → `spike` analisa (RFC-PEP-005).
- `DocumentoAssinado` (RFC-PEP-011).

### Regras

- Fechar exige obrigatórios do modelo e ao menos um CID quando houver pedido de convênio (alerta, não bloqueio, se a clínica configurar).
- Indicação clínica do pedido de exame: contador e bloqueio acima de 500 caracteres (mesmo componente do Campo 23 na RFC-PEP-005).
- Conflito de edição (duas abas ou offline): a versão do servidor vence; o texto local vira **rascunho de adendo** mostrado ao autor, nunca é descartado em silêncio (ADR-PEP-0015).
- Sugestão aceita da Spike que altera a evolução (ex.: "inserir citação da radiografia") grava autoria `ia_sugerida`/`ia_editada` com `analise_id`.

## UI

- Tela 05: linha do tempo (esq.), editor clínico **Tiptap** (ProseMirror headless, estilizado com o CSS do design) com menu `/` no cursor e Mark de autoria `humano × IA` por trecho (centro), painel Tissia IA (dir., componente da RFC-PEP-005). Atalhos `V` (escuta, P2) e `Ctrl+Enter` (finalizar).
- Telas 09/10/11: construtor com arrastar e soltar via `@dnd-kit/core` + `@dnd-kit/sortable` (acessível por teclado e toque, RFC-PEP-015), propriedades do campo, pré-visualização ao vivo avaliando escores/condições de `@pep/contracts`; biblioteca com duplicar/versionar/compartilhar; modelo preenchido com escores calculados e mapa corporal anotável (imagem + marcadores em JSON).
- Tela 08: quatro cartões de documento; "Gerar guia SP/SADT" a partir do pedido de exame.
- Tela 07 (parcial, P1): visão lado a lado da evolução com trechos de IA marcados para decisão. A assinatura ICP-Brasil e a transcrição ficam para a P2 (RFC-PEP-011, RFC-PEP-013).
- Telas 05 e 08 já ligadas à API (`apps/ui/src/pages/Atendimento.tsx`, `Documentos.tsx`); 09–11 seguem estáticas em `apps/ui/src/modulos/modelos/`.

## Segurança e LGPD

- Conteúdo clínico nunca em log; logs só com ids e versões.
- Faturista: leitura do prontuário restrita ao necessário para a guia (perfil configurável; design: "somente leitura no prontuário").
- Cache offline cifrado no dispositivo (chave derivada da sessão), expira em 12 h sem login **[validar com segurança]**.
- Resumo ao paciente e atestado com CID só com consentimento registrado (RFC-PEP-002).

## Critérios de aceite

- [ ] `UPDATE`/`DELETE` em `evolucao_versoes` falha no banco (trigger), inclusive para o owner da aplicação.
- [ ] Salvar com `versao_base` antiga → 409 e o cliente oferece adendo/merge manual.
- [ ] Fechar atendimento com pedido de exame de convênio publica exatamente um `AtendimentoFechado` e uma `GuiaRascunhoCriada` (outbox, teste de reentrega).
- [ ] Modelo publicado não pode ser editado; atendimento antigo continua renderizando com a versão original.
- [ ] Indicação clínica > 500 caracteres bloqueada no cliente e rejeitada no servidor.
- [x] Edição offline de 10 min + reconexão sincroniza sem perda (E2E com rede simulada).
- [ ] Pacotes de Ortopedia e Cardiologia publicados e revisados pelo CMO.

## Perguntas abertas

1. _(Resolvida na RFC-PEP-015: `@dnd-kit/core` + `@dnd-kit/sortable` para o construtor e Tiptap headless para o editor clínico.)_
2. Receita: integrar com plataforma de prescrição digital (bulário, interações) ou bulário próprio mínimo?
3. Quais escores entram no pacote de Ortopedia (Lysholm, KOOS…) e de Cardiologia **[validar com CMO]**.

## Notas de implementação

**Marco D / D2 (08/11/2026): fatia entregue** ([[release-plan-pep#12-marco-d--mvp-de-demonstração-para-investidores|release plan §12]]).

- Entregue: `atendimentos`, `evolucao_versoes` (trigger append-only, testado até com o papel admin), `pedidos_exame`; `POST /api/v1/atendimentos {agendamentoId}` (inicia e abre o atendimento, idempotente), `GET /atendimentos/{id}`, `PUT .../evolucao` (409 se `versaoBase` desatualizada; conteúdo igual ao da última versão não gera versão), `POST/DELETE .../pedidos`, `POST .../fechar` (idempotente). Tela `/atendimento/:id` (SOAP, CID-10, pedido de exame com contador de 500 caracteres, `Ctrl+Enter` fecha).
- Fechar publica `AtendimentoFechado`, cria a guia SP/SADT em rascunho (`guias`, módulo `faturamento`) e `GuiaRascunhoCriada` quando há pedido de convênio, e leva o agendamento a `atendido`.
- **Desvios assumidos para a demo, a corrigir na P1:**
  - A rota de início é `POST /atendimentos` (e não `/agendamentos/{id}/iniciar`) para evitar dependência circular entre `agenda` e `prontuario`; `/agendamentos/{id}/iniciar` segue só mudando o estado.
  - A guia é criada **na mesma transação** do fechamento, chamando `FaturamentoService`, e não por consumidor do outbox (a infraestrutura de consumidores entra com a RFC-PEP-005). A atualização do agendamento para `atendido` roda em transação própria e é idempotente: refechar corrige falha parcial.
  - Pedido de exame fica em `pedidos_exame` (módulo `prontuario`), sem a tabela `documentos` genérica, sem receita, atestado ou encaminhamento. O catálogo TUSS é uma constante em `@pep/contracts` (`CATALOGO_TUSS_DEMO`, códigos e termos copiados de `tissia-ai/data/tuss/22_procedimentos.csv`); passa a vir da Spike na RFC-PEP-005.
  - Sem modelos por especialidade, `autoria` por trecho, adendos, linha do tempo, anexos, offline, log de acesso e Tiptap. O CID fica no conteúdo da evolução (`conteudo.cids`), sem alerta de "pedido de convênio sem CID".
  - Qualquer médico da unidade abre qualquer atendimento (sem restrição por profissional). O autosave salva 3 s após a digitação e ao sair do campo, o que gera mais versões que o necessário.
  - Em dev, o `usuarioId` padrão passou a ser um UUID fixo (`…00d0`), porque as versões da evolução gravam o autor.
  - Os casos do golden set da Spike ainda não estão no seed (entram no D3, junto com o adapter).

**Marco D / D2b (08/11/2026): prontuário fiel ao design (tela 05).** O D2 inicial foi refeito para seguir o design (`C05-Prontuario`) e a proposta de IA como diferencial.

- Tela `/atendimento/:id`: cabeçalho do paciente (idade, sexo, operadora e elegibilidade, tempo em atendimento), linha do tempo real (atendimentos, exames pedidos e guias, com filtros e problemas ativos), editor em blocos e painel Tissia IA.
- Editor: Tiptap 3 com schema próprio, no padrão do editor do knowee-v2 (sem StarterKit): `doc` com 4 blocos `soapSecao` (letra S/O/A/P e título), menu `/` por `@tiptap/suggestion` (`/receita`, `/exame`, `/atestado`, `/encaminhamento`), autosave 2 s e ao sair do campo, `UndoRedo`. A marca `ia` guarda a autoria por trecho (`ia_sugerida` vira `ia_editada` ao ser editada) e é persistida em `conteudo.autoria` (RF-PRO-07).
- `/exame` abre o pedido com TUSS, quantidade e indicação (≤ 500, com contador). `PATCH /atendimentos/{id}/pedidos/{pedidoId}` ajusta só a indicação.
- `GET /api/v1/pacientes/{id}/linha-do-tempo`.
- **Desvios:** `/receita`, `/atestado` e `/encaminhamento` ainda só avisam que dependem da tela 08 (documentos). O botão "Iniciar escuta" fica desabilitado até a escuta ambiental (telas 06 e 07). A aba "Modelo" e "Em uso" (medicações) ficam como prévia. Nenhuma sugestão da Spike insere texto no editor hoje; a marca `ia` e a autoria estão prontas para a escuta e a revisão.

**Marco D / D2c (08/11/2026): documentos do atendimento (tela 08).**

- Tela `/atendimento/:id/documentos` (design `C08-Documentos`): receita, pedido de exame, atestado, resumo ao paciente e encaminhamento (este só aparece ao usar `/encaminhamento`). Os comandos `/receita`, `/atestado` e `/encaminhamento` do editor salvam a evolução e abrem o cartão certo; o cabeçalho do prontuário ganhou o botão "Documentos".
- Tabela `documentos` (módulo `documentos`): um por tipo e atendimento, `rascunho → revisado → assinado`. Assinado é imutável por API (409) e por trigger (`UPDATE`/`DELETE`, inclusive para o admin). `DocumentoAssinado` vai ao outbox. `PUT /atendimentos/{id}/documentos/{tipo}`, `GET` da lista e `POST .../documentos/assinatura` (um, vários ou todos os pendentes).
- Receita confere cada item contra as alergias do paciente (`pacientes.alergias`, novo): conflito bloqueia a assinatura (409) e aparece em vermelho. Atestado: CID só com a autorização expressa registrada; afastamento exige dias; o texto da pré-visualização vem da mesma função no cliente e no servidor (`textoAtestado`).
- **Desvios:** assinatura eletrônica **simples** (sem ICP-Brasil, RFC-PEP-011); receita só "Simples" (controle especial e antimicrobiano ficam desabilitados); sem PDF nem impressão e sem envio por WhatsApp (botões desabilitados, com a razão); sem bulário (alergia por busca de texto no nome do medicamento); uma receita por atendimento; o "Aceitar indicação" do pedido registra a decisão na análise da Spike; "Gerar guia SP/SADT" só informa, porque a guia nasce ao finalizar o atendimento; a geração do resumo por IA entra com a escuta (telas 06 e 07).

**Onda 3 do MVP (10/11/2026): adendo, avulso, "quem viu", documentos por paciente, impressão/PDF e modelos por especialidade** ([[release-plan-pep#13-plano-do-mvp-funcional-ondas|release plan §13]]). O **offline (ADR-PEP-0015, RF-PRO-10)** ficou para a próxima parte da onda.

- **Adendo (RF-PRO-03):** tabela `adendos` (migração `0013`, trigger append-only como a evolução), `POST /atendimentos/{id}/adendos` só com o atendimento fechado (409 antes), texto e justificativa (≥ 10 caracteres), autor e data do servidor; evento `AdendoRegistrado`; aparece no atendimento e na linha do tempo. Qualquer médico da unidade registra (ainda sem restrição ao autor do atendimento).
- **Atendimento avulso (RF-PRO-01):** `POST /atendimentos { pacienteId, particular? }`, do próprio profissional na unidade ativa; reaproveita o avulso ainda aberto do mesmo paciente. Botão "Atendimento avulso" na ficha.
- **Quem viu (RF-PRO-09):** `GET /atendimentos/{id}/acessos` agrega `prontuario.visualizado` da trilha (uma linha por pessoa, último acesso e total). Visível a quem tem `prontuario:escrever`.
- **Documentos por paciente:** `GET /pacientes/{id}/documentos` (sem conteúdo clínico) e aba "Documentos" da tela 04.
- **PDF e impressão (RF-DOC-06):** página A4 `/atendimento/{id}/documentos/{tipo}/imprimir`; o PDF é o do navegador ("Salvar como PDF"), sem gerador no servidor nem `pdf_objeto`. Assinatura eletrônica simples; rascunho sai marcado "não assinado". O envio por WhatsApp segue desabilitado.
- **Modelos (RF-MOD-01 a 06):** módulo `modelos` (migração `0014`): `modelos`, `modelo_versoes` (publicada é imutável por trigger; só pode ser arquivada) e `modelo_compartilhamentos`; `atendimentos.modelo_versao_id`. API: `GET/POST /modelos`, `GET /modelos/{id}`, `POST /modelos/{id}/duplicar`, `PATCH /modelos/{id}` (ativar/desativar), `POST /modelos/{id}/versoes`, `PUT /modelos/{id}/versoes/{v}`, `POST …/publicar`, compartilhamentos e `GET /modelos/disponiveis`; `PUT /atendimentos/{id}/modelo`. A definição (seções, campos, condição `exibirSe`, escore soma/média com faixas, vínculos de faturamento) é validada em `@pep/contracts`: o rascunho aceita estrutura incompleta, a publicação exige referências corretas. Funções puras (`valorDoEscore`, `campoVisivel`, `obrigatoriosPendentes`, `sugestoesDoModelo`) rodam no cliente e no servidor. Fechar exige os obrigatórios visíveis do modelo (422 com a lista). A evolução guarda `modelo: { versaoId, campos }` (snapshot com rótulos), então o atendimento antigo renderiza com a versão que usou e os valores entram no `texto_plano` e na análise da Spike.
- UI: Biblioteca (`/modelos`, tela 10) com filtro, duplicar, compartilhar, ativar/desativar e "nova versão"; Construtor (`/modelos/:id`, tela 09) com seções e campos reordenáveis por `@dnd-kit` (mouse, toque e teclado), propriedades, vínculos, desfazer (`Ctrl Z`) e pré-visualização ao vivo; aba "Modelo" do prontuário (tela 11) com formulário, escores calculados e sugestões. "Modelos" entrou no menu (grupo Plataforma): o design só alcança a biblioteca por Configurações.
- **Desvios e pendências:**
  - **Catálogo por clínica:** os pacotes (Ortopedia: joelho e coluna lombar; Cardiologia: consulta e risco) são copiados para cada clínica na primeira leitura da biblioteca, em vez de `tenant_id NULL` com policy própria, para manter a RLS uniforme. São **estrutura de exemplo**: sem vínculo de CID/TUSS e sem escores validados (nenhum código inventado), marcados "Aguardando revisão do CMO" até a curadoria (pergunta 3 segue aberta). Vínculos aceitam `<CID_…>`/`<TUSS_…>` como marcador e a sugestão só vira ação com CID válido.
  - **Tipos de campo:** texto, seleção, escala 0–10, escore (soma ou média) e data. Tabela, cálculo e imagem anotada (mapa corporal) ficam inertes no construtor, com o motivo; escores como o Lysholm (itens com pesos próprios) pedem a curadoria e o cálculo.
  - O construtor adiciona campos por clique na seção ativa e reordena por arrastar; arrastar da paleta e mover campo entre seções ainda não. Sem blocos prontos (SOAP, sinais vitais) nem importar modelo.
  - Compartilhar é por profissional (ids da unidade); só o autor edita. O admin vê todos, mas não edita.
  - Sem `autoria` por trecho nos campos do modelo e sem versão de modelo "em uso por N profissionais".

**Offline do prontuário (10/11/2026, ADR-PEP-0015, RF-PRO-10):** fila local da evolução em `apps/ui/src/offline/cofre.ts`; reenvio em `Atendimento.tsx`.
  - **Comando:** `PUT /atendimentos/:id/evolucao` aceita `comandoId` (UUID do cliente) e `offline: { registradoEm }`. Reenvio com o mesmo `comandoId` devolve o resultado anterior (índices únicos parciais em `evolucao_versoes` e `adendos`, migração 0015). Comando offline com `versaoBase` antiga, ou com o atendimento já fechado, **nunca sobrescreve**: vira adendo `origem = offline`, com a hora do dispositivo e a do servidor na justificativa; a resposta traz `adendoId` e a UI recarrega a versão do servidor. Sem a marca `offline`, o 409 de antes continua.
  - **Fila:** uma entrada por atendimento. A edição offline seguinte substitui o conteúdo (novo `comandoId`) e preserva a versão-base mais antiga ainda não confirmada, então a queda de rede gera no máximo um adendo, não um por autosave. Escrever sem rede mostra "Salvo neste dispositivo — sincroniza ao reconectar", bloqueia "Finalizar" ("aguardando conexão para finalizar") e tenta de novo no evento `online` e a cada 15 s.
  - **Segurança:** conteúdo cifrado com AES-GCM; no claro só ids. Apagado no logout e após 12 h sem uso. **Desvio da ADR:** a chave é não extraível e fica no próprio IndexedDB, não "derivada da sessão". O token de sessão só existe em memória, então uma chave derivada dele tornaria a fila ilegível depois de recarregar a página, o que descartaria a evolução em silêncio. Protege o dado em disco, não contra o JavaScript da própria origem; **validar com segurança** junto do prazo de 12 h.
  - **Service worker:** gerado pelo `vite-plugin-pwa`, registrado só no build de produção, com cache apenas do shell e dos assets (nenhuma resposta da API).
  - **Fora desta entrega:** rascunho de documento offline (`CriarRascunhoDocumento`); cache do atendimento e da linha do tempo para leitura offline. Abrir o app sem rede não é possível porque o login precisa do Keycloak e o token só existe em memória, então o offline vale para a página de atendimento já aberta. O erro de rede também fica sem toast próprio na hora de finalizar (mensagem genérica de "Sem conexão").
  - Testes: 3 de API (idempotência, base antiga, atendimento fechado), cifra do cofre (UI) e E2E `apps/e2e/specs-demo/offline.spec.ts` com rede simulada.

**Documentos, linha do tempo e demo de dados fictícios (19/11/2026, PR #30 do `tissia`, em rascunho).** Ver [[demo-dados-ficticios]].

- **Desvio fechado:** o bloco "Em uso (medicações)", que estava como prévia, virou **"Receitado nos últimos 90 dias"** (`LinhaDoTempo.tsx` e a nova aba "Histórico clínico" da ficha do paciente). Fonte: `GET /api/v1/pacientes/:id/documentos/em-uso` (`documentos.controller.ts`; permissão `prontuario:ler`; schema `EmUsoSchema` em `@pep/contracts`). Considera receitas **assinadas** dos últimos 90 dias, um item por medicamento (o mais recente), e omite cursos de duração explícita já encerrados (`duracaoExplicitaDias`: reconhece "por/durante N [a N] dias"; posologia em semanas ou meses conta como em uso).
- **Auditoria:** cada chamada grava `documento.visualizado` (recurso `paciente`) antes da leitura, só quando o paciente tem atendimentos. A linha do tempo chama o endpoint a cada versão da evolução, então abrir ou recarregar a tela gera uma linha de auditoria (ponto de revisão futura: deduplicar).
- **Outros ajustes:** `pendentes()` ignora o `resumo_paciente` (não se assina); o atestado nasce com data e horário do atendimento; o resumo da linha do tempo e do adendo usa `resumir()` (limite de 90 caracteres, com "…"); `textoModelo` e `textoPlano` foram movidos para `@pep/contracts` (`textoModeloEvolucao`, `textoPlanoEvolucao`) para o seed montar o `texto_plano` do mesmo jeito.
- **Seed `clinico`:** 93 atendimentos do elenco com evolução versionada, adendos, receitas e atestados (assinados e rascunho) e 4 modelos da clínica. Cobre os casos do roteiro (Carla com 2 versões e adendo; Ana com infiltração do joelho E).

**Indicação clínica única por atendimento (19/11/2026, status: em implementação).** Ver [[plano-guia-indicacao-unica-e-autopreenchimento|plano]] e [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]].

- **Já entregue na fundação (`feat/guia-fundacao`, commit `dc4ae7b`):** coluna `atendimentos.indicacao_pedidos` (`text`, `CHECK` 1..500, migração `0047`), contratos aditivos em `@pep/contracts` (`Atendimento.indicacaoPedidos`, `CriarPedidoExame.indicacaoClinica` opcional, `DefinirIndicacaoPedidosSchema`) e *fallback* de leitura `a.indicacaoPedidos ?? pedidos[0]?.indicacaoClinica ?? null` em `ProntuarioService.montar` (sem *backfill* SQL por causa do RLS forçado).
- **Implementado (fatias G1 e G2; validado em 20/11/2026: API 437/437 testes e e2e:demo 42/42):** rota `PUT /api/v1/atendimentos/{id}/indicacao-pedidos`, herança automática ao criar pedido sem `indicacaoClinica` (mantendo cópia em `pedidos_exame.indicacao_clinica`), propagação ao editar pedido, preenchimento do Campo 23 da revisão 1 em `ProntuarioService.fechar`, leitura da indicação do atendimento no envio à Spike (sem alterar o contrato HTTP `indicacao_clinica`) e unificação da UI nas telas 05a–08 (um campo de indicação por atendimento; pedidos só com exame e quantidade).
