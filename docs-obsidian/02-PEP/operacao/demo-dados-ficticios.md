---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - demo
  - seed
  - dados-ficticios
aliases:
  - Demo de dados fictícios
  - Seed de demonstração
  - seed:demo
origem: tissia/docs/demo-dados-ficticios.md
---

# Demo de dados fictícios (seed `demo:reset`)

Como o PEP é populado para a apresentação a um CMO: **dados 100% sintéticos em todos os fluxos**, por módulos de área com um contrato comum. Entregue no PR #30 do `tissia` (`feat/demo-integrada`, **ainda em rascunho** quando esta nota foi escrita). A infraestrutura da demonstração (Docker Compose + Caddy) está em [[demo]]; os números do deck em [[demo-numeros-do-deck]]; a decisão em [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026]]. Hub: [[hub-pep-tissia]].

## Módulos e ordem

Código em `apps/api/scripts/demo/*`; a ordem está em `index.ts` e o seed base é `apps/api/scripts/seed-demo.ts`. Cada módulo exporta `semear(ctx)` e só depende do que os anteriores gravaram no banco.

| # | Módulo | O que semeia |
| :-- | :--- | :--- |
| 1 | `clinico` | 93 atendimentos do elenco com evolução SOAP versionada, adendos, receitas e atestados (assinados e rascunho), 4 modelos da clínica, 4 sessões de escuta (pendente de revisão, revisada, pausada) |
| 2 | `agenda` | 24 pacientes de elenco, agenda de hoje e ±4 semanas, elegibilidade variada, 14 notificações; os "atendido" do passado são derivados dos atendimentos |
| 3 | `faturamento` | mês corrente: 47 guias em todos os estados, 8 lotes com XML validado no XSD TISS 4.03, 4 contratos com preços, pacotes e auditoria |
| 4 | `receita` | 5 meses de história em 120 pacientes sintéticos próprios: 340 guias, 38 demonstrativos de retorno, 58 glosas (Tabela 38), 34 recursos, 23 atendimentos particulares |
| 5 | `ia` | análises e decisões persistidas (só elenco e guias sem atendimento), modos, trilha de auditoria selada, acessos de suporte |
| 6 | `bi` | reprocessamento final do BI (`BiService.reprocessarTenant`, o mesmo de `pnpm bi:reprocessar`) |

### Contrato `ContextoDemo` (`contexto.ts`)

`db` (conexão `app_admin`, `BYPASSRLS`), `dono()` (nova conexão do dono do schema, para DDL e triggers de imutabilidade), `tenantId`, `clinicaId`, `unidadeId` (Centro) e `unidadeNorteId`, `profs` (`A` = Dra. Exemplo A, ortopedia; `B` = Dr. Exemplo B, cardiologia), `usuarios` (faturista, gestora, recepção), `pacientes` (elenco fixo de 24), `hoje` (Brasília, `AAAA-MM-DD`), `salas`, `chavePii` e `diasAtras(n, hora?)`.

## Regras de todo módulo

- **100% sintético:** sobrenome "Exemplo", e-mails `.test`, sem CNS nem telefone. Os CPFs são sintéticos com dígitos verificadores errados de propósito (faixas `0000000xx` e `000001xxx`), nunca válidos.
- **Códigos só das bases** ([[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]]): TUSS da Tabela 22 oficial, CID-10 e DUT das bases da `tissia-ai`. Nada inventado.
- **Idempotente:** cada módulo apaga do tenant só o que ele cria. Trilhas append-only ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], auditoria, análises) são limpas pelo `ctx.dono()`, desligando o trigger só dentro da transação e religando.
- **Datas relativas a hoje** (`ctx.hoje`, `ctx.diasAtras()`): a história não envelhece.
- O seed recusa `NODE_ENV=production`.

## Como rodar

| Comando | O que faz |
| :--- | :--- |
| `pnpm demo:reset` (raiz do `tissia`) | `db:setup` (migra), `scripts/demo-refs.sh` (carrega as referências da ANS se existirem) e `seed:demo` |
| `pnpm --filter @pep/api seed:demo` | só o seed (idempotente), com o banco já migrado |
| `DEMO_ANS_DIR=<pasta>` | pasta com os dois `.zip` da ANS (Representação de Conceitos 202609 e Conteúdo e Estrutura 202511). Padrão do script: `$HOME/dev/pedro/tissia-ai-tunning/padrao_tiss_setembro_2026` |
| `infra/demo/` (`demo.sh`, serviço `init` do Compose) | monta `DEMO_ANS_DIR` em `/ans:ro`; o `init` roda `db:setup`, carrega as referências **se** houver `.zip` e roda `seed:demo`. Sem a variável, monta `infra/demo/ans-vazio/` e o seed sai parcial. `gerar-env.sh` preenche `DEMO_ANS_DIR` no `.env` |

`demo-refs.sh` é idempotente (arquivo já carregado diz "nada a fazer"), restaura `ref/MANIFEST.yaml` se a carga o alterou e sai 0 mesmo sem os `.zip`. Ele **não** carrega o CADOP.

## Três cenários de referências

| Cenário | O que carregar | Resultado |
| :--- | :--- | :--- |
| Sem nada | nenhuma referência | Demo **parcial** com aviso final ("A DEMO FICOU PARCIAL"). Semeia clínico, agenda, faturamento (sem validação local), IA e BI; pula o histórico de receita (precisa da Tabela 38). Painel do Gestor vazio. Saída 0 |
| **Só ANS, sem CADOP (recomendado)** | domínios TISS e campos das guias | Demo completa. Operadoras criadas pelo seed com nome fictício "Operadora A–D (demonstração)" |
| Com CADOP | os dois acima e o cadastro de operadoras | Demo completa, mas as linhas de operadora passam a ter o nome real do CADOP (a carga posterior sobrescreve as linhas `demo`). Com `VITE_AMBIENTE_DEMO=true` a UI ainda mostra "Operadora A–D" |

Nos três casos os registros ANS das quatro operadoras são os públicos de operadoras reais (326305, 005711, 006246, 343889); só os nomes são fictícios. Verificado, nos cenários ii e iii: Painel idêntico e `alertasSemMapeamento = 0`.

## O que cada tela mostra

Perfis da demo e roteiro: [[demo]]. Verificação visual de todas as telas por perfil feita com Playwright em `AUTH_MODE=dev`.

| Tela | Conteúdo semeado |
| :--- | :--- |
| `/home`, `/agenda`, `/pacientes`, ficha do paciente | KPIs, agenda de hoje e de 4 semanas, 144 pacientes cadastrados (24 de elenco e 120 históricos), elegibilidade e pendências, histórico clínico e "Receitado nos últimos 90 dias" |
| `/prontuario`, `/atendimento/:id`, documentos, modelos | evoluções versionadas com adendo, receitas e atestados, painel de IA que lê a análise persistida |
| `/atendimento/:id/escuta` e `/revisao` | sessões de escuta sintéticas; texto de exemplo para a gravação ao vivo |
| `/faturamento` (Mesa), guias, lotes, contratos | fila de 26 guias do mês, guia impressa (guia real), lotes com XML, 4 contratos com pacotes e auditoria de divergências |
| `/retornos`, `/glosas` | 38 demonstrativos (paginados), central de glosas com recurso em preparo e enviado |
| `/gestao` | Painel do Gestor real sobre o histórico, mais a seção "Simulação sobre o lote sintético" (só na demo) |
| `/modos` | modo silencioso e assistido, simulação do lote sintético de 200 guias |
| `/config` | trilha de auditoria íntegra, 3 acessos de suporte, cartão de IA |

**Prévia estática** (selo "Dados ilustrativos", sem API por trás): `/financeiro`, `/receita` e `/dispositivos`; em `/config` partes são rotuladas "Ilustrativo".

## Invariantes do Painel do Gestor

- Taxa de glosa **18,5% (linha de base) → 9,2% (assistido)**; glosa evitada **R$ 14.792** (`1479200` centavos, igual a `SUM(valor_evitado)`); as quatro operadoras A–D com linha de base maior que o assistido.
- Simulação sobre o lote sintético: 200 guias, 46 glosadas, 39 com alerta, **R$ 39.956** (conferido na verificação visual; não remedido depois das últimas correções de dados).
- A história é fixa por data (competências `2026-07` a `2026-11`, novembro sem desfecho). Nenhum teste automatizado fixa esses números; foram conferidos por SQL e pela API do Painel após rodar o seed.
- **Rodar `seed:demo` na manhã da apresentação, antes das 08:00:** a agenda tem horários fixos (08:00 a 16:30) e alguns estados dependem do relógio do seed ("em atendimento há N min", o atendido das 08:00 da Ana, "Abertos recentemente", notificações).

## Limites e riscos conhecidos

- **e2e não executado:** `pnpm e2e:demo` (precisa de Keycloak, IA e usa o banco de desenvolvimento) não rodou contra esta versão do seed. A análise dos specs é estática; o roteiro ao vivo da Carla (`roteiro.spec.ts`, agora com `30713153`) só se valida lá. O `testIgnore` por `E2E_SEM_REFERENCIAS` (runner sem os `.zip` da ANS) foi ampliado e o CI do PR ainda precisa ser confirmado.
- **Registros ANS reais:** as quatro operadoras mantêm o registro real em guias, editor, guia impressa e XML. Eles sustentam a regra de autorização prévia da IA para o 005711 (`projetos/tissia-ia/data/regras/operadora_005711.yaml`); trocar por registros fictícios mudaria o comportamento da IA. **Decisão (19/11/2026, dono do produto): os registros reais ficam.** São identificadores públicos do cadastro da ANS, sem restrição de LGPD; só os nomes das operadoras são fictícios na demo.
- **`lotes.service.ts`** (cerca das linhas 100 e 200) ainda leva data de texto livre ao SQL `date` (risco pré-existente de erro 500 ao gerar lote com data malformada). O detalhe da guia, a transição e o reajuste já validam `AAAA-MM-DD` ([[RFC-PEP-006-guias-e-lotes-tiss]]).
- **Guias da Mesa:** a G-004818 (consulta do seed base) continua não liberável porque o `consulta.spec.ts` depende disso, e as guias de alto risco do seed base têm 15 erros de propósito. A G-004801 aponta para o atendimento de hoje da Ana; a G-004818 aponta para o atendimento fechado anterior mais recente (04/11), porque o índice `ux_guia_atendimento` permite uma guia por atendimento.
- **Não verificado:** modo OIDC (nome do usuário e Semana da agenda), análise de IA ao vivo (Spike), perfis recepção, somente leitura e DPO, impressão real em PDF, a stack Docker (`init` no contêiner) e ações de escrita nas telas.
- A tabela `auditoria_eventos` cresce a cada execução do seed (os `paciente.visualizado` da agenda e do financeiro particular não são limpos; é append-only).

## O episódio que motivou as regras

Cada módulo passava isolado; a integração revelou que:

- o módulo `ia` apagava as análises e decisões do **tenant inteiro** e as recriava em modo `assistido`, destruindo a linha de base da `receita` (Painel: `linhaBasePct: null`, glosa evitada 0). Correção: `limparAnalisesIa` e `derivarAlvos` restritos ao elenco e às guias sem atendimento, com semente estável (`numero_prestador`) para as decisões serem idempotentes;
- o histórico de receita nos 24 pacientes do elenco poluía as fichas clínicas. Correção: elenco histórico próprio de 120 pacientes;
- guias do seed base e o catálogo TUSS da demo tinham códigos que não correspondiam ao termo oficial da Tabela 22 (11 termos do `CATALOGO_TUSS_DEMO` errados; por exemplo `20102020` descrito como MAPA, que é `20102038`). Correção: catálogo conferido contra a Tabela 22 e guias, pedidos, lote sintético e roteiro remapeados.

Registrado como [[ADR-PEP-0026-seed-de-demonstracao-modular|ADR-PEP-0026]]. As notas de implementação das RFCs afetadas: [[RFC-PEP-004-prontuario]], [[RFC-PEP-006-guias-e-lotes-tiss]], [[RFC-PEP-007-retorno-e-glosas]], [[RFC-PEP-008-contratos]], [[RFC-PEP-012-painel-do-gestor]], [[RFC-PEP-013-voz-ambiental]] e [[RFC-PEP-001-identidade-e-acesso]]; referências da ANS em [[RFC-PEP-021-dados-de-referencia-ans]].
