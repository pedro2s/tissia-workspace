---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - design
  - identidade-visual
origem: tissia/docs/marca.md
---

# Marca Tissia

Fonte visual: página **00 · Marca** do [canvas do design](https://claude.ai/code/artifact/f1aa775e-58db-457c-b094-c147a8d8b48d). Em 09/11/2026 o produto deixou de se chamar "MedScribe" (nome já usado por terceiros) e passou a **Tissia**; o backoffice, a Spike e todo o texto dos docs seguem o novo nome. A página abaixo resume as regras para quem implementa; em dúvida, vale o canvas.

## Nome

| Onde | Como escrever |
| :--- | :--- |
| Texto corrido e interface | **Tissia** ("Entrar na Tissia", "a Tissia atua como operadora") |
| O produto de prontuário, em contraste com a Spike ou com outros PEPs | **PEP Tissia** |
| A IA anti-glosa | **Tissia IA**; o codinome técnico **Spike** continua nos docs e no backoffice ("Tissia IA (Spike)") |
| Backoffice interno | **Tissia Backoffice** |
| Repositório da Spike | `tissia-ai` (antes `medscribe-ai`) |
| Prefixos de documento e pacote | `PEP`, `ADR-PEP-NNNN`, `RFC-PEP-NNN`, `@pep/*` **não mudam**: "PEP" é a categoria do produto, não a marca |

## Logotipo e símbolo

- **Logotipo:** "tıssıa" em Fraunces Semibold, entreletra −1,5%, **sempre minúsculo**, com os pontos dos dois "i" (letra sem pingo, `ı`) em Água `#5FC3CC` no fundo escuro ou Petróleo `#0B6E78` no fundo claro. Os pontos são o profissional e a IA, lado a lado. Nunca em vermelho, âmbar ou verde (reservados aos status).
- **Símbolo:** "t" em Fraunces dentro de um quadrado arredondado Petróleo (ou Tinta, no fundo escuro), com o ponto Bruma `#A8E3E8` no canto. Ícone do app, favicon e barra lateral recolhida (tablet).
- **Tamanho mínimo** do logotipo: 20 px de altura na tela; abaixo disso, use o símbolo. **Respiro:** no mínimo a altura de um "s" em volta.
- No código: `Logotipo`, `LogotipoPositivo` e `Simbolo` em `apps/ui/src/ds/marca.tsx` (o backoffice tem a própria cópia em `apps/backoffice/src/Marca.tsx`, porque não importa de `@pep/ui`, [[ADR-PEP-0020-backoffice-app-separado|ADR-PEP-0020]]). O logotipo é uma imagem acessível com nome "Tissia"; o texto visual não é lido.

## Tipografia e cores

| Uso | Fonte |
| :--- | :--- |
| Marca e momentos editoriais (login, onboarding, relatórios ao gestor), com itálico para ênfase | Fraunces (`@fontsource-variable/fraunces`, eixos `wght` e `opsz`, sem CDN) |
| Toda a interface: dados, tabelas, formulários, guias; números tabulares | Inter |

| Cor | Hex | Token Tailwind |
| :--- | :--- | :--- |
| Tinta | `#0F1B2D` | `navy` |
| Petróleo | `#0B6E78` | `teal` |
| Água | `#5FC3CC` | `agua` |
| Bruma | `#A8E3E8` | `bruma` |
| Névoa | `#F4F6F8` | `bg` |

## Ícones do app

`apps/ui/public/`: `favicon-32.png`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png` e `icon-maskable-512.png` (manifesto PWA em `apps/ui/vite.config.ts`). Foram renderizados com a Fraunces a partir do símbolo do canvas 00; para refazer, renderize o mesmo símbolo nesses tamanhos (cheio e sem cantos arredondados no iOS e no "maskable", com margem de segurança de 15%). O backoffice reutiliza o `favicon-32.png`.
