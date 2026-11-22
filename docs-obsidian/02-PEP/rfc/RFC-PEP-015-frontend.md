---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-015-frontend.md
---

# RFC-PEP-015 — Frontend: arquitetura da UI, design system e migração

**Fase:** P1 (design system e shell começam na P0) · **Status:** Implementada (base de bibliotecas e tokens configurada em `apps/ui`) · **ADRs:** ADR-PEP-0004, 0015, 0018
**Telas:** 02 (shell, barra de comandos, notificações), 26a–d (tablet e mobile); padrões usados em todas
**Requisitos:** RF-UI-01…RF-UI-12

## Objetivo

Levar `@pep/ui` (`apps/ui`) do estado inicial (telas estáticas fiéis ao design) a uma aplicação real: design system com os tokens exatos do design, contratos compartilhados via `@pep/contracts`, dados vindos de `@pep/api`, estados de carregamento/erro/parcial, offline no prontuário e layout tablet/mobile, **sem perder um pixel da fidelidade visual das 26 telas**.

## Estado inicial e base P0 (07/11/2026)

| Item | Situação |
| :--- | :--- |
| Pacote no monorepo | `apps/ui` (`@pep/ui`), consumindo `@pep/contracts` |
| Estilo e blindagem visual | `apps/ui/src/styles.css` organizado em CSS `@layer` com tokens (`--ink #0F1B2D`, `--teal #0B6E78`, navy, Inter Variable com peso `650`, `tabular-nums`), Tailwind CSS e primitivos shadcn/ui (`cva` + `tailwind-merge`) sem quebrar o visual das telas estáticas `.sc04`…`.sc26` |
| Tipografia | `@fontsource-variable/inter` auto-hospedada (elimina dependência externa do Google Fonts, respeita CSP/LGPD, funciona offline e renderiza `font-weight: 650` real) |
| Layout e primitivos | `Layout.tsx` (`Sidebar`, `Topbar`, `Shell`) e `ui.tsx` (`Icon` inline, `Badge`, `Kbd`, `Sample`), integrados aos providers globais (`QueryClientProvider`, `NuqsAdapter`, `Toaster`) |
| Telas à mão | Credenciais (01a), Login MFA/unidade (01b, `LoginFrame`), Home (02, barra de comandos Ctrl+K), Agenda (03) |
| Telas geradas | 04–26 em `apps/ui/src/pages/screens/*.tsx`, convertidas do design: JSX estático e CSS escopado `.scNN` |

As telas geradas continuam como referência visual viva: cada uma é reescrita com componentes reais quando a RFC do domínio dela é implementada.

## Requisitos

| ID | Requisito |
| :--- | :--- |
| RF-UI-01 | Design system único (tokens + componentes) usado por todas as telas migradas; nenhuma cor/tamanho literal fora do tema. |
| RF-UI-02 | Roteamento por caminho (`/agenda`, `/pacientes/:id`) com `react-router` e estado de filtros na URL tipado com `nuqs`, deep link e guarda de autenticação/permissão. |
| RF-UI-03 | Dados via cliente HTTP tipado sobre `@pep/contracts` e cache/sincronização com TanStack Query v5. |
| RF-UI-04 | Toda tela tem estados vazio, carregando (skeleton), erro (RFC 9457 com repetir) e, onde a Spike participa, parcial (tela 14). |
| RF-UI-05 | Barra de comandos (Ctrl/Cmd+K) com ações, comandos clínicos (`/receita`…) e recentes; atalhos `G A`, `G F`, `N`, `?`. |
| RF-UI-06 | Central de notificações com abas (Todas, Clínico, Faturamento, Sistema) e contador na Topbar. |
| RF-UI-07 | Indicador de status da IA na Topbar: `ok` ("IA ativa · TISS x.y") ou `parcial` ("Análise parcial"). |
| RF-UI-08 | Acessibilidade WCAG 2.1 AA: navegação por teclado, foco visível, contraste ≥ 4,5:1, rótulos em todo controle. |
| RF-UI-09 | Textos em pt-BR centralizados; datas, moeda e números no formato brasileiro (`tabular-nums`). |
| RF-UI-10 | PWA instalável; prontuário funciona offline (leitura do atendimento aberto e escrita em fila local). |
| RF-UI-11 | Layout tablet (834×1194, 1194×834) e mobile (390×844) para as telas 5, 6, 13 e 16; alvos de toque ≥ 44 px. |
| RF-UI-12 | Dados sensíveis mascarados por padrão (CPF, CNS, carteirinha); revelar exige ação explícita e é auditado. |

## Design

### Stack completa aprovada

| Tema | Escolha | Motivo técnico e preservação de fidelidade |
| :--- | :--- | :--- |
| Build e contratos | Vite + TypeScript estrito + `@pep/contracts` | Compartilha schemas Zod e tipos sem codegen entre repos ([[ADR-PEP-0018-monorepo-pnpm-turborepo\|ADR-PEP-0018]]) |
| Estilo e primitivos | Tailwind CSS + **shadcn/ui** (Radix UI) + `class-variance-authority` (`cva`) + `tailwind-merge` | `cva` codifica exatamente as variantes do design (`13px` base, raios `8/10/12px`, tons semânticos `r/a/g/n/t/b`); CSS `@layer` impede o Preflight de desconfigurar `.sc04`…`.sc26` |
| Tipografia | `@fontsource-variable/inter` | Habilita `font-weight: 650` nativo do design, zero requisição externa (LGPD/CSP) e precache offline |
| Rotas e URL state | `react-router` + `nuqs` | Deep links e filtros tipados na URL (Agenda, Mesa do Faturista, Glosas, Auditoria) para visões salvas sem `useState` solto |
| Estado de servidor | TanStack Query v5 + cliente HTTP tipado | Cache por recurso (*Query Key Factories*), mutações otimistas reversíveis e pausa/retomada offline |
| Estado local efêmero | `zustand` | Apenas estado global fora da URL: token de sessão em memória, unidade ativa, `CommandPalette`, status `ai` na Topbar e fila offline |
| Streaming SSE | `@microsoft/fetch-event-source` | Permite enviar `Authorization: Bearer <token>` em streams SSE (`/api/v1/guias/{id}/analise/stream`), impossível no `EventSource` nativo sem expor JWT na URL |
| Formulários | `react-hook-form` + `zod` (`@hookform/resolvers`) | Consome diretamente os schemas de `@pep/contracts`; zero re-render global em telas densas (Editor de Guia tela 17) |
| Editor clínico | **Tiptap** (ProseMirror headless) | Headless (100% CSS do design); viabiliza menu `/` no cursor (`/receita`, `/exame`) e marcação por trecho `humano × IA` (`.ai` tracejado teal) + âncoras `E1`/`E2` (telas 05 e 07) |
| Tabelas e listas longas | TanStack Table v8 + TanStack Virtual | Headless (preserva `.tb` do design); garante 60 fps e seleção em lote `J/K/X` para 5.000 guias (tela 16) e tabelas TUSS (tela 21) |
| Drag-and-drop | `@dnd-kit/core` + `@dnd-kit/sortable` | Construtor de modelos (tela 09) acessível por teclado (WCAG 2.1 AA) e compatível com toque no tablet |
| Comandos, gavetas e avisos | `cmdk` + `react-hotkeys-hook` + `vaul` + `sonner` | `cmdk` para Ctrl+K; `react-hotkeys-hook` para sequências `G A`/`G F` e decisões rápidas `A/E/I`; `vaul` para bottom sheet no tablet (26a) e gaveta de lote (16); `sonner` para toasts discretos |
| Auth | `oidc-client-ts` (Authorization Code + PKCE) + **Keycloakify** | Etapas 1 e 2 de login no Keycloak compiladas com os mesmos tokens/componentes do PEP (RFC-PEP-001) |
| Offline | `vite-plugin-pwa` (Workbox) + IndexedDB (`idb`) | Prontuário cifrado localmente e fila idempotente de sincronização |
| Testes e mocks | Vitest + Testing Library + **MSW** + Playwright (`@axe-core/playwright`) | MSW simula estados vazio/carregando/erro RFC 9457/degradado; Playwright garante regressão visual e acessibilidade |

### Blindagem de CSS `@layer` durante a convivência

Para que Tailwind + shadcn convivam com `styles.css` e com os 23 arquivos `.scNN` gerados sem qualquer regressão visual:

1. `styles.css` declara `@layer theme, base, legacy, components, utilities;` e mantém todas as variáveis originais (`--ink`, `--mut`, `--line`, `--line2`, `--bg`, `--card`, `--teal`, `--teal-d`, `--teal-l`, `--navy` + tokens semânticos mapeados para o Tailwind `@theme`).
2. As classes originais (`.card`, `.btn`, `.bdg`, `.seg`, `.chip`, `.tab`, `.nav`, `.topbar`, `.aipill`, etc.) ficam em `@layer legacy`, preservando `font-size: 13px`, `line-height: 1.45` e controles nativos (`appearance: auto` em inputs dentro de `.sc*` e `.login`).
3. Utilitários Tailwind ficam em `@layer utilities`, vencendo apenas onde explícitos nos componentes novos em `src/ds/` e `src/modulos/`.

### Design system (`src/ds/`)

| Padrão do design | Componente | Base shadcn / Radix | Variantes (`cva`) |
| :--- | :--- | :--- | :--- |
| `.bdg` (selo) | `Badge` | — (`cva`) | tom `r` (risco/erro), `a` (aviso), `g` (ok), `n` (neutro), `t` (teal/ativo), `b` (informativo); tamanho `default` (22 px) / `sm` (20 px) |
| `.card` | `Card` | Card | `padding` none / sm / md |
| `.btn` | `Button` | `@radix-ui/react-slot` + `cva` | `p` (primário teal), `default` (contorno), `q` (texto), `s` (30 px), `default` (36 px), `lg` (44 px toque) |
| `.kbd` | `Kbd` | — (próprio) | — |
| `.tab` + `role=tablist` | `Tabs` | `@radix-ui/react-tabs` | sublinhado teal |
| `.seg` | `Segmented` | `@radix-ui/react-toggle-group` | `single`, ativo navy `#0F1B2D` |
| `.chip` (filtro) | `FilterChip` | `@radix-ui/react-popover` + `cmdk` | com contagem |
| Tabelas (`.tb`) | `DataTable` | TanStack Table + Virtual | ordenação, seleção em lote, scroll horizontal |
| Barra de comandos | `CommandPalette` | `@radix-ui/react-dialog` + `cmdk` | — |
| Gaveta / Bottom sheet | `Drawer` | `vaul` | lateral no desktop (tela 16), bottom sheet no tablet (26a) |
| Selo de risco | `RiskSeal` | — | baixo/médio/alto + score 0,0–1,0 (`tabular-nums`) |
| Aviso legal da IA | `AiDisclaimer` | — | fixo no rodapé dos painéis de IA |
| Trecho de IA | `AiSpan` / Mark Tiptap | — | fundo teal claro, sublinhado tracejado (`.ai` do design) |
| `Icon` | `Icon` | — | mantém os paths SVG inline atuais (zero dependência de pacote de ícones genéricos) |

Regra: componente novo só entra em `src/ds/` quando aparece em duas telas; antes disso fica no módulo.

### Estrutura de `apps/ui`

```
apps/ui/src/
├── app/                # providers (Query, Nuqs, Toaster), store Zustand, layout (Shell, Sidebar, Topbar)
├── ds/                 # design system (utils cn/cva, Button, Badge, Card, Dialog, Tabs, CommandPalette…)
├── api/                # cliente HTTP tipado (@pep/contracts) + SSE autenticado + Query Keys
├── modulos/<dominio>/  # rotas, telas e componentes migrados por RFC (agenda, prontuario, faturamento…)
├── pages/              # telas iniciais e pages/screens/*.tsx (removidas conforme cada RFC migra a sua)
├── offline/            # service worker, fila IndexedDB (idb), sincronização
└── i18n/               # textos e formatadores pt-BR (Intl America/Sao_Paulo, BRL)
```

### Roteamento

| Rota | Tela | RFC |
| :--- | :--- | :--- |
| `/entrar/*` | 01a/01b (Keycloak + seleção de unidade) | 001 |
| `/` | 02 Início | 015 |
| `/agenda` | 03 | 003 |
| `/pacientes/:id` | 04 | 002 |
| `/atendimentos/:id` (`/voz`, `/revisao`, `/documentos`) | 05–08 | 004, 013 |
| `/modelos`, `/modelos/:id` | 09–11 | 004 |
| `/guias/:id` (`/analise`) | 12–14 | 005, 006 |
| `/faturamento`, `/faturamento/guias/:id`, `/faturamento/lotes` | 16–18 | 006 |
| `/retornos`, `/glosas` | 19–20 | 007 |
| `/contratos` | 21 | 008 |
| `/faturamento/receita-esquecida` | 22 | 009 |
| `/financeiro` | 23 | 010 |
| `/gestao` | 24 | 012 |
| `/configuracoes/*` (incl. `modos-ia`) | 15, 25 | 005, 014 |

Durante a transição da P0 para a P1, o roteador suporta tanto os caminhos canônicos quanto os hashes legados dos mockups para preservar todos os links internos das telas estáticas até sua migração; guarda de rota checa permissão do token (RFC-PEP-001) e esconde itens da Sidebar sem permissão.

### Dados e estados

- Um hook por recurso (`useAgendaDia`, `useGuia`…) sobre o cliente tipado com `@pep/contracts`. Mutações com atualização otimista só onde a ação é reversível (ex.: confirmar presença); guias e prontuário esperam o servidor.
- `ETag`/412 da API vira diálogo "este item mudou; recarregar".
- Estados padronizados por componente `Estado` (vazio, carregando, erro). Estado **parcial** (tela 14): quando a resposta da Spike tem `degradado: true` ou a chamada estourou 6 s, a UI mostra o aviso discreto "Análise parcial — regras TISS/ANS ativas, IA clínica indisponível", a Topbar muda para `ai="partial"` (via store Zustand) e nada é bloqueado.
- Erros RFC 9457 (`ProblemDetails` de `@pep/contracts`) mapeados a mensagens pt-BR por `type`.

### Barra de comandos e atalhos

- Ctrl/Cmd+K abre (`cmdk`); busca paciente, guia, lote e ações (endpoint de busca da API + ações locais registradas por módulo).
- Comandos clínicos (`/receita`, `/exame`, `/atestado`, `/encaminhamento`) também funcionam inline dentro do editor Tiptap da evolução (tela 05).
- Atalhos de duas teclas (`G A`, `G F`, `G L`, `G S`) e de uma (`N`, `V`, `P`, `M`, `?`) via `react-hotkeys-hook`; desligados automaticamente em campos editáveis. `?` mostra todos.

### Notificações

- Lista paginada da API com abas por categoria; contador não lidas na Topbar.
- Atualização por polling de 60 s na P1. `# ponytail:` polling; SSE (`@microsoft/fetch-event-source`) quando houver notificação que precise chegar em segundos.

### Offline (somente prontuário)

- Service worker (`vite-plugin-pwa`) faz cache do shell, da fonte Inter Variable e dos assets. Dados não são cacheados globalmente.
- Ao abrir um atendimento, a UI guarda no IndexedDB (`idb`) o atendimento, a evolução atual e o modelo, cifrados com chave Web Crypto derivada da sessão (descartada no logout).
- Sem rede: a escrita da evolução vai para uma fila local (`pendentes`) e a UI mostra "Salvo neste dispositivo — sincroniza ao reconectar". Ao reconectar, envia em ordem com `Idempotency-Key`.
- Conflito (a versão no servidor mudou) nunca sobrescreve: a alteração local vira **adendo** (ADR-PEP-0015, ADR-PEP-0007).
- Fechar atendimento, assinar e chamar a Spike exigem rede.

### Tablet e mobile

- Breakpoints: `sm` 390, `md` 834, `lg` 1194, `xl` 1280+.
- Tablet: Sidebar vira menu (botão 44 px); painel de IA vira bottom sheet `vaul` (26a); mesa do faturista em lista de cartões com ações em lote (26d).
- Mobile: captura de voz (26b) e revisão de guia SP/SADT (26c) com botões de largura total; tabelas viram listas.
- Alvos de toque ≥ 44 px em toda a UI (já seguido pelo design).

### Acessibilidade e i18n

- Componentes Radix (foco, ARIA) e `@axe-core/playwright` em toda tela. `role="tab"`/`aria-selected` e `aria-pressed` reais nos componentes migrados.
- Formatação por `Intl` (`pt-BR`, `BRL`, fuso `America/Sao_Paulo`). Textos em `i18n/pt-BR.ts`; sem segundo idioma por enquanto.

### Ordem de migração das telas

| Etapa | Telas | Junto com |
| :--- | :--- | :--- |
| 1 (P0) | Base do design system, tokens Tailwind `@layer`, providers, Shell (Sidebar, Topbar), 01a/01b, roteamento, auth | RFC-PEP-000/001 |
| 2 (P1) | 02 Início, 04 Paciente, 03 Agenda | RFC-PEP-002/003 |
| 3 (P1) | 05 Prontuário (Tiptap), 08 Documentos, 09–11 Modelos (`@dnd-kit`), 26a (`vaul`) | RFC-PEP-004 |
| 4 (P1) | 12–15 Anti-glosa (SSE), 13 Guia SP/SADT, 26c | RFC-PEP-005 |
| 5 (P1) | 16–18 Mesa (TanStack Table/Virtual + `nuqs`), editor de guia, lotes, 26d; 25 (núcleo) | RFC-PEP-006/014 |
| 6 (P2) | 06/07 Voz e revisão, 26b; 19–24 | RFC-PEP-007…013 |

A tela estática correspondente em `apps/ui/src/modulos/` é substituída pela versão ligada à API quando a nova entra e passa no teste de regressão visual (as telas geradas de `pages/screens/` foram apagadas em 09/11/2026).

### Critério "tela pronta"

- [ ] Usa só componentes do design system e o tema (nenhum estilo inline do arquivo gerado).
- [ ] Dados reais pela API usando contratos de `@pep/contracts`; estados vazio, carregando, erro (e parcial, se aplicável).
- [ ] Permissões respeitadas (itens e ações ocultos/desabilitados).
- [ ] `axe` sem violações; navegação completa por teclado.
- [ ] Conferida contra o [canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d) (fonte visual canônica). Diferença visual contra a tela do design dentro da tolerância do teste visual, em 1440 px e, se houver variante, em tablet/mobile.
- [ ] Teste E2E do fluxo principal da tela.

## UI

Esta RFC é a própria UI; comportamento por tela fica na RFC do domínio.

## Segurança/LGPD

- Tokens em memória (store Zustand); refresh via cookie `HttpOnly` do backend, nunca em `localStorage`.
- IndexedDB só com o atendimento aberto, cifrado e apagado no logout e após 12 h.
- Mascaramento de PII por padrão (RF-UI-12); revelar chama a API, que registra quem e por quê.
- CSP restritiva (só origem própria e Keycloak; fontes auto-hospedadas via `@fontsource-variable/inter`); sem `dangerouslySetInnerHTML` com dado de usuário.

## Critérios de aceite

- [x] Base `@pep/ui` configurada com Tailwind CSS em `@layer`, tokens do design, `@fontsource-variable/inter` (peso 650), `cva`/`cn`, TanStack Query, React Router + `nuqs`, Zustand e bibliotecas aprovadas sem quebrar as 26 telas.
- [ ] Tema validado com lint proibindo cores literais nas telas migradas.
- [ ] Shell, barra de comandos e notificações ligados à API em `staging`.
- [ ] Teste E2E: queda de rede durante a escrita da evolução → reconecta → evolução salva; conflito vira adendo.
- [ ] Tela 14: com a Spike simulada degradada, a guia continua editável e o aviso aparece.
- [ ] `axe` sem violações nas telas migradas; Lighthouse PWA instalável.
- [ ] Telas 26a–d funcionando nos tamanhos do design.

## Perguntas abertas

| # | Pergunta | Bloqueia |
| :--- | :--- | :--- |
| Q1 | Modo escuro entra no escopo? (o design só tem claro; variáveis CSS já preparadas) | Tema |
| Q2 | _(Resolvida na RFC-PEP-001: Keycloakify reaproveita os tokens e componentes React de `@pep/ui` para as etapas 1 e 2; etapa 3 roda no app)_ | — |
| Q3 | Offline também para a agenda da recepção, ou só prontuário? | RF-UI-10 |

## Notas de implementação

07/11/2026:
- UI movida para `apps/ui` (`@pep/ui`) no monorepo `pnpm` + `Turborepo` (ADR-PEP-0018), integrada a `@pep/contracts`.
- Removido link externo do Google Fonts em `index.html` e adotado `@fontsource-variable/inter` local (habilitando `font-weight: 650` fiel ao design, compatível com CSP estrita e offline).
- Configurado Tailwind CSS v4 (`@tailwindcss/vite`) com `@layer theme, base, legacy, components, utilities` preservando 100% da fidelidade das telas `.sc04`…`.sc26`, primitivos base `ds/` (`cn`, `Button`, `Badge`, `Card`), providers (`QueryClientProvider`, `NuqsAdapter`, `Toaster`), store Zustand (`useUiStore`), `react-router` e demais pacotes da stack alvo.

**Migração completa das telas 01–26 para Tailwind + shadcn (09/11/2026).**

- Todas as telas estáticas geradas (`pages/screens/*` + `.sNN` CSS) e as páginas escritas à mão foram reescritas como componentes com utilitários Tailwind v4 e primitivos de `src/ds/` (Button, Badge, Kbd, Chip, Tabs/Radix, Segmented, Table, Input/Label/Sec/H1/H2, Card, Dialog). Os arquivos geradas foram apagados; as telas agora vivem em `src/modulos/<dominio>/` (`agenda`, `prontuario`, `atendimento`, `modelos`, `antiglosa`, `faturamento`, `financeiro`, `plataforma`, `dispositivos`) e as páginas iniciais seguem em `src/pages/`.
- O bloco `@layer legacy` e a camada `screens` foram removidos; sobrou uma base mínima em `@layer base` (`box-sizing`, `body`, cor de links, margem zero de `h1/h2/p`). O remendo de abas/segmentados do `App.tsx` saiu (abas agora são Radix; segmentados usam `Segmented`).
- Tokens novos no `@theme` (`ink-2`, `wash`, `surface`, `line-soft`, `teal-ink`, `err/warn/ok/info` e seus fundos, `navy-*`, `ambar`). Os `line-height` dos tamanhos de texto do Tailwind foram fixados em 1,45 como no design.
- **Achado de fidelidade:** o CSS gerado pedia `font-family: Inter`, que não é carregada (só `Inter Variable`), então as telas estáticas renderizavam na fonte do sistema. Com `font-sans` (Inter Variable) a diferença contra o design caiu de 3–5% para 0,2–0,4% por tela. A Sidebar e a Topbar também foram refeitas (a Topbar antiga estava 4 px deslocada).
- **Verificação:** diff visual pixel a pixel contra as telas do design renderizadas localmente (1440 px). As telas ligadas a dados reais (agenda, mesa do faturista, modos, painel do gestor, atendimento) divergem do design apenas no conteúdo.
- **Desvios assumidos:** itens que dependem de fases futuras (P1/P2) aparecem como no design, mas inertes (`aria-disabled` + `title`); o contador de Faturamento no menu vem da fila real da unidade (o de Glosas segue fixo até a RFC-PEP-007); `GET /api/v1/guias` ganhou `limite` (padrão 100, máx. 200).
- Pendente: lint que proíba cores literais fora do tema nas telas migradas (várias telas ainda usam valores arbitrários `[#hex]` pontuais) e teste visual automatizado no CI (hoje o script de comparação é local).

**Selo "Prévia" no menu lateral (09/11/2026).** Os itens que levam a telas estáticas do design (Pacientes, Prontuário, Glosas, Financeiro, Contratos e Configurações) exibem o selo "Prévia" na `Sidebar` (`previa: true` em `NAV`, `Layout.tsx`); Agenda, Faturamento e Gestão têm fatia real no Marco D e ficam sem selo. Fecha o critério "toda tela não implementada marcada como Prévia" do release plan §12.8 para o menu; os demais pontos do D5 continuam abertos. Quando uma RFC migrar a tela para dados reais, basta remover `previa` do item.

**Onda 0 do MVP (09/11/2026): limpeza de telas órfãs.** Removidas as telas estáticas que duplicavam fluxos reais (`/prontuario`, `/voz`, `/revisao`, `/documentos`, `/antiglosa`, `/guia`, `/parcial`); `/prontuario` redireciona para a busca de pacientes. "Início" ganhou o selo "Prévia" (é estática até a onda 2). Primeiro uso de `nuqs` (busca e aba da tela 04) e do `usePode` (`auth/auth-store.ts`), que esconde ações sem permissão; o filtro do menu por permissão segue na onda 6. O canvas do design passou a ser citado no `CLAUDE.md` e no critério "tela pronta" como fonte visual canônica.

**Marca Tissia (09/11/2026).** O produto passou a se chamar Tissia ([[marca]]). Na UI: componentes `Logotipo`, `LogotipoPositivo` e `Simbolo` (`ds/marca.tsx`), fonte Fraunces (`--font-serif`) e tokens `agua` e `bruma`; barra lateral e telas de login (01a, 01b e o login real) com o novo logotipo, título editorial em Fraunces e "Entrar na Tissia"; trilha do tablet (26d) com o símbolo; "MedScribe IA" virou "Tissia IA" no painel, em Configurações e no tablet; `<title>`, favicon, ícones do app e manifesto PWA. O backoffice ganhou a mesma identidade em cópia própria (`apps/backoffice/src/Marca.tsx`). Pendente: sem lint que proíba o logotipo desenhado à mão; teste visual do login contra o canvas ainda é manual.

- **Breadcrumb removido (13/11/2026).** O design tirou a trilha da barra superior (PEP e backoffice): a `Topbar` começa pela busca global e o `Shell` não recebe mais `crumb`. O status mostra "IA ativa · TISS 4.03.00 · TUSS 202609" (versões em `apps/ui/src/ans.ts` até a RFC-PEP-021 trazer o manifesto) e a variante "base defasada: CADOP", hoje só nas telas de prévia, porque depende de `GET /referencias/status`.

- **Onda 6, fatia 1 (14/11/2026): guarda de rota e menu por permissão.** `ROTAS_UI`/`permissaoDaRota` em `@pep/contracts` (`rotas.ts`) é a fonte única de rota → permissão: `RequerSessao` mostra "Sem acesso a esta tela" (sem redirecionar) e a `Sidebar` esconde itens e grupos sem permissão; o selo da fila só consulta com `guias:ler`. Permissões continuam simples (`recurso:ação`); os escopos da RFC-PEP-014 §1.1 (`proprios | todos | resumo`) esperam a Q3 (CMO). Teste de matriz perfil × rota em `packages/contracts` e E2E por perfil em `apps/e2e/specs-demo/perfis.spec.ts` (recepção e faturista).

- **Onda 6, fatia 4 (14/11/2026): notificações reais.** O sino da Topbar (`CentralNotificacoes.tsx`, Radix Popover) troca o "5" fixo pela central da API: contador de não lidas, abas Todas/Clínico/Faturamento/Sistema, "Marcar todas como lidas" e clique que marca e abre o link; polling de 60 s e reconsulta ao abrir. Backend no módulo `notificacoes`: o consumidor `notificacoes` do outbox (idempotente) cria `GuiaRascunhoCriada` → "Nova guia na fila de faturamento" (público `guias:escrever`) e `LoteGerado` → "Lote gerado" (`guias:ler`); `GET /api/v1/notificacoes`, `POST /notificacoes/:id/leitura` e `POST /notificacoes/leitura`. A notificação é do tenant e visível a quem tem a permissão de público (sem uma linha por destinatário); a leitura é por usuário (`notificacoes_leituras`); janela de 30 dias. Só título, link e ids. Isso entrega a "guia nova" fora da Mesa. **Pendente:** as abas Clínico e Sistema ainda não têm origem (eventos clínicos e de IA degradada entram com as RFCs de cada domínio) e o painel "Notificações" da Home (tela 02) segue estático (Prévia). Testes: `apps/api/test/notificacoes.spec.ts` e E2E `apps/e2e/specs-demo/notificacoes.spec.ts`.

- **Onda 6, fatia 5 (14/11/2026): barra de comandos e atalhos reais.** `CommandPalette.tsx` (cmdk + Radix Dialog), montada uma vez no `App`, substitui a paleta estática da Home e a do atalho que ninguém renderizava: `Ctrl/Cmd+K` (e o campo de busca da Topbar em qualquer tela) oferece "Ir para" só com o que o perfil alcança (`permissaoDaRota`), busca de pacientes (API, debounce de 250 ms) e de lotes por número, e os comandos do prontuário (`/receita`, `/exame`) visíveis e inertes (`aria-disabled`, são do editor do atendimento, RFC-PEP-004). Atalhos de duas teclas `G I/A/P/F/L/G/M/S` e `?` (lista dos atalhos do perfil), desligados em campos de texto; o `?` usa a tecla produzida (`useKey`), não a posição, por causa do ABNT2. Os destinos filtram na hora (o debounce é só da busca na API), senão Enter logo após digitar ia ao item errado. **Pendente:** busca de guia por número (não há endpoint de busca; a Mesa só filtra), "recentes" e os atalhos de uma tecla (`N`, `V`, `P`, `M`) e de decisão (`A/E/I`) que dependem das telas de cada domínio. E2E: `apps/e2e/specs-demo/comandos.spec.ts`.
