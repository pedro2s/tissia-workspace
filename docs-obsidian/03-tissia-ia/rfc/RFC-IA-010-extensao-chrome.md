---
projeto: tissia-ia
tipo: rfc
status: rascunho
tags:
  - tissia
  - tissia-ia
  - rfc
  - especificacao
aliases:
  - RFC-010-extensao-chrome
origem: tissia-ai/docs/rfc/RFC-010-extensao-chrome.md
---

# RFC-010 — Extensão Chrome "Universal Autofill"

**Fase:** F5 · **Status:** Rascunho

## Escopo (RF10 do PRD v2.0)

- Extensão Manifest V3 que captura o áudio da aba (telemedicina) ou do microfone, usa o Scribe streaming (RFC-009) e preenche os campos do PEP de terceiros (iClinic, Feegow, Amplimed, HiDoctor, MV, Tasy) com 1 clique.
- **Mapeamento de telas:** um seletor por campo e por sistema, em arquivo de configuração versionado e servido pela API (atualizável sem publicar nova versão da extensão). Ideia da reunião: um modo de "aprender" o formulário (o usuário aponta o campo uma vez).
- Mostra também os alertas anti-glosa (RFC-002) sobre o que foi preenchido.
- Canal de aquisição: plano Pro R$ 497/mês; funil para migrar ao PEP completo da JV.

## Riscos a avaliar antes de detalhar

- Termos de uso dos PEPs de terceiros sobre automação de UI.
- Fragilidade de seletores DOM: cada atualização do PEP alheio quebra o mapeamento. Precisa de monitoramento e de um fallback para copiar e colar.
- Revisão da Chrome Web Store para extensões com captura de áudio e acesso amplo a sites (permissões mínimas por domínio).
- Autenticação do médico individual (hoje a API só conhece integradores). Exige auth de usuário final + billing.
