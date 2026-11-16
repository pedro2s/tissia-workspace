---
projeto: ecossistema
tipo: spk
status: proposta
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-11-contratos-auditar-substituicao.md
---

# SPK-11 — Semântica de substituição em `POST /v1/contratos/auditar`

**Status:** Proposta · **Destino:** repo `tissia-ai` (RFC-011+) · **Necessária para:** auditoria de contratos com provisionamento seguro de regras de autorização ([[RFC-PEP-008-contratos|RFC-PEP-008]], Q3/Q4)

## Problema

Hoje `POST /v1/contratos/auditar` na Spike tem duas limitações que impedem o PEP de ligar `provisionar_regras_autorizacao = true`:

1. **Acúmulo de linhas por chamada:** cada execução de `contratos/auditar` grava novas linhas em `contratos_operadora` para o mesmo `(clinica_ref, registro_ans)` sem substituir ou encerrar as anteriores. Quando uma tabela negociada é reajustada ou fatiada (ex.: tabelas com mais de 2.000 itens enviadas em múltiplos lotes), o histórico na Spike acumula registros duplicados ou obsoletos.
2. **Provisão apenas aditiva de regras:** `provisionar_regras_autorizacao = true` apenas **acrescenta** códigos TUSS à regra `CONTRATO-AUT-<registro_ans>` e nunca remove um TUSS que deixou de exigir autorização numa nova versão do contrato. Além disso, regras condicionais do PEP (`eletivo`, `acima_de_quantidade`) não são representáveis por um booleano simples.

Por esse motivo, o PEP envia `provisionar_regras_autorizacao = false` e mantém a avaliação das regras de autorização localmente.

## Proposta (aditiva em `/v1`)

1. **Identificador de snapshot/versão no payload de `POST /v1/contratos/auditar`:**
   - Aceitar campos opcionais `substituir_tabela: boolean` (default `false` para retrocompatibilidade) e `versao_tabela?: string`, além de `lote_indice?: number` e `total_lotes?: number` (para tabelas fatiadas em lotes de até 2.000 itens).
   - Quando `substituir_tabela = true` (ou na recepção do lote `lote_indice === 1` de um mesmo `versao_tabela`), a Spike substitui atomicamente a tabela vigente daquela `(clinica_ref, registro_ans)` pelo novo conjunto de itens enviados, preservando versões anteriores apenas para histórico auditável (marcadas como substituídas/inativas).
2. **Sincronização completa de regras provisionadas:**
   - Quando `provisionar_regras_autorizacao = true` e `substituir_tabela = true`, a regra `CONTRATO-AUT-<registro_ans>` passa a refletir exatamente os códigos TUSS com `exige_autorizacao_previa = true` da versão enviada, **removendo** os TUSS que deixaram de exigir autorização.
   - Evolução opcional: aceitar `condicao_autorizacao` por item (`sempre | eletivo | acima_de_quantidade | nao_exige`) com parâmetros opcionais (`quantidade_maxima`), para alinhar a análise da Spike às regras parametrizadas no PEP.
3. **Idempotência por `versao_tabela` + `lote_indice`:**
   - Reenvios do mesmo `(clinica_ref, registro_ans, versao_tabela, lote_indice)` por retry de fila devem ser idempotentes sem duplicar linhas.

## Do lado do PEP (até a Spike implementar)

- O PEP continua enviando `provisionar_regras_autorizacao = false` em `POST /v1/contratos/auditar`, em fatias de até 2.000 itens ordenadas por `codigo_tuss`.
- O PEP calcula o SHA-256 do conteúdo enviado para evitar chamadas repetidas quando não houve mudança (`tissia_ai_estado = sem_mudanca`).
- A avaliação de autorização prévia na agenda e na guia usa exclusivamente `regras_autorizacao` locais via função pura `exigeAutorizacao` em `@pep/contracts`.

