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
  - 0007-minimizacao-de-dados
origem: tissia-ai/docs/adr/0007-minimizacao-de-dados.md
---

# ADR-0007 — Minimização de dados e retenção (LGPD)

**Status:** Aceita — 29/10/2026

## Contexto

O texto da evolução clínica é dado pessoal sensível (LGPD Art. 5º, II e Art. 11). O motor não precisa saber **quem** é o paciente, só o conteúdo clínico, o sexo e a idade (que algumas regras de compatibilidade CID usam).

## Decisão

1. **Minimização no contrato:** a API não tem campos para nome, CPF, RG, CNS, carteirinha, telefone ou endereço. O paciente aparece como `paciente_ref` (identificador opaco do integrador), `sexo` e `idade` (anos).
2. **Scrubber de texto livre** (`app/pii.py`) antes de qualquer chamada de IA: troca por marcadores CPF, CNS, RG, CRM/COREN, telefone, e-mail, CEP e datas completas (`[CPF]`, `[TELEFONE]`…). Para não destruir evidências temporais exigidas por DUTs (ex.: tempo mínimo de tratamento conservador), datas completas que não sejam de nascimento são convertidas em intervalo relativo à data da análise (`[DATA: há ~N dias]`), enquanto datas precedidas por contexto de nascimento (`DN`, `nasc.`, `nascimento`) viram `[DATA_NASCIMENTO]`. `# ponytail:` regex; nomes próprios em texto livre não são detectados na F1 (exceto prefixos nominais explícitos como `Paciente: …`). Upgrade: NER (ex.: spaCy pt) se a auditoria do golden set achar nomes com frequência.
3. **Logs:** nunca o corpo da requisição nem o texto da evolução. Apenas `analise_id`, `clinica_id`, códigos, latências e erros.
4. **Persistência:** a tabela `analises` guarda a resposta completa (alertas, códigos, versões), o hash SHA-256 do texto de entrada e, **só se a clínica tiver `guardar_entrada_para_avaliacao = true`** (opt-in do piloto, com respaldo contratual), o texto já passado pelo scrubber. Retenção: 180 dias para o texto; resposta e feedback, enquanto durar o contrato.
5. **Provedores de IA:** só com termos que proíbem treino com os dados enviados. Fora do Brasil, só com cláusulas-padrão ANPD (ver [[ADR-IA-0004-estrategia-de-modelos|ADR-0004]]).
6. Papéis (a confirmar com jurídico, Q6 do PRD): clínica = controladora; software house do integrador = operadora; Tissia IA = suboperadora. Um RIPD (Relatório de Impacto) precisa estar pronto antes da F2 receber dado real.

## Consequências

- Testes automáticos garantem que o scrubber cobre os padrões e que os logs não contêm o texto ([[test-plan-ia|test plan]]).
- Áudio (F4): zero retenção, conforme o PRD v2. Decisão a detalhar na RFC-009.
