---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - demo
  - seed
aliases:
  - ADR-PEP-0026
  - Seed de demonstração modular
origem: tissia/docs/adr/ADR-PEP-0026-seed-de-demonstracao-modular.md
---

# ADR-PEP-0026 — Seed de demonstração modular, com contrato de contexto, pacientes de história próprios e referências opcionais

**Status:** Proposta — 19/11/2026 (implementada no PR #30 do `tissia`, ainda em rascunho; passa a **Aceita** quando o PR for mesclado, e só então se torna imutável; ver [[demo-dados-ficticios]])
**Relacionadas:** [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]] (seed), [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]] (referências da ANS e códigos só de bases oficiais), [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[RFC-PEP-012-painel-do-gestor|RFC-PEP-012]], [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021]]

## Contexto

A demonstração a um CMO exige dados fictícios coerentes em todos os fluxos (prontuário, agenda, guias, retorno e glosas, IA, Painel do Gestor). O seed único (`apps/api/scripts/seed-demo.ts`) não comportava isso. A integração das frentes de dados revelou quatro problemas concretos:

- **Acoplamento silencioso.** O módulo `ia` apagava do tenant inteiro as análises e decisões, inclusive as 340 análises históricas da `receita`, e as recriava em modo `assistido`. Ao fim do seed completo o Painel do Gestor perdia a linha de base (`linhaBasePct: null`, glosa evitada 0). Cada módulo passava bem isolado; o erro só apareceu na junção.
- **Poluição do elenco clínico.** O histórico de 5 meses de receita, ligado aos 24 pacientes do elenco, deixava as fichas deles poluídas (medido com a fatia isolada: 365 atendimentos e 348 guias no elenco, contra 2 e 8 depois da separação).
- **Dependência das bases da ANS.** O seed lançava erro se o CADOP ou as bases de campos e domínios da guia não estivessem carregados (quebraria o `globalSetup` do e2e no CI, que não tem os pacotes da ANS). Com o CADOP carregado, a demo ainda expunha razão social e registro de operadoras reais ao CMO.
- **Códigos errados.** Guias do seed base e o `CATALOGO_TUSS_DEMO` tinham códigos TUSS cujo termo oficial não correspondia à indicação clínica (por exemplo, `41101227` é RM de coluna, não de joelho).

## Decisão

1. **Seed modular com contrato de contexto e ordem fixa.** Cada área é um módulo `semear(ctx)` em `apps/api/scripts/demo/*`, que recebe o `ContextoDemo` (`contexto.ts`: conexão administrativa, `dono()` do schema, tenant, unidade, profissionais, elenco de 24 pacientes, `hoje`, `diasAtras()`). Ordem em `index.ts`: **clínico → agenda → faturamento → receita → ia → BI**. Um módulo só depende do que os anteriores gravaram no banco, nunca de arquivos de outro módulo, e só apaga o que ele mesmo criou (o `ia` ficou restrito ao elenco e às guias sem atendimento). Trilhas append-only são limpas pelo dono do schema, desligando o trigger só dentro da transação. O seed é idempotente e termina reprocessando o BI.
2. **Histórico de receita em pacientes sintéticos próprios.** A `receita` cria 120 pacientes históricos (112 de convênio, 8 particulares) com beneficiário, elegibilidade e consentimento, e liga a eles as 340 guias, os atendimentos e as análises. Os 24 pacientes do elenco só recebem o que a `clinico` e a `faturamento` geram. Não se usam guias sem atendimento para o histórico porque recurso de glosa, recebível particular e produção diária do BI leem `atendimentos` e `evolucao_versoes`.
3. **Operadoras fictícias quando o CADOP não está carregado.** O módulo `faturamento` insere as 4 operadoras ausentes como "Operadora A–D (demonstração)" (`versao_ref = 'demo'`, `ON CONFLICT DO NOTHING`); linhas reais do CADOP nunca são tocadas. O caminho recomendado é carregar as bases de campos e domínios da ANS e **não** carregar o CADOP. Os registros ANS (326305, 005711, 006246, 343889) continuam reais, porque as regras da IA os usam; só os nomes são fictícios. Com `VITE_AMBIENTE_DEMO=true`, a UI mostra o rótulo A–D e limita a busca de operadoras a essas quatro.
4. **Seed que degrada com aviso, sem as referências da ANS.** Sem `ref_campos_guia` ou `ref_dominios_tiss`, a validação local das guias é pulada; sem a Tabela 38, o histórico de receita é pulado. Cada módulo registra o que pulou (`avisos.ts`) e o seed imprime um único aviso final, "A DEMO FICOU PARCIAL", com o comando que carrega as referências. A saída é 0. A carga é opcional e idempotente (`demo-refs.sh`, `DEMO_ANS_DIR`).
5. **Catálogo TUSS da demo = Tabela 22 oficial.** Os 19 códigos de `CATALOGO_TUSS_DEMO` (`packages/contracts`) têm o termo da Tabela 22 vigente (`tissia-ai/data/tuss/22_procedimentos.csv`), conferidos um a um. Guias, pedidos, lote sintético, receita e roteiro ao vivo foram remapeados para códigos cujo termo e DUT sustentam a indicação clínica (consistente com [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]]: nada digitado à mão fora das bases oficiais).

## Consequências

- O seed completo roda 2 vezes seguidas com contagens idênticas e a narrativa do Painel do Gestor estável (glosa 18,5% → 9,2%, R$ 14.792 evitados), nos três cenários de referências.
- Um módulo novo entra implementando `semear(ctx)` e declarando seu lugar na ordem; a regra de não apagar o que é de outro módulo é de revisão, não de tipo.
- O seed deixou de depender do CADOP, então o CI da demo roda sem pacotes da ANS (os specs que dependem da validação local ou do histórico de receita ficam em `E2E_SEM_REFERENCIAS`).
- **Custo:** os 4 registros ANS reais permanecem em guias, editor e XML da demo; decidir se ficam é pendência do PR #30. O código TUSS corrigido muda o termo exibido para o mesmo código; registros já gravados com o termo antigo (por exemplo `pedidosExame.termo`) não são reescritos.
- A agenda tem horários fixos (08:00–16:30) e alguns estados dependem da hora em que o seed roda; ele deve rodar na manhã da apresentação.
- O `e2e:demo` não foi executado contra esta versão do seed.
