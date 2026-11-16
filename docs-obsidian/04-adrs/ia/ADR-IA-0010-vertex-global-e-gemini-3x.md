---
projeto: tissia-ia
tipo: adr
status: aceita
tags:
  - tissia
  - tissia-ia
  - adr
  - decisao-arquitetural
aliases:
  - 0010-vertex-global-e-gemini-3x
origem: tissia-ai/docs/adr/0010-vertex-global-e-gemini-3x.md
---

# ADR-0010 — Vertex AI no endpoint `global` e modelos Gemini 3.x

**Status:** Aceita — 09/11/2026
**Substitui parcialmente:** [[ADR-IA-0003-gcp-sao-paulo|ADR-0003]] (a linha "LLM" e a proibição do endpoint `global`). O restante do ADR-0003 continua valendo (API, banco, segredos, logs em `southamerica-east1`).

## Contexto

- O ADR-0003 exigia o LLM no endpoint regional `southamerica-east1` e proibia o `global`, para evitar a discussão de transferência internacional.
- Em 09/11/2026 nenhum modelo Gemini responde em `southamerica-east1` no projeto da Spike (404 em todos os 2.5 e 3.x testados). A [página de ciclo de vida do Vertex](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions) aposentou o `gemini-2.5-flash`, o `2.5-flash-lite` e o `2.5-pro` em 20/10/2026. Os modelos 3.x respondem pelo endpoint `global`.
- A LGPD (Lei 13.709/2018) **não exige** que o tratamento ocorra em território nacional. Ela regula a **transferência internacional** (arts. 33 a 36): país com grau de adequação, cláusulas contratuais padrão aprovadas pela ANPD (Res. CD/ANPD nº 19/2024), consentimento específico, entre outras hipóteses. Dado de saúde segue sendo dado sensível (art. 11) e o RIPD precisa descrever o fluxo.
- A Spike já minimiza o dado (ADR-0007): do paciente só vão `ref` pseudônima, sexo e idade, e o texto clínico passa por `app/pii.py` antes de qualquer provedor.

## Decisão

1. **O LLM pode usar o endpoint `global` do Vertex AI** (ou regiões e multirregiões `us`/`eu`), escolhido por `VERTEX_LOCATION`. O padrão passa a ser `global`. API, banco, segredos e logs continuam em `southamerica-east1`.
2. **Só modelos Gemini 3.x vigentes**, nunca modelo descontinuado ou com aposentadoria anunciada em menos de 6 meses (`gemini-3.6-flash`, por exemplo, não). Padrões: `GEMINI_MODEL=gemini-3.5-flash-lite` (decisão dos critérios e extração, ~1,3 s) e `GEMINI_MODEL_QUALIDADE=gemini-3.5-flash` (justificativa, transcrição e estruturação do Scribe, ~2,2 s). O `gemini-3.8-flash` foi medido em ~30 s para uma resposta trivial e fica de fora por ora.
3. A **minimização não muda**: nada de nome, CPF, CNS, carteirinha, telefone ou data de nascimento vai à Spike (ADR-0007), nem por ela ao Vertex.
4. **O áudio do Scribe** não pode passar pelo scrubber. Ele vai ao Vertex em memória, sem retenção na Spike e no PEP (ADR-PEP-0013), e entra no RIPD como tratamento por operador no exterior.

## Condições para produção (pendentes, não bloqueiam o desenvolvimento)

- DPO/jurídico validar a base legal e o mecanismo de transferência (cláusulas-padrão da ANPD no contrato do Google Cloud) **[validar]**, e atualizar o RIPD.
- Confirmar contratualmente que o Vertex não usa dado do cliente para treinar modelos **[validar]**.
- Se um contrato com operadora ou hospital exigir processamento no Brasil ou na UE, trocar `VERTEX_LOCATION` por uma região ou multirregião que o atenda. A Spike não depende do `global`.
- Revisar `GEMINI_MODEL*` a cada release: a página de ciclo de vida do Vertex é a fonte das datas.

## Consequências

- A Spike deixa de falhar no startup com `VERTEX_LOCATION=global`; o teste correspondente passa a verificar que o `global` é aceito e que região vazia é recusada.
- O CLAUDE.md e o README da Spike deixam de proibir o `global`.
- `IA_PROVEDOR=vertex` liga o Gemini; `local` mantém as heurísticas (CI e testes sem rede).
