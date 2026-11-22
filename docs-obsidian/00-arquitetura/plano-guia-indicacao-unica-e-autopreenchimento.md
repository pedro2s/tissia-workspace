---
projeto: PEP
tipo: plano
status: concluido
tags:
  - tissia
  - pep
  - guia-tiss
  - prontuario
  - faturamento
  - plano
aliases:
  - Plano indicação única e autopreenchimento da guia
---

# Plano — indicação clínica única (Campo 23) e autopreenchimento da guia SP/SADT

Origem: rodada de testes manuais de 19/11/2026 após o merge da demo. Dois achados:

1. **Telas 05a–08 (médico).** Cada pedido de exame pede e repete a indicação clínica. Na guia SP/SADT o **Campo 23 é único por guia**, não por linha de exame.
2. **Tela 17 (faturista, Editor de guia).** Dados que já pertencem ao paciente, ao médico e ao contrato com a operadora (carteirinha, validade, solicitante, conselho, CBO, contratado, CNES) não vêm preenchidos; o faturista sai do fluxo para buscá-los em outra tela.

Notas relacionadas: [[RFC-PEP-004-prontuario]] (RF-DOC-02), [[RFC-PEP-006-guias-e-lotes-tiss]], [[RFC-PEP-008-contratos]], [[hub-pep-tissia]].

## Diagnóstico (código em 19/11/2026)

| Achado | Causa |
| :--- | :--- |
| Indicação repetida | `pedidos_exame.indicacao_clinica` é `NOT NULL` por linha; a UI (`Atendimento.tsx`, `Revisao.tsx`, `Documentos.tsx`) pede o texto a cada pedido. |
| Campo 23 vazio na guia | `PedidosService.fechar` cria a revisão 1 com `indicacaoClinica: null`; o texto só existe dentro dos `itens[].indicacaoClinica`. O faturista digita o Campo 23 do zero. |
| Guia nasce sem `dados` | `criarRascunho` só preenche `dados` na consulta (`dataAtendimento`). SP/SADT nasce vazia. |
| Faltam cadastros de origem | `usuarios` tem conselho/número/UF **sem tela nem API de edição** e **sem CBO**; `contratos` **não tem o código do contratado na operadora**; o CNES está em `clinicas.registro_cnes`. |

## Decisões

- **D1.** A indicação clínica passa a ser **do atendimento** (`atendimentos.indicacao_pedidos`, ≤ 500). `pedidos_exame.indicacao_clinica` continua existindo (compatibilidade, migração só aditiva) e guarda uma **cópia** da indicação do atendimento. Atendimentos antigos: leitura com *fallback* para a indicação do primeiro pedido (não há *backfill* em SQL porque o RLS forçado impede o `UPDATE` na migração).
- **D2.** O **Campo 23 da revisão 1** da guia nasce da indicação do atendimento. O faturista continua podendo editar (nova revisão com autor).
- **D3.** O autopreenchimento é **no servidor**: um resolvedor único (`SugestaoDadosGuia`) lê beneficiário, médico, unidade e **contrato vigente da operadora na data do atendimento**, devolve `dados` + `origens` + `pendencias`. A guia já nasce preenchida e o editor consulta o mesmo resolvedor ao trocar a operadora. Nunca sobrescreve o que o faturista digitou.
- **D4.** Nenhum código ANS é inventado: valores de domínio (caráter, tipo de atendimento, regime) só são pré-preenchidos se confirmados nas bases de `apps/api/ref/`; senão viram `pendencias`.
- **D5.** Faltou cadastro na origem → a tela diz **onde cadastrar** (contrato, perfil do profissional, clínica), em vez de deixar o campo vazio sem explicação.

## Fundação (feita pelo orquestrador, branch `feat/guia-fundacao`)

Contratos aditivos em `@pep/contracts`: `Atendimento.indicacaoPedidos`, `CriarPedidoExame.indicacaoClinica` opcional, `DefinirIndicacaoPedidosSchema`, `codigoContratado` em contratos, `SugestaoDadosGuiaSchema`/`OrigemDadoGuia`. Migração `0047`: `atendimentos.indicacao_pedidos`, `contratos.codigo_contratado`, `usuarios.cbo`. Fatias abaixo partem dela.

## Fatias (em paralelo)

| Fatia | Escopo | Branch |
| :--- | :--- | :--- |
| **G1** | API da indicação única: `PUT /atendimentos/:id/indicacao-pedidos`, pedido herda a indicação, editar pedido atualiza a do atendimento, Campo 23 da revisão 1, XML e Spike, seed clínico | `feat/guia-g1` |
| **G2** | UI do médico (05a–08): um campo de indicação por atendimento; pedidos só com exame e quantidade | `feat/guia-g2` |
| **G3** | API do resolvedor de dados da guia: `GET /api/v1/guias/:id/sugestao-dados`, preenchimento no fechamento do atendimento | `feat/guia-g3` |
| **G4** | UI do Editor de guia (tela 17): usa a sugestão, não sobrescreve, mostra a origem, pendências com atalho para o cadastro | `feat/guia-g4` |
| **G5** | Cadastros de origem: `codigoContratado` no contrato (API + tela), conselho/número/UF/CBO do profissional (API + tela) | `feat/guia-g5` |
| **G6** | Documentação: ADR, notas nas RFCs, manual de integração (se a Spike for tocada) | `docs/guia-indicacao-e-autopreenchimento` |
| **S7** (depois do merge de G1–G5) | Seed da demo + e2e: contrato com código, CBO e CNES nos dados da demo; specs atualizados; roteiro da Carla | — |

Dependências: G2 depende do contrato de G1; G4 do de G3; G3 e G4 dependem de G5 só para os dados existirem (o contrato já está na fundação). Conflitos esperados: `prontuario.service` (G1 × G3, no ponto de `criarRascunho`) e `faturamento.service`; o orquestrador resolve na integração e regenera o snapshot OpenAPI **uma vez**.

## Critérios de aceite

- Médico informa a indicação **uma vez** por atendimento; adicionar o 2º, 3º exame não pede texto.
- Guia SP/SADT fechada nasce com Campo 23 preenchido e `dados` resolvidos (beneficiário, solicitante, conselho, UF, CBO, contratado, CNES) quando os cadastros existem.
- Trocar a operadora no Editor atualiza contratado/código sem apagar edição manual.
- Falta de cadastro aparece como pendência com o caminho para resolver.
- `pnpm typecheck`, `pnpm deps`, testes de contracts/API/UI e `pnpm e2e:demo` verdes; invariantes do Painel do Gestor inalterados.
