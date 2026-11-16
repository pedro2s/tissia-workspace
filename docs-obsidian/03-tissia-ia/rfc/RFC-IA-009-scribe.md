---
projeto: tissia-ia
tipo: rfc
status: em-implementacao
tags:
  - tissia
  - tissia-ia
  - rfc
  - especificacao
aliases:
  - RFC-009-scribe
origem: tissia-ai/docs/rfc/RFC-009-scribe.md
---

# RFC-009 — Scribe (evolução clínica a partir da consulta)

**Fase:** F4 · **Status:** Em implementação (etapa pós-consulta em trechos entregue em 08/11/2026; streaming pendente)

## Escopo (RF01–RF05 do PRD v2.0)

Em duas etapas:

1. **Pós-consulta sobre a transcrição existente:** o PEP do Thiago já transcreve e diariza. Novo endpoint `POST /v1/consultas/estruturar` recebe a transcrição diarizada e devolve evolução SOAP/template por especialidade + documentos (prescrição, pedidos de exame com indicação clínica, atestado, resumo ao paciente), já chamando o motor anti-glosa sobre o resultado. Payload de saída = o JSON do PRD v2 §4. Não há áudio nessa etapa.
2. **Streaming próprio** (para a extensão F5 e PEPs sem transcrição): captura de áudio, STT de baixa latência com vocabulário médico BR (ANVISA, siglas como `BEG`, `LOTE`, `RCR 2T BNF`), diarização, filtro de small talk, eventos parciais via WebSocket (sugestão do Thiago: "jogo o stream e você vai mandando evento no socket").

## Decisões a tomar (futuras ADRs)

- STT: candidatos Deepgram, Whisper (self-hosted no GCP SP), Google Chirp (Vertex SP). Critério: WER em consultas pt-BR reais, latência, residência de dados.
- Áudio: zero retenção (processado em memória e descartado), conforme o PRD v2 §5.
- Templates por especialidade: YAML curado pelo CMO, como as DUTs.

## Ponto de atenção (da reunião)

O Thiago observou que, para receita, o médico costuma ser mais rápido clicando em protocolos do que ditando. Medir no piloto antes de investir em extração de prescrição por voz.

## Notas de implementação

**Etapa 1 do PEP (pós-consulta em trechos), 08/11/2026.** Implementados `POST /v1/consultas/transcrever` e `POST /v1/consultas/estruturar` (módulos `app/consultas.py`, `app/api/consultas.py`, `app/ia/vertex.py`), a pedido do PEP ([[SPK-04-requisitos-do-scribe|SPK-04]]).

- `transcrever` recebe **um trecho de áudio por chamada** (base64, até ~4,5 MB, `audio/webm` etc.), com os últimos segmentos como contexto para manter os locutores. Devolve segmentos com locutor, tempo (deslocado por `inicio_s`) e `clinico`. Áudio e texto não são gravados. Não é o WebSocket da etapa 2 do SPK-04: o PEP troca de trecho a cada ~12 s para dar a sensação de ao vivo.
- `estruturar` devolve `soap` (lista de trechos por seção, cada um com `origens`), `campos` do modelo, `trechos_descartados` e `resumo_paciente`. **Desvio do SPK-04:** `soap[S|O|A|P]` é uma **lista** de trechos, não um texto único, para a revisão decidir trecho a trecho. `documentos`, `analise` encadeada e feedback próprio entraram em 15/11/2026 (notas abaixo).
- R1 aplicado no servidor: trecho ou campo sem `origens` válidas é removido (campo vira `pendente`). O texto passa por `scrub` antes do Gemini.
- Sem credencial do Vertex, timeout ou erro: `200` com `degradado: true`. `transcrever` volta sem segmentos. `estruturar` cai em um classificador **local por palavras-chave** (`provedor: "local"`, sem IA), que a resposta declara.
- **`documentos` e `analise` encadeada, 15/11/2026.** `estruturar` aceita, aditivamente e no topo do request (no estilo de `resumo_paciente`, não em `saidas` como no SPK-04), `documentos: ["pedido_exame", "receita", "atestado"]`, `analisar_guia: bool` e `operadora` opcional. Cada documento volta com `origens` e `gerada_por_ia`; sem origem válida não é gerado (R1). `pedido_exame` reaproveita as `sugestoes` TUSS (só código vigente em `bases.tuss`), com origem nos segmentos em que a evidência foi dita, e a indicação clínica vem de `dut.gerar_e_validar_justificativa` (o mesmo gerador validado da guia SP/SADT, sem CID no texto) lendo só os segmentos citados pelo SOAP e pelos procedimentos. `receita` e `atestado` saem da mesma chamada do Gemini que monta o SOAP; item de receita só fica se o nome do medicamento está no texto dos segmentos citados (R3). Sem IA: só o `pedido_exame` heurístico, `degradado: true`. `analisar_guia` chama `analisar_guia` de `app/api/guias.py` (flags, modo sombra e gravação em `analises`) com a evolução = SOAP em texto e os procedimentos do pedido; sem `operadora`, `paciente` ou procedimento, `analise` fica nula. Limites: o pedido não distingue procedimento realizado de solicitado (a `sugestao` não traz o tipo) e a posologia não é conferida contra o texto, só o nome do medicamento.
- **Achado:** o motor anti-glosa atual (`app/ia/gemini.py`) ainda é heurístico (regex e templates); nenhuma chamada ao Vertex existia antes de `app/ia/vertex.py`. Em 09/11/2026 a extração de termos, a decisão dos critérios DUT e a justificativa SP/SADT passaram a usar o Gemini 3.x quando `IA_PROVEDOR=vertex` ([[ADR-IA-0010-vertex-global-e-gemini-3x|ADR-0010]]), com fallback heurístico.

**Persistência e feedback da estruturação, 15/11/2026.** `estruturar` devolve `estruturacao_id` e grava uma linha em `estruturacoes` (migração `0008`) com integrador, clínica, `atendimento_ref`, provedor, modelo, latência, `degradado` e `metadados` só com contagens (segmentos, trechos por seção SOAP, `status` por id de campo, descartados, se houve resumo e sugestões). Nenhum texto clínico é gravado (ADR-0007). A clínica segue a política de `analisar` (auto-provisiona, ou `404` com `exigir_clinica_cadastrada`/desativada); falha de banco é logada e não vira 5xx, como em `analisar`. `POST /v1/consultas/estruturacoes/{id}/feedback` espelha o feedback de análises, com isolamento por integrador e `item_id` validado (`campo:<id>`, `soap:<S|O|A|P>`, `doc:<n>`, `resumo_paciente`). **Desvios:** como `soap` é lista de trechos, `item_id` também aceita `soap:<S|O|A|P>:<índice>` para a decisão por trecho da tela 07; e `texto_final` (texto clínico editado pelo médico) só é gravado, após `scrub`, com o opt-in `guardar_entrada_para_avaliacao` da clínica. Em `analises` ele é sempre gravado após `scrub`.
