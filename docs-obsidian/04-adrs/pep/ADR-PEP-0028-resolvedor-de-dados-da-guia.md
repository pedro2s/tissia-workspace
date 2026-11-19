---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
  - faturamento
  - guia-tiss
  - contratos
aliases:
  - ADR-PEP-0028
  - Resolvedor de dados da guia no servidor
---

# ADR-PEP-0028 — Resolvedor de dados da guia no servidor e cadastros de origem

**Status:** Proposta — 19/11/2026 (base em `feat/guia-fundacao`, fatias G3/G4/G5 em implementação; vira `aceita` somente no merge do código — ADRs aceitas são imutáveis; ver [[plano-guia-indicacao-unica-e-autopreenchimento|plano]])
**Relacionadas:** [[ADR-PEP-0002-backend-nestjs-monolito-modular|ADR-PEP-0002]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]], [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]], [[RFC-PEP-008-contratos|RFC-PEP-008]]

## Contexto

Na tela 17 (`EditorGuia.tsx`), os campos TISS da guia SP/SADT (carteirinha, validade, solicitante, conselho, número, UF, CBO, contratado e CNES) nasciam vazios: `FaturamentoService.criarRascunho` só preenchia `dados.dataAtendimento` para guias de consulta. O faturista precisava sair do editor para procurar informações que já pertencem ao paciente, ao médico, à clínica e ao contrato com a operadora.

Além disso, faltavam dois campos nos cadastros de origem:
- `usuarios` já possuía `conselho`, `numero_conselho` e `uf_conselho`, porém **sem `cbo`** e **sem API ou tela de edição**;
- `contratos` **não possuía o código do contratado na operadora** (campos 13/29 da SP/SADT e campo 9 da consulta).

## Decisão

1. **D3 — Resolvedor único no servidor e preenchimento não destrutivo.**
   - A resolução dos dados da guia é centralizada **no servidor**, retornando `SugestaoDadosGuia` (`registroAns`, `contratoId`, `dados`, `origens` com `OrigemDadoGuia`: `beneficiario | profissional | contrato | unidade | padrao`, e `pendencias`: `{ chave, motivo }[]`, schemas já definidos em `packages/contracts/src/validacao-guia.ts` na fundação `feat/guia-fundacao`).
   - *(Intenção do plano, fatia G3 — a confirmar na integração)*: um endpoint dedicado `GET /api/v1/guias/:id/sugestao-dados` expõe o resolvedor (aceitando troca de `registroAns` por query para consulta no editor). No fechamento do atendimento (`ProntuarioService.fechar` → `FaturamentoService.criarRascunho`), o servidor já cria a revisão 1 com `dados` resolvidos.
   - *(Intenção do plano, fatia G4 — a confirmar na integração)*: o Editor de guia (tela 17) consulta o mesmo resolvedor ao abrir ou ao trocar a operadora e **nunca sobrescreve** valores já digitados pelo faturista.

2. **Módulos donos de cada dado (fronteiras estritas da [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]] e [[ADR-PEP-0018-monorepo-pnpm-turborepo|ADR-PEP-0018]]).**
   O resolvedor consulta cada domínio exclusivamente pelos serviços públicos dos respectivos módulos (proibido importar `schema.ts` de outro módulo):

   | Dado da guia | Módulo dono | Origem cadastral / observação |
   | :--- | :--- | :--- |
   | Carteirinha, validade, `registroAns`, atendimento RN | `pacientes` | `beneficiarios` (número da carteirinha é cifrado em repouso e decifrado apenas pelo módulo dono) |
   | Solicitante/executante, conselho, número, UF, CBO | `identidade` | `usuarios` (`conselho`, `numero_conselho`, `uf_conselho` + `cbo` adicionada na migração `0047`) |
   | CNES e nome do contratado | `organizacao` | `clinicas.registro_cnes` / `clinicas.nome` |
   | Código do contratado na operadora | `contratos` | `contratos.codigo_contratado` (migração `0047`), selecionado pelo **contrato vigente na data do atendimento/solicitação** para a unidade × `registroAns` |

3. **D4 — Nenhum código ANS inventado ([[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]]).**
   - Campos de domínio fechado (caráter de atendimento / tabela 23, tipo de atendimento / tabela 50, indicação de acidente / tabela 36, regime de atendimento / tabela 76, tipo de consulta / tabela 52) só podem ser pré-preenchidos se confirmados nas tabelas de referência carregadas (`ref_dominios_tiss` / `apps/api/ref/`).
   - Quando a base não estiver carregada ou não houver confirmação segura de domínio, o campo permanece vazio e é listado em `pendencias`.

4. **D5 — Pendências acionáveis e cadastros de origem.**
   - Quando falta dado na origem, o resolvedor inclui uma entrada em `pendencias` explicando **o que falta e onde cadastrar** (contrato, perfil do profissional ou clínica/unidade), em vez de deixar o campo vazio sem orientação. A UI da tela 17 (fatia G4, *a confirmar na integração*) exibe a origem de cada dado preenchido e atalhos para o cadastro correspondente.
   - Os cadastros de origem faltantes foram adicionados na migração `0047_guia_indicacao_unica_e_cadastros_de_origem.sql` (`contratos.codigo_contratado` e `usuarios.cbo`, já em `feat/guia-fundacao`) e têm API e telas previstas na fatia G5 (*a confirmar na integração*: `codigoContratado` em `Contratos` e `/api/v1/me/dados-profissionais` + tela do profissional).

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Montar o autopreenchimento no frontend (`@pep/ui`) com múltiplas requisições | Duplicaria a regra de seleção do contrato vigente por data, exigiria expor dados cadastrais/cifrados diretamente à tela do faturista e não preencheria a revisão 1 da guia no fechamento do atendimento |
| Gravar valores *default* fixos no código para domínios ANS quando a base não estiver carregada | Viola a [[ADR-PEP-0024-dados-de-referencia-ans\|ADR-PEP-0024]] e arrisca emitir XML rejeitado pelo XSD ou pela operadora |

## Consequências

- **Menos retrabalho e menos erro no faturamento:** a guia SP/SADT já nasce preenchida quando os cadastros existem, e trocar a operadora no Editor atualiza os campos dependentes do contrato sem apagar edições manuais.
- **Transparência e rastreabilidade:** `origens` mostra de onde veio cada valor e `pendencias` orienta o usuário a corrigir a origem do dado (contrato, perfil profissional ou unidade).
- **Dependência de cadastros reais (fatia G5 e S7):** enquanto os cadastros de origem não forem preenchidos nas unidades/contratos/profissionais (e no seed da demo), o resolvedor sinalizará pendências acionáveis em vez de inventar valores.
- **Efeito no contrato com a Spike (`tissia-ia` API `v1`):** **nenhum** — campos administrativos/cadastrais (`guia_revisoes.dados`) pertencem ao XML TISS do PEP e não integram o payload de `POST /v1/guias/analisar`.
