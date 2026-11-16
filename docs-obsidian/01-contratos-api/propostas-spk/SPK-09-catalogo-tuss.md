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
origem: tissia/docs/spike/SPK-09-catalogo-tuss.md
---

# SPK-09 — Catálogo de terminologia TUSS e CID-10 (listagem e busca)

**Status:** Implementada na `tissia-ia` ([[RFC-IA-011-catalogo-terminologia-e-rag|RFC-IA-011]], `app/api/catalogos.py`) e consumida pelo `PEP` (`catalogo.service.ts`, [[RFC-PEP-005-spike-adapter|RFC-PEP-005]], Onda 4 parte 3); sem pendência bloqueante no PEP (eventual export em lote para réplica local permanece como evolução futura) · **Necessária para:** pedido de exame/procedimento no prontuário ([[RFC-PEP-004-prontuario|RFC-PEP-004]]), guia SP/SADT ([[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]) e sugestão de CID sem lista fixa no PEP

## Problema

O PEP oferece ao médico, no pedido de exame, um catálogo pequeno e fixo (`CATALOGO_TUSS_DEMO` em `packages/contracts/src/atendimento.ts`: a consulta e os procedimentos das DUTs curadas). A Spike já carrega as bases TUSS e DUT (`bases.tuss`, ingeridas por `scripts/ingest_tuss.py`) e a base de CID-10, e devolve o `termo_tuss` de cada código analisado, mas **não expõe nenhuma rota para listar ou buscar** esses códigos. Sem isso o PEP não tem como ter o catálogo "vindo da Spike" (release plan §13.1, onda 4) e a lista fixa fica desatualizada a cada versão das bases (`versao_bases`).

Regra do projeto: nenhum código TUSS/CID inventado no PEP. A fonte única precisa ser a base que a Spike já valida.

## Decisão de arquitetura: fonte única na Spike

- A terminologia **mora na Spike**, não no PEP. É a Spike que valida o código sugerido pelo LLM contra a base (impede código alucinado de chegar à operadora) e roda a DUT; a Spike atende vários integradores, então a terminologia pertence ao motor.
- O PEP **não mantém cópia local** do catálogo por ora: consulta a Spike na busca (com cache em memória, ver abaixo). Uma réplica local sincronizada fica como evolução futura, se a latência ou a dependência virarem problema; nesse caso seria preciso um export em lote com cursor (fora do escopo desta proposta).
- O RAG (pgvector) e a listagem usam a **mesma tabela de terminologia**: sugestão ao time da Spike é uma coluna `embedding` e um índice vetorial sobre a própria tabela de códigos, em vez de um segundo armazenamento. Assim o catálogo exposto ao PEP e o catálogo que alimenta o RAG nunca divergem. A busca do PEP é **léxica e determinística** (não semântica), pois dispara a cada tecla.

## Proposta (aditiva em `/v1`)

1. `GET /v1/catalogos/tuss?q=<texto|código>&limit=20` com `{ itens: [{ codigo_tuss, termo, grupo?, tem_dut: bool }], versao_bases }`. Busca por prefixo de código e por termo normalizado (sem acento, sem caixa), a mesma normalização da codificação.
2. `GET /v1/catalogos/tuss/{codigo}` com o termo, `tem_dut` e os critérios resumidos (só para tooltip; o checklist completo continua na análise).
3. `GET /v1/catalogos/cid?q=<texto|código>&limit=20` com `{ itens: [{ codigo_cid, termo }], versao_bases }`, mesma busca e normalização, e `GET /v1/catalogos/cid/{codigo}`.
4. `limit` simples: padrão 20, máximo 50, sem cursor. O uso é sugestão em campo de digitação, em que o usuário refina a busca digitando mais; ninguém pagina.
5. Mesma autenticação por integrador. Só dados públicos de terminologia: sem PII, sem dado da clínica.
6. `versao_bases` em toda resposta, para o PEP invalidar o cache quando mudar.
7. Desempenho esperado: índice por prefixo no código e índice trigram (ou coluna normalizada) no termo; resposta na casa de dezenas de milissegundos.

## Do lado do PEP (quando existir)

- `TussService` e `CidService` no módulo `tissia-ai` (cliente HTTP com o circuit breaker já existente) e busca incremental com debounce nos campos "Procedimento" do pedido e "CID".
- Cache em memória por `(catálogo, termo buscado)` com TTL curto, invalidado quando `versao_bases` mudar.
- **Redundância:** Spike fora do ar ou chave inválida não pode bloquear o pedido. O PEP cai no último resultado em cache e, sem cache, no `CATALOGO_TUSS_DEMO` (rótulo "lista reduzida"). O pedido valida o código contra o catálogo disponível e nunca bloqueia pela rede.
- O `CATALOGO_TUSS_DEMO` deixa de ser fonte de verdade e vira apenas o último recurso e a massa dos testes.

## Decisões a tomar (time da Spike)

- Confirmar a tabela única com coluna de embedding para RAG e listagem (recomendado) ou, se o desenho do RAG exigir, o mecanismo que garanta que ambos partam da mesma versão de `bases`.
- Estratégia de busca por termo: trigram, `unaccent` + `ILIKE`, ou índice de texto completo; o contrato acima não depende da escolha.
