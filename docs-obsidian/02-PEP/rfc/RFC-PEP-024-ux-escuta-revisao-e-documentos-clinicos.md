---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
  - prontuario
  - voz-ambiental
  - documentos
  - ux
aliases:
  - RFC-PEP-024
  - Evolução UX Escuta, Revisão e Documentos
---

# RFC-PEP-024 — Evolução de UX Clínica: Escuta Ambiental, Revisão, Documentos e Catálogos

**Fase:** P2 (evolução pós-P1 / pós-RFC-022 e 023) · **Status:** Rascunho · **ADRs:** [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0024-bases-ans-vigentes-sem-inventar-codigos|ADR-PEP-0024]], [[ADR-PEP-0027-indicacao-clinica-unica-por-atendimento|ADR-PEP-0027]]
**Telas:** 05 (Prontuário — botão Finalizar integrado e editor em bloco), 06 (Escuta Ambiental), 07 (Revisão Pós-Consulta), 08 (Documentos do Atendimento)
**Requisitos:** RF-PRO-05..10, RF-ENV-03..04, RF-DOC-01..06
**Depende de:** [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]], [[RFC-PEP-021-dados-de-referencia-ans|RFC-PEP-021]], [[RFC-PEP-022-triagem-e-checkout-recepcao|RFC-PEP-022]], [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023]]

## Contexto e Motivação

Após validação manual completa dos fluxos assistidos por escuta ambiental, revisão pós-consulta e emissão de documentos (em sequência aos ajustes P0 entregues no PR `pedro2s/tissia#32`), identificaram-se atritos críticos de experiência clínica e lacunas de modelo de dados que impedem o médico de percorrer o atendimento de forma natural:

1. **Encerramento da escuta sem feedback claro (`Voz.tsx`):** `POST /api/v1/atendimentos/escuta/:sid/encerrar` processa transcrição e estruturação por IA de forma síncrona e pode levar vários segundos exibindo apenas `"Encerrando…"` no botão.
2. **Cliques manuais desnecessários em sugestões já prontas (`Revisao.tsx` e `Documentos.tsx`):**
   - A indicação clínica derivada dos trechos S/O/A nasce vazia, bloqueando o uso dos pedidos sugeridos até que o médico clique em `"Usar sugestão"`.
   - O **Resumo para o paciente** nasce vazio, exigindo um clique adicional em `"Usar resumo sugerido pela IA"` quando a escuta já o gerou.
3. **Inversão do momento de imutabilidade no fluxo com escuta (`Revisao.tsx` → `Documentos.tsx`):**
   - Hoje `"Assinar e finalizar"` na revisão trava o prontuário (tornando a evolução imutável e emitindo guia TISS) **antes** de o médico elaborar receita, atestado, pedidos finais e encaminhamento. Qualquer ajuste clínico percebido durante a prescrição obriga a abrir adendo, e sair de Documentos leva a uma tela de prontuário já *read-only*, sem CTA natural de conclusão.
4. **Restrição a documento único por tipo (`documentos/schema.ts`):**
   - A constraint `uniqueIndex('ux_documento_tipo')` sobre `(atendimento_id, tipo)` impede emitir mais de um atestado (ex.: menor de idade atendido que precisa de atestado escolar/afastamento + atestado de acompanhamento para o responsável), mais de um encaminhamento ou receitas separadas (simples vs controle especial). Além disso, `AtestadoSchema` na modalidade `'acompanhante'` não coleta o nome/identificação do acompanhante.
5. **Digitação livre em medicamentos e especialidades (`Documentos.tsx`):**
   - Receita e encaminhamento dependem de digitação manual pura em vez de catálogos sugestivos (Anvisa/CMED e especialidades médicas CFM), reduzindo agilidade e limitando a checagem de alergias.

---

## Parte 1 (Fatia P1) — Fluxo Unificado Escuta → Revisão → Documentos → Finalização

### 1.1 Feedback de processamento ao encerrar escuta (`Voz.tsx`)
- Ao acionar **Encerrar e revisar**, exibir overlay/painel de carregamento explícito com indicador visual (*spinner* + etapas progressivas: *"Finalizando transcrição…"* → *"Estruturando SOAP, CID-10 e resumo clínico…"*), impedindo cliques duplicados e orientando o médico a aguardar sem perda de contexto.
- Caso a IA entre em degradação/timeout, manter o comportamento resiliente (transicionar para revisão parcial ou prontuário manual com aviso claro).

### 1.2 Pré-carregamento assistido sem gravação silenciosa (`Revisao.tsx` e `Documentos.tsx`)
- **Indicação clínica na Revisão:** quando `!a.indicacaoPedidos` e houver texto derivado `indicacao.trim()`, pré-popular localmente o editor com esse texto, exibindo badge `"Sugerida pela escuta — revise antes de salvar"` e mantendo a regra de não gravar no banco até o médico clicar em `"Salvar indicação"` ou adicionar um pedido de exame.
- **Resumo para o paciente em Documentos:** quando `!doc` (ainda não salvo) e `sugerido` estiver disponível na sessão de escuta, inicializar o estado do card já preenchido com `sugerido` e `origem = 'ia_sugerida'`, com destaque visual de sugestão IA e removendo o botão redundante `"Usar resumo sugerido pela IA"`. O checkbox obrigatório `"Revisei o resumo e autorizo o envio"` continua travando o status `'revisado'` e qualquer envio futuro.

### 1.3 Novo fluxo clínico unificado: Revisão não fecha prematuramente o prontuário
Convergir os fluxos **manual** e **por escuta** para a mesma sequência clínica:

```text
Atendimento aberto (Manual OU Escuta -> Revisão)
   └─> [Aplicar revisão ao prontuário e abrir Documentos]  (prontuário AINDA editável)
          └─> Tela Documentos (Receita / Atestado / Pedidos / Resumo / Encaminhamento)
                 └─> [Assinar documentos] + [Finalizar atendimento]
                        └─> Imutabilidade SOAP + Guias TISS / Check-out Recepção
```

- **Novo endpoint / ação na `Revisao.tsx`:**
  - Introduzir `POST /api/v1/atendimentos/escuta/:sid/aplicar` (ou flag explícita `finalizarAtendimento?: boolean` em `FinalizarEscutaSchema`, mantendo retrocompatibilidade `default: true` na API e passando `false` na UI).
  - Quando `finalizarAtendimento = false`, grava evolução revisada, trilha de auditoria/autoria dos trechos, CIDs e gera rascunho de evolução no atendimento **mantendo `atendimento.status = 'aberto'`**.
  - **Botão primário na `Revisao.tsx`:** `"Aplicar ao prontuário e ir para Documentos"`.
  - **Ação secundária opcional:** `"Aplicar e finalizar atendimento"` (para consultas simples sem receita, atestado ou encaminhamento).
- **Conclusão direta em `Documentos.tsx`:**
  - Quando `atendimento.status !== 'fechado'`, exibir na barra de cabeçalho/rodapé de `Documentos.tsx` o botão primário **"Finalizar atendimento"** (ao lado de `"Voltar ao prontuário"`), executando `atendimentoApi.fechar` (respeitando janela de retorno/particular) após confirmar se há documentos em rascunho não assinados (oferecendo `"Assinar pendentes e finalizar"`).
  - Ao finalizar, redirecionar para a tela de resumo/prontuário fechado ou oferecer ações diretas (**Abrir guia**, **Ver na Mesa**, **Voltar à agenda**).

---

## Parte 2 (Fatia P2) — Múltiplos Documentos, Acompanhante e Catálogos Assistivos

### 2.1 Múltiplos atestados, receitas e encaminhamentos por atendimento
- **Migração aditiva de banco (`documentos`):**
  - Substituir `uniqueIndex('ux_documento_tipo').on(t.atendimentoId, t.tipo)` por índice único parcial restrito a `'resumo_paciente'`:
    `CREATE UNIQUE INDEX ux_documento_resumo_unico ON documentos (atendimento_id) WHERE tipo = 'resumo_paciente';`
  - Para `'receita'`, `'atestado'` e `'encaminhamento'`, permitir múltiplos registros no mesmo atendimento, identificados por `id` (UUID v7).
- **Contratos e API (`packages/contracts/src/documentos.ts` e `documentos.controller.ts`):**
  - Evoluir `AtestadoSchema` adicionando campos opcionais na modalidade `'acompanhante'`:
    - `nomeAcompanhante?: string` (`min(2).max(150)`)
    - `documentoAcompanhante?: string` (`max(30)`, opcional — RG/CPF exibido apenas no impresso quando informado)
    - `parentesco?: string` (`max(40)`, opcional)
  - Atualizar `textoAtestado` para incluir nominalmente o acompanhante (ex.: *"Atesto, para os devidos fins, que [nomeAcompanhante] esteve presente como acompanhante do(a) paciente [paciente]..."*), mantendo fallback genérico apenas quando o campo estiver vazio em registros legados.
  - Adicionar `PUT /api/v1/atendimentos/:id/documentos/itens/:docId` e `DELETE /api/v1/atendimentos/:id/documentos/itens/:docId` (apenas rascunhos não assinados) ou `POST /api/v1/atendimentos/:id/documentos/:tipo` criando novo item, preservando `PUT /api/v1/atendimentos/:id/documentos/:tipo` quando `documentoId` não for informado para retrocompatibilidade com clientes existentes.
- **UI (`Documentos.tsx` e `CheckoutDrawer.tsx`):**
  - Permitir lista de cartões/abas na seção de **Atestado** (com botão `"+ Adicionar atestado"` — exemplo de uso: 1 para o menor + 1 de acompanhante para o responsável), bem como `"+ Nova receita"` e `"+ Novo encaminhamento"`.
  - No `CheckoutService` / `CheckoutDrawer` (que já usam `d.id` único em `/checkout/documentos/:documentoId/imprimir`), enriquecer `DocumentoCheckout.rotulo` com subtítulo discriminador quando houver mais de 1 documento do mesmo tipo (ex.: `"Atestado · Comparecimento"` e `"Atestado · Acompanhante"`).

### 2.2 Select sugestivo de Especialidades (Encaminhamento)
- Criar catálogo estático curado das especialidades e áreas de atuação médicas reconhecidas pelo **CFM** (Resolução CFM vigente + CBO correspondente quando útil) em `@pep/contracts` ou `referencias`.
- Substituir o `<input id="enc-esp">` em `Documentos.tsx` por combobox com autocomplete (*tipo-ahead*), mantendo permissão de digitação livre caso o encaminhamento seja para área multiprofissional (ex.: Fisioterapia, Fonoaudiologia, Nutrição, Psicologia).

### 2.3 Base regulatória Anvisa / CMED de Medicamentos (Receita)
- **Ingestão (`apps/api/ref/` e módulo `referencias`, conforme `ADR-PEP-0024`):**
  - Adicionar tabela de referência somente leitura `medicamentos_anvisa` (princípio ativo / DCB, nome comercial, apresentação/concentração, tarja/controle especial, registro Anvisa, hash/versão em manifesto) populada via script dedicado em `apps/api/scripts/ref/`.
  - Expor endpoint `GET /api/v1/referencias/medicamentos?q=...&limite=20` com busca textual (`pg_trgm` / prefixo por nome comercial ou princípio ativo).
- **UI (`Documentos.tsx` -> `CartaoReceita`):**
  - Substituir o input livre de medicamento por combobox assíncrono (`BuscaCatalogo tipo="medicamento"`), preenchendo automaticamente nome padronizado + apresentação sugerida e permitindo entrada em texto livre para fórmulas manipuladas.
  - Evoluir `conflitosAlergia` para cruzar tanto o nome comercial quanto o **princípio ativo (DCB)** cadastrado no item selecionado.

---

## Segurança e LGPD / CFM

- O prontuário permanece estritamente imutável após a finalização do atendimento (`ADR-PEP-0007`), mas não é travado artificialmente antes de o médico concluir a emissão documental da mesma consulta.
- Documentos assinados continuam imutáveis (`409` ao tentar editar/excluir).
- Impressão no check-out pela recepção continua utilizando exclusivamente `/checkout/documentos/:documentoId/impressao` com metadados autorizados (sem SOAP/alergias/CID não autorizado).
- Nenhum nome comercial ou código de medicamento é inventado em código/testes fora da base oficial carregada (`ADR-PEP-0024`).

---

## Critérios de aceite

- [ ] **P1.1:** Encerrar escuta exibe indicador claro de carregamento até a abertura de `/revisao` (sem estado silencioso).
- [ ] **P1.2:** Na `Revisao.tsx`, indicação clínica sugerida já carrega preenchida localmente quando ausente no servidor; na `Documentos.tsx`, o resumo sugerido pela IA já carrega preenchido com marcação de revisão pendente.
- [ ] **P1.3:** O botão primário da `Revisao.tsx` aplica a evolução mantendo o atendimento aberto e abre `Documentos.tsx`; em `Documentos.tsx`, `"Finalizar atendimento"` fecha a evolução, gera as guias TISS aplicáveis e disponibiliza os documentos assinados para check-out.
- [ ] **P2.1:** É possível emitir e assinar dois atestados no mesmo atendimento (ex.: afastamento do paciente + acompanhante nominal) e imprimi-los individualmente no check-out da recepção.
- [ ] **P2.2:** Campo de especialidade no encaminhamento oferece autocomplete de especialidades CFM/multiprofissionais com fallback livre.
- [ ] **P2.3:** Campo de medicamento na receita sugere apresentações da base Anvisa/CMED carregada e detecta alergia por princípio ativo.
