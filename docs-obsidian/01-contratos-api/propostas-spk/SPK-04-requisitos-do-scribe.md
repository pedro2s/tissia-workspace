---
projeto: ecossistema
tipo: spk
status: ativo
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-04-requisitos-do-scribe.md
---

# SPK-04 — Requisitos do PEP para o Scribe

**Status:** Implementada em parte (08/11/2026: pós-consulta em trechos, `consultas/transcrever` e `consultas/estruturar`; streaming pendente; desvios nas notas da Spike RFC-009) · **Destino:** repo tissia-ai, como insumo para detalhar a [[RFC-IA-009-scribe|Spike RFC-009]] · **Necessária para:** telas 06 (voz ambiental), 07 (revisão pós-consulta), 08 (resumo ao paciente), [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]], P2

## Objetivo

Registrar o que o PEP precisa do Scribe para que a Spike RFC-009 seja detalhada com um consumidor real. O ASR fica na Spike, como módulo vendável à parte ([[ADR-PEP-0013-asr-na-spike|ADR-PEP-0013]]).

## Motivação

A Spike RFC-009 está em rascunho e não tem endpoint. O design já tem as telas: escuta ambiental com locutores separados, conversa casual descartada, modelo da especialidade preenchido ao vivo com o trecho de origem de cada campo (06); transcrição e prontuário lado a lado com cada trecho de IA decidido antes da assinatura (07); resumo em linguagem simples para WhatsApp (08).

## Contrato proposto

### Etapa 1 — `POST /v1/consultas/estruturar` (pós-consulta)

Entrada: transcrição já diarizada (do streaming da etapa 2 ou de outro ASR).

```json
{
  "clinica_ref": "un_7f3a…",
  "atendimento_ref": "at_55d1…",
  "especialidade": "ortopedia",
  "modelo": { "id": "ortopedia_joelho", "versao": 3, "campos": [ { "id": "queixa_principal", "rotulo": "Queixa principal", "tipo": "texto" }, { "id": "testes_especiais", "rotulo": "Testes especiais", "tipo": "lista", "opcoes": ["McMurray", "Lachman", "Gaveta anterior"] } ] },
  "paciente": { "ref": "…", "sexo": "F", "idade": 54 },
  "transcricao": [
    { "inicio_s": 12.0, "fim_s": 16.4, "locutor": "medico", "texto": "Bom dia. Como está o joelho desde a última consulta?" },
    { "inicio_s": 18.0, "fim_s": 29.5, "locutor": "paciente", "texto": "Continua doendo há uns quatro meses…" }
  ],
  "saidas": { "soap": true, "modelo": true, "documentos": ["pedido_exame", "receita", "atestado"], "resumo_paciente": true, "analisar_guia": true }
}
```

- `modelo` vem do PEP (o construtor de modelos é do PEP, tela 09). A Spike não guarda modelos de clínica.
- `locutor`: `medico` | `paciente` | `acompanhante` | `outro`.
- `analisar_guia: true` encadeia o motor anti-glosa sobre o resultado, com a mesma resposta de `analisar` em `analise`.

Resposta (resumo):

```json
{
  "estruturacao_id": "…",
  "soap": { "S": { "texto": "…", "origens": [ { "inicio_s": 18.0, "fim_s": 29.5 } ] }, "O": { "…": "…" }, "A": { "…": "…" }, "P": { "…": "…" } },
  "campos": [ { "id": "queixa_principal", "valor": "Dor em joelho direito há ~4 meses…", "origens": [ { "inicio_s": 18.0, "fim_s": 29.5 } ], "gerada_por_ia": true, "status": "preenchido" } ],
  "trechos_descartados": [ { "inicio_s": 125.0, "fim_s": 220.0, "motivo": "conversa_nao_clinica" } ],
  "documentos": [ { "tipo": "pedido_exame", "procedimentos": [ { "codigo_tuss": "<TUSS_…>", "termo": "…" } ], "indicacao_clinica": "…", "gerada_por_ia": true } ],
  "resumo_paciente": { "texto": "…", "nivel_leitura": "simples", "gerada_por_ia": true },
  "analise": { "…": "mesma resposta de /v1/guias/analisar" },
  "degradado": false,
  "aviso_legal": "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."
}
```

Requisitos:

| # | Requisito |
| :--- | :--- |
| R1 | Todo campo preenchido tem `origens` (intervalos da transcrição). Campo sem origem fica `status: "pendente"`, nunca inventado (Spike ADR-0008) |
| R2 | Conversa não clínica volta em `trechos_descartados` e não alimenta nenhum campo |
| R3 | Medicação e posologia: só sugerir se dita explicitamente. O Thiago observou que receita por clique em protocolo é mais rápida (Spike RFC-009); medir no piloto |
| R4 | Códigos TUSS/CID só das bases vigentes (Spike ADR-0006) |
| R5 | `resumo_paciente` sem jargão, sem diagnóstico não confirmado pelo médico, ≤ 1.000 caracteres |
| R6 | Latência p95 ≤ 8 s para consulta de 15 min **[validar com eval]** |
| R7 | Degradação: IA fora → `200`, `degradado: true`, campos `pendente` |

Feedback: `POST /v1/consultas/estruturacoes/{id}/feedback` com `item_id = "campo:<id>"`, `"soap:<S|O|A|P>"`, `"doc:<n>"`, `"resumo_paciente"` e as mesmas `acao`. A tela 07 exige decisão por trecho antes de assinar.

### Etapa 2 — streaming (voz ambiental)

`GET /v1/consultas/stream` com upgrade para WebSocket. Autenticação: token de sessão de curta duração emitido por `POST /v1/consultas/sessoes` (chamado pelo backend do PEP com a API key; o navegador nunca vê a chave).

| Mensagem | Direção | Conteúdo |
| :--- | :--- | :--- |
| `inicio` | PEP → Spike | `atendimento_ref`, `especialidade`, `modelo`, formato de áudio (Opus 16 kHz mono), `consentimento_ref` |
| áudio | PEP → Spike | Frames binários |
| `marcar` / `pausar` / `retomar` | PEP → Spike | Controles da tela 06 |
| `transcricao_parcial` / `transcricao_final` | Spike → PEP | Segmento com `locutor`, tempos, texto |
| `campo_atualizado` | Spike → PEP | Mesmo formato de `campos[]` |
| `trecho_descartado` | Spike → PEP | Intervalo e motivo |
| `fim` | PEP → Spike | Encerra; Spike responde com a `estruturacao` completa (etapa 1) |

- Sem `consentimento_ref`, a sessão é recusada. O consentimento (verbal + termo digital) é registrado e guardado no PEP; a Spike só guarda a referência.
- **Zero retenção de áudio:** processado em memória e descartado (PRD v2 da Spike §5). Transcrição: não gravada na Spike, salvo o opt-in da ADR-0007.
- ASR em região Brasil. Candidatos e critério (WER pt-BR, latência, residência) na Spike RFC-009.

## Impacto no modelo de dados da Spike

Aditivo: `estruturacoes` (id, integrador, clínica, atendimento_ref, resposta sem texto clínico bruto, versões de modelo/ASR, latências), `estruturacoes_feedbacks`, `sessoes_stream` (token, expiração, consentimento_ref).

## LGPD e segurança

- Transcrição passa pelo scrubber antes de qualquer LLM (Spike ADR-0007).
- Voz é dado biométrico em potencial **[validar enquadramento LGPD]**: zero retenção é requisito, não otimização.
- Token de sessão: expira em 2 h, vinculado a um `atendimento_ref`.

## Compatibilidade

Rotas novas. O motor anti-glosa é reutilizado sem mudança de contrato.

## Critérios de aceite (do ponto de vista do PEP)

- [ ] Nenhum campo com valor e sem `origens` no golden set de consultas.
- [ ] Conversa casual do golden set não aparece em nenhum campo.
- [ ] Áudio não é persistido (teste de infraestrutura + revisão de buckets/logs).
- [ ] Sessão sem `consentimento_ref` é recusada.
- [ ] Reconexão do WebSocket retoma a sessão sem perder segmentos finais.

## Perguntas abertas e decisões encaminhadas (15/11/2026)

1. **O PEP Tissia usa a etapa 2 (streaming) desde o início ou começa com trechos + etapa 1?**
   - **Decisão encaminhada:** manter **trechos de ~12 s via HTTP (`POST /v1/consultas/transcrever` + `POST /v1/consultas/estruturar`)** no piloto/MVP (conforme desvio já aceito na [[ADR-PEP-0013-asr-na-spike|ADR-PEP-0013]]), adiando o WebSocket da etapa 2 para quando houver áudio real pt-BR e demanda comercial (extensão Chrome F5 / telemedicina P3).
   - **Condições para destravar a etapa 2:**
     - **STT:** coletar métricas de erro (edições humanas na revisão) e vocabulário médico real no piloto antes de rodar benchmark comparativo entre Deepgram, Whisper self-hosted no GCP SP e Google Chirp, mantendo residência de dados no Brasil como critério eliminatório;
     - **Jurídico:** manter **zero retenção de áudio** (processamento em memória e descarte imediato) e exigência de `consentimento_ref` até parecer formal sobre enquadramento de voz como dado biométrico (LGPD);
     - **Infraestrutura:** manter Cloud Run no piloto; reavaliar GKE apenas quando o WebSocket de longa duração e/ou GPU para Whisper forem necessários.
2. **Telemedicina (P3):** o áudio vem da chamada de vídeo do PEP. Mesmo canal de streaming? *(Em aberto para P3.)*
3. **Preço do Scribe separado do anti-glosa:** ver [[PRD-002-spike-standalone|PRD-002]].
