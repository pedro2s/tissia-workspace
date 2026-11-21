---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
  - enfermagem
  - triagem
  - documentos
  - checkout
  - ia
aliases:
  - RFC-PEP-025
  - Estação de enfermagem, assinatura de exames, resumo IA e baixa de check-out
---

# RFC-PEP-025 — Estação Dedicada de Enfermagem, Assinatura de Pedidos de Exame, Resumo IA fora da Escuta e Baixa de Check-out

**Fase:** P2 (evolução pós-RFC-024; Onda C do plano pós-auditoria) · **Status:** Rascunho · **ADRs:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]], [[ADR-PEP-0030-separacao-pedido-externo-execucao-local-e-retorno|ADR-PEP-0030]], [[ADR-PEP-0031-separacao-papeis-checkout-recepcao-e-triagem|ADR-PEP-0031]]
**Telas:** 03 (Agenda — check-out com baixa), 05/08 (Documentos — assinatura/impressão de pedidos de exame e resumo IA), nova tela dedicada `/triagem` (Estação de Enfermagem)
**Requisitos:** RF-AGE-10..13, RF-PRO-11, RF-DOC-02, RF-DOC-05..07
**Depende de:** [[RFC-PEP-022-triagem-e-checkout-recepcao|RFC-PEP-022]], [[RFC-PEP-023-faturamento-clinico-execucao-vs-solicitacao-e-retorno|RFC-PEP-023]], [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024]], [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas|SPK-15]] (Parte 3 — Resumo IA a partir do SOAP)
**Relacionadas (Pós-Auditoria):** [[auditoria-clinica-regulatoria-ux-2026-11|Auditoria Clínica, Regulatória e UX (Nov/2026)]], [[RFC-PEP-026-blindagem-regulatoria-autoria-sigilo-e-governanca-ia|RFC-PEP-026]] (sigilo `documentos:imprimir_checkout`, versão append-only de triagens, segurança de alergias e validade documental), [[RFC-PEP-027-motor-tiss-autorizacao-outras-despesas-e-cbhpm|RFC-PEP-027]], [[RFC-PEP-028-jornada-hibrida-grande-clinica-e-sae-enfermagem|RFC-PEP-028]] (evolução SAE/COFEN e Sala de Medicação sobre `/triagem`), [[RFC-PEP-029-ergonomia-clinica-resiliencia-e-aceleradores-produtividade|RFC-PEP-029]] (impressão consolidada no check-out e impressão de resumo)

## Contexto e Motivação

Após a validação manual da [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024]] com `enfermagem.exemplo`, `dra.exemplo` e `recepcao.exemplo` (e corrigidas no PR `pedro2s/tissia#33` a remoção de `prontuario:escrever` da enfermagem, o bloqueio de edição de triagem pós-atendimento, a ordem `Anti-glosa → Sugestões` + deduplicação de indicação no `PainelIA` e os ajustes em `Documentos.tsx`), restaram **4 evoluções estruturais** essenciais para espelhar a rotina real de uma clínica:

1. **Estação dedicada de Enfermagem (`/triagem`) com registro real de alergias e medicações:**
   - A triagem em um drawer dentro da Agenda obriga a enfermagem a navegar na grade da recepção e oferece apenas checkboxes booleanos (`alergiasRevistas`, `medicacoesRevistas`), sem permitir registrar **quais** substâncias/medicamentos causam alergia ou estão em uso contínuo.
2. **Assinatura e impressão de Pedidos de Exame (`solicitação externa`):**
   - Pela prática clínica e regulatória brasileira, solicitações de exames entregues ao paciente exigem assinatura médica (CRM + data + assinatura eletrônica/ICP-Brasil) e via imprimível — inclusive para entrega pela recepção no check-out.
3. **Resumo para o paciente via IA também sem Escuta Ambiental:**
   - Hoje o texto sugerido do resumo ao paciente só existe quando o atendimento passou pela Escuta Ambiental (`escuta.resumoPaciente`). Em consultas digitadas manualmente no prontuário SOAP, o campo nasce vazio.
4. **Baixa explícita na fila de Check-out da Recepção:**
   - Pacientes atendidos permanecem indefinidamente na fila do check-out do dia, pois não existe ação explícita `"Concluir check-out"` para registrar que o paciente concluiu pagamento, retirou documentos e agendou retorno.

---

## Parte 1 — Estação Dedicada de Enfermagem (`/triagem`) e Anamnese de Segurança

### 1.1 Tela dedicada `/triagem` (`ROTAS_UI` com permissão `triagem:escrever`)
- Adicionar item **"Triagem"** na navegação principal para perfis com `triagem:escrever` (`enfermagem` e `medico`), tornando `/triagem` a página inicial padrão do perfil `enfermagem`.
- Layout em duas colunas (ou fila + painel de trabalho):
  - **Fila de espera (esquerda):** agendamentos do dia da unidade ativa com `status = 'chegou'` ordenados por horário de chegada (`aguardando triagem` primeiro, depois `triados` prontos para o consultório, e aba secundária `Atendidos` somente leitura).
  - **Formulário de pré-consulta (direita):** sinais vitais + IMC automático + queixa breve + **Anamnese de Segurança (Alergias e Medicações contínuas)**.

### 1.2 Registro estruturado de Alergias e Medicações Contínuas na Triagem
- **Alergias (`pacientes.alergias`):**
  - Exibir as alergias já registradas na ficha do paciente.
  - Permitir à enfermagem **adicionar novas alergias** (via combobox conectado ao catálogo Anvisa/CMED de `principio_ativo` / `nome_comercial` + opção de texto livre, ex.: `"Látex"`, `"Iodo"`, `"Dipirona"`) ou marcar explicitamente **"Nega alergias conhecidas (NKDA)"**.
  - Ao salvar a triagem, as alergias adicionadas são sincronizadas em `pacientes.alergias` (com trilha de auditoria) para acionar automaticamente o bloqueio de segurança na prescrição médica (`conflitosAlergia`).
- **Medicações em uso contínuo (`triagens.medicacoes_em_uso` jsonb):**
  - Substituir o checkbox isolado por lista estruturada opcional (seleção via `referenciasApi.buscarMedicamentos` ou texto livre + dose/frequência resumida) e opção **"Não faz uso contínuo"**.
  - Exibir alergias e medicações contínuas coletadas na triagem na faixa de contexto do prontuário médico (`Atendimento.tsx`).
- **Imutabilidade:** manter bloqueio `409 Conflict` na API quando o agendamento estiver em `atendido`, `cancelado`, `remarcado` ou `faltou` (e avaliar trava após início do atendimento médico com opção de nota complementar).

---

## Parte 2 — Assinatura e Impressão de Pedidos de Exame (Solicitação Externa)

- **Modelo e Contratos:**
  - Introduzir tipo documental `'pedido_exame'` (ou tabela/vínculo assinado sobre `pedidos_exame` / `documentos` com `tipo = 'pedido_exame'`) contendo os itens de `solicitacao_externa` (código TUSS, descrição, quantidade) e a indicação clínica única (Campo 23).
  - Exigir assinatura médica (`POST /api/v1/atendimentos/:id/documentos/assinatura`) antes de liberar a impressão oficial sem tarja de rascunho.
- **UI (`Documentos.tsx`, `FolhaDocumentoImpressao.tsx` e `CheckoutDrawer.tsx`):**
  - Em `CartaoExame`, permitir **Assinar pedido de exame** (gerando o documento assinado com CRM, data e assinatura eletrônica) e **Imprimir solicitação** diretamente da tela de Documentos.
  - Expor o pedido de exame assinado automaticamente em `GET /api/v1/agenda/checkout` e `/checkout/documentos/:documentoId/impressao`, permitindo à **recepção** imprimir a guia/pedido externo no balcão (sem expor SOAP).

---

## Parte 3 — Geração de Resumo para o Paciente via IA fora da Escuta

- **Integração `PEP` ↔ `tissia-ia` (*Contract-First*):**
  - Quando `Documentos.tsx` abrir um atendimento que **não** tem `escuta.resumoPaciente` nem documento `resumo_paciente` salvo, mas possui evolução SOAP salva (`a.evolucao`), disponibilizar geração via endpoint `POST /api/v1/atendimentos/:id/documentos/resumo-sugestao` (consumindo endpoint aditivo ou existente no `tissia-ia` em linguagem simples, após passar pelo scrubber de PII em `app/pii.py`).
  - Se a IA estiver indisponível/degradada (`200 OK`, `degradado: true`), degradar graciosamente para edição manual sem erro `5xx`.
  - Pré-carregar o texto retornado no `CartaoResumo` com `origem = 'ia_sugerida'` e trava obrigatória `"Revisei o resumo e autorizo o envio"`.

---

## Parte 4 — Conclusão Explícita de Check-out na Recepção

- **Migração aditiva (`agendamentos`):**
  - Adicionar `checkout_concluido_em timestamptz` e `checkout_concluido_por uuid` em `agendamentos`.
- **API (`CheckoutController` / `CheckoutService`):**
  - Novo endpoint `POST /api/v1/agenda/checkout/:agendamentoId/concluir` (`@RequerPermissao('agenda:ler')` ou `'agenda:escrever'`, auditado com `agenda.checkout_concluido`) gravando `checkout_concluido_em = now()`.
  - `ItemCheckoutAgenda` passa a refletir `concluido = Boolean(ag.checkoutConcluidoEm)` (e `checkoutConcluidoEm: string | null`), mantendo alertas visuais caso haja recebível particular ainda aberto antes de concluir.
- **UI (`PainelCheckoutDia` e `CheckoutDrawer`):**
  - Adicionar botão primário **"Concluir check-out"** no `CheckoutDrawer` (com aviso/confirmação caso particular ainda esteja pendente).
  - Separar `PainelCheckoutDia` em abas/filtros **Pendentes** (padrão — paciente sai da fila ao concluir) e **Concluídos** (histórico do dia para eventual reimpressão).

---

## Critérios de aceite

- [ ] **F1 (Enfermagem):** Perfil `enfermagem` acessa tela dedicada `/triagem`, visualiza fila do dia, registra alergias (atualizando `pacientes.alergias` e disparando alerta na receita médica) e medicações contínuas, sem qualquer acesso de escrita ao prontuário médico.
- [ ] **F2 (Pedidos de Exame):** Médico assina e imprime pedido de exame (solicitação externa) em `Documentos.tsx`; recepção consegue imprimir o pedido assinado pelo check-out sem acesso ao SOAP.
- [ ] **F3 (Resumo IA):** Atendimentos digitados manualmente (sem Escuta Ambiental) carregam sugestão de resumo em linguagem simples gerada a partir do SOAP, com fallback silencioso se a IA falhar.
- [ ] **F4 (Baixa Check-out):** Recepção conclui o check-out explicitamente no `CheckoutDrawer`, removendo o paciente da aba **Pendentes** e mantendo-o consultável em **Concluídos**.
