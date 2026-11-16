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
origem: tissia/docs/spike/SPK-03-processamento-assincrono.md
---

# SPK-03 — Jobs assíncronos e webhooks

**Status:** Implementada na `tissia-ia` (`app/api/jobs.py`, `app/jobs.py`, `app/api/preditivo.py` e `app/api/desfechos.py` + testes em `tests/test_spk03_jobs_e_webhooks.py`: `Prefer: respond-async`, `Idempotency-Key`, `GET /v1/jobs/{id}` e webhooks `X-Tissia-Assinatura`); falta adotar no `PEP` (chamadas com `Prefer: respond-async` / `Idempotency-Key` em `tissia-ai.client.ts` e endpoint receptor de webhook) · **Necessária para:** ingestão de XML grande, auditoria de contratos, guias esquecidas e desfechos em lote ([[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]], [[RFC-PEP-008-contratos|008]], [[RFC-PEP-009-receita-esquecida|009]]), P2

## Objetivo

Oferecer uma variante assíncrona das rotas pesadas, com acompanhamento por `GET` e notificação por webhook assinado.

## Motivação

- `POST /v1/tiss/lotes/ingerir` aceita até 5.000.000 caracteres e processa tudo na requisição ([[manual-integracao-tissia-ia-v1|integracao.md §5.4]]). Um demonstrativo grande pode passar do timeout do Cloud Run e do cliente.
- `contratos/auditar` (até 2.000 itens), `guias/esquecidas` (até 2.000) e `desfechos` (até 1.000) são rotinas de fundo no PEP. Não há motivo para segurar a conexão.
- Spike ADR-0005 já previa callback para análises de lote na F3.
- Hoje não há idempotência explícita: reenviar o mesmo XML após timeout pode duplicar `historico_guias_tiss` **[validar no código se há deduplicação por `lote_ref`]**.

## Contrato proposto

### Disparo

As quatro rotas aceitam o header `Prefer: respond-async`. Com ele:

```http
POST /v1/tiss/lotes/ingerir
Prefer: respond-async
Idempotency-Key: un_7f3a:lote:PROT-2026-991
```

```json
202 Accepted
Location: /v1/jobs/4b8e…
{ "job_id": "4b8e…", "status": "na_fila" }
```

Sem o header, o comportamento síncrono atual continua igual.

- `Idempotency-Key` (até 200 caracteres): mesma chave + mesmo corpo em 7 dias → devolve o mesmo `job_id`. Mesma chave com corpo diferente → `409`.
- Limite do corpo na versão assíncrona: 20 MB (XML). A versão síncrona mantém os limites atuais.

### Acompanhamento

`GET /v1/jobs/{job_id}`

```json
{
  "job_id": "4b8e…",
  "tipo": "tiss_lote_ingerir",
  "status": "concluido",
  "criado_em": "2026-11-07T10:00:00Z",
  "concluido_em": "2026-11-07T10:00:42Z",
  "resultado": { "tipo_mensagem": "demonstrativo_retorno", "versao_tiss": "4.01.00", "lote_ref": "PROT-2026-991", "registro_ans": "005711", "itens_ingeridos": 68, "desfechos_vinculados": 39 },
  "erro": null
}
```

`status`: `na_fila` | `processando` | `concluido` | `falhou`. `resultado` tem o mesmo schema da resposta síncrona da rota. `erro`: `{ "codigo": "xml_invalido", "mensagem": "…" }`. Resultados ficam disponíveis por 7 dias.

### Webhooks

`PUT /v1/integrador/webhook` com `{"url": "https://api.pep…/webhooks/spike", "eventos": ["job.concluido", "job.falhou"]}`. A resposta devolve o segredo `whsec_…` uma única vez.

Entrega:

```http
POST https://api.pep…/webhooks/spike
X-Tissia-Assinatura: t=1730980842,v1=<hex HMAC-SHA256(segredo, t + "." + corpo)>
X-Tissia-Evento: job.concluido
X-Tissia-Entrega: <uuid>
```

```json
{ "evento": "job.concluido", "job_id": "4b8e…", "tipo": "tiss_lote_ingerir" }
```

- O corpo não traz o resultado; o PEP busca em `GET /v1/jobs/{id}`. Assim nenhum dado sai por um canal com menos controle.
- Retry com backoff exponencial por 24 h em respostas não-2xx. O PEP rejeita assinatura com `t` mais velho que 5 min (replay).
- `POST /v1/integrador/webhook/teste` envia um evento `ping`.

## Impacto no modelo de dados da Spike

Migração aditiva: `jobs` (id, integrador_id, clinica_id, tipo, status, idempotency_key, hash do corpo, resultado jsonb, erro jsonb, timestamps), índice único `(integrador_id, idempotency_key)`; `webhooks` (integrador_id, url, eventos, segredo cifrado); `webhooks_entregas` (tentativas). Worker: Cloud Tasks ou Cloud Run Jobs em `southamerica-east1` (Spike ADR-0003). `# ponytail:` uma fila só; separar por tipo se houver contenção.

## LGPD e segurança

- O corpo enfileirado (XML) é guardado cifrado só até o processamento e apagado em seguida. Tags de identificação direta continuam sem ser lidas (Spike RFC-008 §2).
- URL de webhook só HTTPS, sem IP privado/loopback (proteção contra SSRF), validada no cadastro e em cada entrega.
- Logs: só `job_id`, tipo, status, latência.

## Compatibilidade

Opt-in por header. Nenhuma rota existente muda de comportamento.

## Critérios de aceite

- [ ] Mesmo `Idempotency-Key` + mesmo corpo → mesmo `job_id`, sem processar duas vezes.
- [ ] `resultado` do job valida contra o schema da resposta síncrona.
- [ ] Assinatura HMAC verificável com o exemplo do manual; replay > 5 min rejeitado no exemplo de verificação.
- [ ] Webhook para `http://`, `localhost` ou IP privado → 422.
- [ ] XML de 15 MB processado sem timeout em `staging`.
- [ ] Corpo do job apagado após conclusão (teste).

## Perguntas abertas

1. `analisar` em lote (conferência do faturista com centenas de guias) entra aqui ou continua uma chamada por guia? Proposta: continua por guia; o PEP paraleliza com limite.
2. Retenção de 7 dias para resultados é suficiente para o PEP?
