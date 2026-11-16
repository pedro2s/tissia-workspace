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
  - 0002-python-fastapi
origem: tissia-ai/docs/adr/0002-python-fastapi.md
---

# ADR-0002 — Python + FastAPI

**Status:** Aceita — 29/10/2026

## Contexto

O microsserviço de IA vai crescer para análise preditiva sobre histórico de guias (F3), avaliação de modelos e processamento de áudio (F4). O ecossistema Python é o mais maduro para isso (pandas, scikit-learn, SDKs de provedores de IA, ferramentas de STT).

## Decisão

- Python 3.12, FastAPI, Pydantic v2 (schemas da API = fonte do OpenAPI).
- `uv` para dependências e lockfile; `ruff` para lint/format; `pytest` para testes.
- SQLAlchemy 2 (Core/ORM leve) + Alembic para o Postgres.
- `httpx` para chamadas HTTP de saída (Jev, OpenRouter); SDK `google-genai` para Gemini/Vertex.
- Um único pacote `app/`, com estrutura plana. Sem monorepo nem separação em libs até existir um segundo deployable.

## Consequências

- Front-end (extensão F5, eventual painel) fica em TypeScript num diretório separado quando existir. Contrato entre eles = OpenAPI gerado pelo FastAPI.
- Sem worker/fila na F1. Tudo é síncrono na requisição. Fila (Cloud Tasks) entra na F3 para a ingestão de lotes.
