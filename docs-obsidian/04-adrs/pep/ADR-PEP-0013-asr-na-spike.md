---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0013-asr-na-spike.md
---

# ADR-PEP-0013 — ASR/voz ambiental como módulo da Spike (Scribe), vendável à parte

**Status:** Aceita — 09/11/2026 (proposta em 07/11/2026; já implementada na fatia do Marco D. Desvio assumido: áudio em trechos por HTTP, não streaming; ver notas da [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]])

## Contexto

As telas 06 e 07 mostram escuta ambiental com consentimento, separação de locutores, descarte de conversa casual e preenchimento do modelo ao vivo, com trecho de origem por campo. A Spike tem o Scribe planejado ([[RFC-IA-009-scribe|Spike RFC-009]], rascunho): estruturar a transcrição e, depois, streaming próprio. A decisão em aberto nº 3 do design de produto é onde roda o ASR: dentro do PEP ou como módulo separado.

## Decisão

- **ASR, diarização e estruturação ficam na Spike (Scribe)**, como capacidade vendável à parte (extensão Chrome, outros PEPs), seguindo a mesma minimização de dados.
- **O PEP captura o áudio** (navegador/PWA, microfone da sala) após registrar o consentimento, envia em streaming à Spike e consome eventos parciais (transcrição diarizada, campos do modelo com trecho de origem).
- **Zero retenção de áudio** na Spike e no PEP: processado em memória e descartado. O PEP guarda só a transcrição revisada e a evolução, se o profissional aceitar.
- O PEP envia o **modelo de prontuário** (campos) para a Spike preencher; o modelo pertence ao PEP ([[RFC-PEP-004-prontuario|RFC-PEP-004]]).
- Contrato e requisitos do PEP em [[SPK-04-requisitos-do-scribe|SPK-04]]; integração na [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]] (P2).
- Enquanto o Scribe não existir, a tela 06 fica fora do MVP; ditado do navegador não é oferecido como substituto.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| ASR dentro do backend do PEP | Duplica o que a Spike vai ter e não vira produto vendável |
| Fornecedor de scribe pronto (SaaS) | Dado de saúde e áudio fora do nosso controle; sem integração com a anti-glosa |
| Ditado do navegador (Web Speech API) | Envia áudio a terceiros sem garantia de região; sem diarização |

## Consequências

- A P2 do PEP depende da F4 da Spike; o cronograma das duas precisa ser casado ([[release-plan-pep|release plan]]).
- Streaming exige WebSocket entre PEP e Spike; possível impacto na escolha Cloud Run × GKE ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009]]).

**Para aceitar:** time da Spike aceitar SPK-04 e o escopo de streaming na F4; escolha do STT (Spike: Deepgram, Whisper self-hosted, Chirp) com região Brasil; jurídico validar consentimento de gravação.
