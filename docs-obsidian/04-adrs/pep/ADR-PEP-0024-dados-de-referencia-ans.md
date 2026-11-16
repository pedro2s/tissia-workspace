---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0024-dados-de-referencia-ans.md
---

# ADR-PEP-0024 — Dados de referência da ANS: fonte oficial, versão com proveniência e alerta de defasagem

**Status:** Aceita — 13/11/2026, com duas decisões diferentes da proposta (dono da manutenção e portão informativo, abaixo)
**Relacionadas:** [[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]] (motor TISS e projeto `tiss` só como referência), [[SPK-09-catalogo-tuss|SPK-09]] (catálogo TUSS/CID na Spike), [[SPK-10-tiss-4-03|SPK-10]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-007-retorno-e-glosas|RFC-PEP-007]], [[RFC-PEP-008-contratos|RFC-PEP-008]]

## Contexto

O faturamento depende de dados publicados pela ANS que mudam com frequência: o Padrão TISS (Comunicação 04.03.00, Conteúdo e Estrutura 202511, Organizacional 202609 e Representação de Conceitos 202609, competência set/2026), a Tabela 38 (mensagens e motivos de glosa), as tabelas de domínio da guia, o cadastro de operadoras ativas (CADOP) e a TUSS. Hoje o PEP ainda citava a 4.01, já superada, e a Spike lê XML só até 4.01.00. Descobrir uma tabela defasada só em produção significa lote recusado pela operadora ou glosa que a IA não previu.

## Decisão

1. **Dono de cada base.**

   | Base | Dono | Fonte oficial |
   | :--- | :--- | :--- |
   | TUSS (procedimentos), CID-10, DUT e Rol | Spike ([[SPK-09-catalogo-tuss\|SPK-09]]); o PEP consulta e não copia | ANS / DATASUS |
   | XSD do Componente de Comunicação (04.03.00) | PEP | Pacote "Componente de Comunicação" da ANS |
   | Campos, obrigatoriedade e domínios das guias (consulta, SP/SADT) | PEP | Componente de Conteúdo e Estrutura (202511) e Representação de Conceitos (202609) |
   | Tabela 38 | PEP (conciliação e glosas) e Spike (análise e recurso), cada um com a sua carga da mesma publicação | Representação de Conceitos em Saúde (202609) |
   | Operadoras ativas (CADOP) | PEP | Dados abertos da ANS, "operadoras de planos de saúde ativas" (CC BY) |

2. **Só a publicação oficial entra.** Toda carga lê o arquivo original da ANS (gov.br ou `dadosabertos.ans.gov.br`). O projeto `tiss` e qualquer outra compilação servem só para conferir o resultado, nunca como fonte ([[ADR-PEP-0012-motor-tiss|ADR-PEP-0012]]). Nada é digitado à mão; testes e seeds usam códigos reais dessas cargas ou o placeholder `<TUSS_…>`.
3. **Versão com proveniência.** Cada base tem um manifesto versionado no repo (`apps/api/ref/MANIFEST.yaml`): fonte, URL, competência/versão da ANS, data de publicação, SHA-256 do arquivo original e licença. XSDs ficam em `apps/api/ref/tiss/<versao>/`; o resto vai para tabelas globais `ref_*` (sem `tenant_id`, só leitura para `app_rw`, como TUSS/CID na Spike), com a coluna `versao_ref`. Carga idempotente por script (`pnpm --filter @pep/api ref:carregar <base> <arquivo>`), que recusa arquivo cujo SHA-256 não bate com o manifesto.
4. **Dono da manutenção: o Backoffice (equipe da plataforma).** Decidido em 13/11/2026. A tela "Bases" do backoffice (RFC-PEP-020) é onde a defasagem aparece e onde o acompanhamento da carga acontece; a carga em si continua revisada em PR com manifesto. Publicação das fontes esperada **mensalmente** (CADOP e Representação de Conceitos).
4a. **Alerta de defasagem, não atualização automática.** Um job semanal do worker compara a competência e o hash publicados pela ANS com o manifesto e, havendo novidade, registra alerta operacional (log estruturado + métrica `ref_defasada{base}`), exibido também no backoffice (tela de bases). Atualizar continua sendo uma mudança revisada (novo manifesto + carga + testes), porque versão nova do XSD ou da Tabela 38 pode mudar regra.
5. **Verificação no release é informativa, não bloqueia.** Decidido em 13/11/2026: a defasagem **não** reprova release. A checklist de release inclui o relatório `ref:verificar` (manifesto × publicação da ANS × competência da Spike), que alerta e é lido por quem libera, mas quem decide é o responsável pelo release. Sem arquivo de exceções. Revisitar se uma base defasada chegar à produção.
6. **Transparência na tela.** O que hoje é texto fixo ("Bases de regras atualizadas: TISS 4.01 e Rol ANS") passa a vir dos manifestos do PEP e da `versao_bases` da Spike.

Implementação detalhada na [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021]].

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Usar os dados empacotados do `tiss` | É compilação de terceiro: atraso e erro de transcrição não são nossos de controlar; vira dependência do produto ([[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]]) |
| Atualização automática a cada publicação da ANS | Mudança de XSD ou da Tabela 38 pode quebrar geração e conciliação sem revisão humana |
| Todas as bases só na Spike | XSD, obrigatoriedade e Tabela 38 são necessários sem rede e sem IA (a Spike nunca bloqueia o faturamento) |

## Consequências

- Tarefa recorrente de manutenção: acompanhar as publicações da ANS e rodar a carga, com dono definido no time.
- A Spike precisa expor a competência de cada base em `versao_bases` (pedido no prompt de refinamento da Spike) para o relatório de release comparar.
- Licenças registradas no manifesto: CADOP e dados abertos em CC BY (atribuição à ANS); TUSS em CC BY-ND (redistribuição sem alteração); Tabela 38 sem licença explícita na publicação **[validar com o jurídico]**.

**Aceite (13/11/2026):** dono = Backoffice; publicação mensal; verificação informativa (não bloqueia release). A licença dos XSD e da Tabela 38 segue com o jurídico e bloqueia só a fatia 1 da RFC-PEP-021.
