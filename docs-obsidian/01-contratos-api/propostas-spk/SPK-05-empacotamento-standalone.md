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
origem: tissia/docs/spike/SPK-05-empacotamento-standalone.md
---

# SPK-05 — Empacotamento standalone

**Status:** Implementada na `tissia-ia` quanto ao contrato de API (`app/api/standalone.py`, `app/main.py` + testes em `tests/test_spk05_standalone.py`: `GET /v1/uso`, `GET/POST/DELETE /v1/integrador/chaves`, header `X-Tissia-Cota-Restante` e `429 Too Many Requests`); portal web de autoatendimento e governança comercial ficam fora do escopo da API clínica do `PEP` (no `PEP`, falta apenas tratar `X-Tissia-Cota-Restante` / `429` no client quando aplicável) · **Necessária para:** [[PRD-002-spike-standalone|PRD-002]] (venda da Spike para outros PEPs e sistemas de faturamento)

## Objetivo

Dar à Spike o mínimo para ser vendida a integradores que não são o PEP Tissia: autoatendimento, chaves seguras, medição de uso, limites, SLA, sandbox e documentação pública.

## Motivação

Hoje um integrador novo é criado por `scripts/criar_integrador.py`, as clínicas por script (ver [[SPK-01-api-de-clinicas-e-flags|SPK-01]]), não há medição de uso para cobrar, nem limite de taxa, nem ambiente de teste com dados sintéticos fora do `staging` interno. O manual de integração existe ([[manual-integracao-tissia-ia-v1|integracao.md]]), mas não está publicado.

## Escopo proposto

| Item | Proposta | Mínimo viável |
| :--- | :--- | :--- |
| Portal do integrador | Web com login (OIDC), cadastro da empresa, aceite de termos e DPA, gestão de chaves, webhooks, clínicas, uso e faturas | Começar com formulário + aprovação manual; portal completo quando houver > 5 integradores |
| Chaves | Uma por ambiente (`sandbox`, `prod`), prefixo indica o ambiente (`tsk_test_…`, `tsk_live_…`), várias ativas por vez para rotação sem downtime, expiração opcional, último uso visível | Rotação: criar nova → trocar → revogar antiga |
| Sandbox | Ambiente com bases e IA reais, para qualquer texto, com proibição contratual de dado real e catálogo de casos sintéticos publicado | Mesmo deploy do `staging`, integradores separados, dados apagados a cada 30 dias |
| Medição de uso | Contador por integrador, clínica, rota e mês: análises, análises com IA (justificativa/sugestão), estruturações (Scribe), itens de lote/contrato | Tabela `uso_mensal` alimentada na própria requisição |
| Limites | Por plano: requisições/s por chave e cota mensal. Excesso → `429 Too Many Requests` com `Retry-After` | Token bucket em memória por instância. `# ponytail:` limite por instância; mover para Redis/Memorystore se houver mais de 3 instâncias |
| Planos | Ver PRD-002 (por médico/mês, por guia analisada, ou pacote) | Cobrança fora do sistema no início (planilha + nota), a partir de `uso_mensal` |
| SLA | 99,5% mensal (Spike PRD-001 RNF-05), p95 publicados, créditos por indisponibilidade | Status page pública com uptime do `/healthz` |
| Documentação pública | Manual de integração, OpenAPI, changelog, coleção Postman, catálogo de casos sintéticos | Site estático gerado do repo |
| Termos | Termos de uso, DPA modelo (Spike como operadora ou suboperadora conforme o cliente), política de privacidade, lista de suboperadores (Google Cloud; Jev se ativado) | Revisão jurídica antes do primeiro cliente fora do PEP **[validar]** |

## Contrato proposto (adições à API)

| Rota | Uso |
| :--- | :--- |
| `GET /v1/uso?mes=2026-11` | Uso do integrador por clínica e rota |
| `GET /v1/integrador/chaves`, `POST /v1/integrador/chaves`, `DELETE /v1/integrador/chaves/{id}` | Rotação de chaves (exige chave com escopo `admin`) |
| Header `X-Tissia-Cota-Restante` | Em todas as respostas |
| `429` | `{"detail": "limite_excedido", "retry_after_s": 12}` |

Escopos de chave: `analise` (rotas de guia, feedback, desfecho), `admin` (clínicas, chaves, webhooks, uso). O PEP usa uma chave `analise` no backend de produção e uma `admin` só no job de onboarding.

## Impacto no modelo de dados da Spike

Aditivo: `integradores.plano`, `integradores.status` (`sandbox`, `ativo`, `suspenso`); `chaves_api` (n por integrador, escopos, expiração, último uso) substituindo a chave única atual; `uso_mensal`; `aceites_termos` (versão, data, IP).

## LGPD e segurança

- Cada cliente standalone assina DPA antes de `prod`. A Spike nunca recebe PII direta (contrato `extra="forbid"`), o que vale como argumento de venda.
- Chaves guardadas como hash (Spike ADR-0005), mostradas uma vez.
- Sandbox: cláusula contratual proibindo dado real + scrubber continua ativo.
- Pen test externo antes do primeiro cliente standalone em `prod`.

## Compatibilidade

A chave atual do PEP vira uma linha em `chaves_api` com escopos `analise` e `admin`. Nenhuma rota existente muda.

## Critérios de aceite

- [ ] Rotação de chave sem erro em requisições em andamento (duas chaves válidas ao mesmo tempo).
- [ ] Chave `analise` recebe 403 em rotas `admin`.
- [ ] `429` com `Retry-After` acima do limite do plano.
- [ ] `uso_mensal` bate com a contagem de `analises` do mês (teste).
- [ ] Manual, OpenAPI e changelog publicados e versionados.
- [ ] Integrador de sandbox não acessa `prod` e vice-versa.

## Perguntas abertas

1. Modelo de cobrança: por médico/mês (como o upsell na base do Thiago) ou por guia analisada? Afeta o que medir.
2. O PEP Tissia paga a Spike internamente (transfer price) ou é o mesmo P&L? Afeta a cota do PEP.
3. Precisamos de região/conta dedicada para clientes hospitalares grandes?
