---
projeto: tissia-ia
tipo: operacao
status: ativo
tags:
  - tissia
  - tissia-ia
  - operacao
  - test-plan
  - qualidade
aliases:
  - test-plan
  - test-plan-tissia-ia
origem: tissia-ai/docs/test-plan.md
---

# Test Plan — Tissia IA

**Escopo:** F0 e F1 em detalhe. F2+ no fim deste documento, em linhas gerais.

## 1. Estratégia

O comportamento de IA não é determinístico, então os testes ficam em duas camadas que não se misturam:

| Camada | O que garante | Onde roda | IA real? |
| :--- | :--- | :--- | :--- |
| **Testes automatizados** (`tests/`) | Contrato, regras, orquestração, degradação, segurança/LGPD | Todo PR | Não. Decisor e Gemini viram stubs |
| **Avaliação** (`evals/`, [[RFC-IA-005-harness-de-avaliacao\|RFC-005]]) | Qualidade clínica: recall/precisão DUT, top-k de códigos, fato não suportado, latência, custo | PR que toca IA/bases/regras + manual | Sim |
| **Validação humana** | Correção clínica e utilidade para a faturista | Fim da F1 (CMO) e F2 (piloto) | Sim |

Ferramentas: `pytest`, `httpx`/`TestClient` do FastAPI, Postgres de verdade como service container na CI (sem SQLite). Não há framework de mocks além de `monkeypatch` e stubs simples em `tests/stubs.py`.

## 2. Testes automatizados por RFC

| RFC | Casos obrigatórios |
| :--- | :--- |
| 000 — Fundações | 401/403; scrubber (CPF com e sem máscara, CNS, RG, CRM, telefone, e-mail, CEP, data de nascimento → `[DATA_NASCIMENTO]`, data clínica → `[DATA: há ~N dias]`; falsos positivos como códigos TUSS de 8 dígitos e CID **não** devem ser mascarados); **log sem PII**: request com CPF e evolução marcada → nenhuma linha do `caplog` contém os marcadores; `VERTEX_LOCATION` aceita `'global'` (padrão desde [[ADR-IA-0010-vertex-global-e-gemini-3x\|ADR-IA-0010]]) e falha no startup apenas se vazio |
| 001 — Bases | Ingestão idempotente; consistência DUT YAML × correlação × TUSS 22; todo DUT tem `revisado_por`; carga < 3 s |
| 002 — API | Snapshot do OpenAPI; 422 para campo extra (`nome`, `cpf`); 413 > 64 KB; matriz de degradação (DUT/justificativa/sugestão: ok, timeout, exceção), sempre schema válido e nunca 5xx por IA; modo sombra grava resposta completa no banco e mascara HTTP; feedback de outra clínica/integrador → 404; análise gravada com versões e `guia_ref` |
| 003 — Regras | Positivo/negativo/fora de vigência por cada um dos 8 tipos fechados; regras de operadora só para aquela operadora; YAML inválido falha no carregamento; performance (500 regras < 20 ms) |
| 004 — DUT/justificativa | Tabela de verdade da lógica de três valores + `condicao_estruturada`; uma chamada ao decisor por guia na avaliação de critérios; limiares; justificativa consolidada por guia ≤ N caracteres; fast-path determinístico + checagem de `fatos_usados` reprova fato inexistente; retry único e depois `null` |
| 005 — Harness | Guarda do OpenRouter com caso real → aborta antes de chamar |
| 006 — Sugestão | Propriedade: todo código devolvido existe e está vigente (stubs devolvendo lixo, código inventado, lista vazia); guia vazia; deduplicação com `acessorio_ausente`; `nenhum_dos_anteriores` não gera sugestão |

Cobertura: sem meta numérica. A regra é que todo ramo de `regras.py`, `dut.py` (lógica de três valores) e `codificacao.py` (cruzamento com a guia) tenha teste.

## 3. Avaliação (qualidade de IA)

Metas da F1 (golden set `sinteticos`, 60 casos):

| Métrica | Meta |
| :--- | :--- |
| Recall de critério DUT ausente | ≥ 85% |
| Precisão de alerta DUT | ≥ 80% |
| Taxa de `incerto` | ≤ 15% |
| Top-3 TUSS / top-3 CID | ≥ 80% / ≥ 80% |
| Recall@50 de candidatos | ≥ 95% |
| Justificativas com fato não suportado | 0 (automático) e 0 na amostra humana de 50 |
| Negação gerando sugestão | 0 |
| p95 (sem / com justificativa+sugestão) | ≤ 1,5 s / ≤ 5 s |
| Custo de IA por guia | registrar; alvo ≤ R$ 0,05 |

Regressão: falha se alguma métrica cair mais de 3 p.p. em relação ao último resultado em `main`.

## 4. Testes não-funcionais

| Tipo | Como | Quando |
| :--- | :--- | :--- |
| Carga | `locust` com 5 req/s por 5 min contra `staging`, mix 50% com justificativa+sugestão | Antes de cada release para `prod` na F1 |
| Resiliência | Derrubar o Jev/Gemini (URL inválida via config em `staging`) → API continua respondendo regras com `degradado: true` | Fim da F1 |
| Segurança | `pip-audit` na CI; checagem manual: `/docs` protegido em `prod`, chaves só no Secret Manager, IAM mínimo da service account | Fim da F0 e a cada release |
| LGPD | Revisão de amostra de logs de `staging` com payload contendo PII sintética → nenhuma ocorrência | Fim da F0 |

## 5. Validação humana (fim da F1, antes do piloto)

- **CMO:** revisa 50 análises do golden set (alertas, justificativas, sugestões). Critérios: clinicamente correto? O auditor da operadora aceitaria? A justificativa diz algo que não está na evolução? Registro em `docs/evals/AAAA-MM-DD-revisao-cmo.md`.
- **Consultor de faturamento / faturista do Thiago:** revisa 20 respostas como se estivesse montando a guia. Pergunta-chave: "você mudaria algo na guia por causa disso?"

## 6. Critérios de saída por fase

| Fase | Sai quando |
| :--- | :--- |
| F0 | Todos os critérios de aceite das RFC-000/001/005; golden set de 60 casos; tabela comparativa de modelos publicada |
| F1 | Critérios de aceite das RFC-002/003/004/006; metas da §3; validação humana da §5 sem achado bloqueante; carga e resiliência da §4 ok |
| F2 | Metas de produção do PRD §4 (precisão de alertas ≥ 70% no piloto, aceitação de justificativa ≥ 80%, redução de glosa medida) |

## 7. F2+ (em linhas gerais)

- F2: golden set `piloto` (200 casos reais), comparação contra baseline, teste de contrato do endpoint de desfecho em lote.
- F3: backtest do score de risco em histórico separado por tempo (treina em meses antigos, testa nos recentes); parser TISS XML contra arquivos reais de cada versão suportada.
- F4: WER do STT em áudios reais consentidos; latência de eventos parciais.
- F5: E2E com Playwright sobre páginas de PEP de terceiros salvas como fixture; monitoramento sintético em produção para detectar quebra de seletores.
