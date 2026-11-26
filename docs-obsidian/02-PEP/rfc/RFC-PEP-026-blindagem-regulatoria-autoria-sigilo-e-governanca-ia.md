---
projeto: PEP
tipo: rfc
status: implementada
tags:
  - tissia
  - pep
  - rfc
  - seguranca-clinica
  - lgpd
  - governanca-ia
  - autoria
aliases:
  - RFC-PEP-026
  - Blindagem regulatória autoria sigilo e governança de IA
---

# RFC-PEP-026 — Blindagem regulatória: autoria, sigilo e governança de IA

**Fase:** P1/P2 (Onda A — Bloqueante para Produção) · **Status:** Implementada (PR `pedro2s/tissia#34`, 21/11/2026)  
**ADRs:** [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]], [[ADR-PEP-0011-assinatura-icp-brasil|ADR-PEP-0011]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]], [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]], [[ADR-PEP-0032-autoria-clinica-sigilo-e-trilha-selada-obrigatoria|ADR-PEP-0032]], [[ADR-PEP-0033-governanca-ia-cfm-2454-e-seguranca-prescricao|ADR-PEP-0033]]  
**Depende de:** [[RFC-PEP-001-identidade-e-acesso|RFC-PEP-001]], [[RFC-PEP-004-prontuario|RFC-PEP-004]], [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]], [[RFC-PEP-013-voz-ambiental|RFC-PEP-013]], [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]], [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024]], [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas|SPK-15]]

## Objetivo e escopo

Fechar riscos bloqueantes de produção em quatro fatias implementáveis, com controles no backend e no banco, contratos estritos, rastreabilidade e evidência de aceite. Abrange prescrição, isolamento do atendimento na UI, autoria/assinatura, imutabilidade, credenciais, sigilo, direitos do titular e governança de IA/escuta ambiental.

As referências regulatórias que ainda dependem de parecer jurídico ou confirmação de vigência permanecem marcadas `[validar]`. A menção à Resolução CFM nº 2.454/2026 não é declaração de conformidade: confirmar existência, texto e aplicabilidade antes de usá-la como requisito normativo.

## Princípios de implementação

- Toda consulta ou mutação clínica aplica autorização no backend, escopo por clínica e paciente e RLS por `tenant_id`; a UI não substitui controles de acesso.
- Toda migração é aditiva. Tabelas novas têm `tenant_id uuid NOT NULL`, política RLS e acesso via `Db.comTenant`; dados existentes são preenchidos por backfill antes de tornar uma coluna obrigatória.
- Contratos de entrada e saída em `@pep/contracts` usam Zod `.strict()`. Campos de autoria são derivados da sessão, nunca aceitos como autoridade do cliente.
- Prontuário e evidências de decisão são append-only. Falha na gravação síncrona de auditoria aborta a operação.
- Não incluir PII ou texto clínico em logs técnicos. Não inventar códigos TUSS/CID; resolver apenas em bases vigentes e verificadas ou usar `<TUSS_…>`.

## Plano executável em quatro fatias

### F1 — Segurança Clínica Imediata: zero contaminação e prescrição segura

**Implementação**

1. Em `App.tsx`, montar a árvore de atendimento com `key={id}` baseada no `atendimentoId` efetivamente carregado. Em `Atendimento.tsx`, na troca de ID, cancelar requisições anteriores, limpar estado derivado, refs, buffers, formulários e instâncias TipTap antes de carregar o novo contexto. Cada resposta assíncrona valida que ainda pertence ao ID ativo.
2. Em `Documentos.tsx`, manter cabeçalho clínico sticky em todas as etapas de prescrição/documentos: nome, idade, peso, alergias destacadas e medicações em uso nos últimos 90 dias. Mostrar origem/data dos dados; indisponibilidade ou dado ausente é explícito, sem inferir ausência clínica.
3. Evoluir `packages/contracts/src/documentos.ts`: remover o banner verde de “sem conflito” da checagem lexical `conflitosAlergia`. O retorno distingue `conflito`, `sem_conflito_identificado` e `nao_avaliado`, com fonte/versão da regra. Criar mapeamento inicial curado por princípio ativo CMED e classes críticas beta-lactâmicos/penicilinas, AINEs, sulfas, cefalosporinas, opioides e contraste iodado. Não considerar essa lista exaustiva nem afirmar equivalências sem curadoria clínica. Desconhecido não vira resultado negativo.
4. Alertas críticos pedem justificativa obrigatória para override, com profissional, atendimento, medicamento/princípio ativo, regra e versão consultadas, justificativa e decisão em evento auditado. Alertas nunca prescrevem automaticamente.
5. `DocumentosService` bloqueia no servidor medicamento controlado ou antimicrobiano em receita simples até a ativação completa da [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]] e dos requisitos de emissão aplicáveis. O desbloqueio é por configuração auditável e validação do tipo de receita, não por flag controlada pela UI.

**Critérios de aceite F1**

- [ ] Ao alternar entre dois `atendimentoId`, nenhum texto, seleção, erro, estado de editor ou resposta atrasada do primeiro aparece ou grava no segundo.
- [ ] Cabeçalho sticky permanece visível nas telas de documentos e exibe origem/recência; alergia ou medicação ausente é “não informado”, não “sem alergia/medicação”.
- [ ] Correspondência lexical isolada nunca produz banner verde. Caso sem dados suficientes retorna `nao_avaliado`.
- [ ] Casos de princípio ativo e classes curadas geram alerta esperado; override sem justificativa é recusado no backend; override válido deixa trilha sem registrar conteúdo desnecessário em log.
- [ ] Tentativa de emitir controlado/antimicrobiano como receita simples é recusada no backend mesmo com chamada HTTP direta.

### F2 — Autoria médica, validade documental, imutabilidade e credenciais profissionais

**Implementação**

1. Validar que o ator pode agir como titular em `atendimentos.profissional_id`. Coassinatura/delegação de plantão exige concessão explícita, motivo, período e evento de auditoria. Adicionar `fechado_por` por migração aditiva; preencher fechamentos antigos quando a evidência existente permitir, mantendo nulo identificado como legado quando não permitir.
2. `FolhaDocumentoImpressao.tsx` e backend imprimem o signatário real de `documentos.assinado_por`, CRM/UF validado, unidade (CNES, CNPJ e endereço) e identificação do paciente. Bloquear rascunho não assinado. Preparar código verificador/QR sem alegar validade de assinatura ICP antes da RFC correspondente.
3. Fechado o atendimento, bloquear alteração e exclusão de `documentos`; documentos posteriores devem apontar para adendo/complementação. Aplicar a mesma imutabilidade a `resumo_paciente` após liberação. Triggers nas tabelas de `atendimentos` e `pedidos_exame` protegem campos clínicos definitivos. Corrigir o fluxo que cria adendo offline em atendimento aberto: conflito não pode transformar rascunho em adendo definitivo.
4. Criar `triagem_versoes` append-only e migrar leitura/escrita gradualmente; preservar autor e sinais vitais de cada versão. Proibir UPDATE/DELETE pelo papel da aplicação e registrar novas observações como nova versão.
5. Centralizar `DadosProfissionais` em fluxo de solicitação, curadoria/aprovação por administrador autorizado e validação perfil × conselho. Bloquear assinatura se registro profissional não estiver válido para o ato. Impedir admin de elevar o próprio perfil ou conceder a si perfil assistencial.

**Critérios de aceite F2**

- [ ] Cliente não consegue fechar, editar, pedir, assinar ou registrar evolução de outro profissional apenas trocando `profissionalId` no payload.
- [ ] Coassinatura/delegação inválida, expirada ou sem justificativa é recusada; a válida identifica separadamente titular e signatário.
- [ ] Impressão de documento assinado mostra signatário real e dados profissionais/unidade validados; rascunho não assinado não gera PDF.
- [ ] UPDATE/DELETE de documento, resumo liberado, versão de triagem ou conteúdo clínico após fechamento falha no backend e no banco; adendo cria novo registro vinculado.
- [ ] Nenhuma versão de triagem tem `autor_id` ou sinais vitais alterados após inserção.
- [ ] Admin não consegue autoescalar; profissional sem registro válido não assina.

### F3 — Sigilo clínico, trilha selada universal e direitos do titular (`lgpd:gerir`)

**Implementação**

1. Criar permissões `documentos:imprimir_checkout`, `lotes:baixar_xml` e `guias:ler_metadados`, e aplicar `prontuario:ler_trecho_guia` no pacote de recurso de glosa. O pacote inclui somente trecho aprovado, não SOAP integral. Retirar alergias de listagens administrativas e `visoes/pacientes`.
2. Registrar sincronamente em `auditoria_eventos`, na transação chamadora, todas as mutações clínicas, documentais, de IA, identidade e consentimento e as leituras clínicas `linha-do-tempo`, `triagem` e `guias/:id`. Incluir `ip` e `user_agent` na consulta/exportação autorizada do DPO. Falha de auditoria aborta leitura ou mutação. Outbox permanece apenas para distribuição posterior.
3. Cifrar `nomeAcompanhante` e `documentoAcompanhante` em `documentos.conteudo`, com chave gerida segundo padrão por tenant. Sanitizar logs de erro em `worker-runtime.ts` e `retorno.service.ts`; remover fallback `/tmp` de `tissia-ai.service.ts`, usando armazenamento temporário protegido e limpeza garantida ou fluxo em memória quando suportado.
4. Tornar `lgpd:gerir` funcional: fila de solicitações do titular (acesso, correção, portabilidade e informação), exportação do prontuário e trilha, guarda mínima de 20 anos conforme Lei 13.787/2018 `[validar]`, vínculo entre consentimentos e cada fluxo que os exige. Substituir painel estático ilustrativo em `Plataforma.tsx` por dados e ações reais com trilha.
5. Exportar apenas dados necessários ao propósito, com autorização explícita, integridade verificável, registro de solicitante e proteção contra exposição em logs. Pedidos de eliminação devem respeitar guarda legal e política de bloqueio/limitação de uso secundário validada juridicamente.

**Critérios de aceite F3**

- [ ] Testes de autorização provam que cada permissão libera somente a finalidade definida; metadados de guia não liberam XML, checkout não libera agenda genérica e pacote de glosa não contém SOAP integral.
- [ ] Listagens administrativas e `visoes/pacientes` não incluem alergias; visualização assistencial individual exige autorização e gera evento.
- [ ] Leituras e mutações clínicas enumeradas persistem evento com ator, recurso, paciente quando aplicável, data/hora, IP e agente do usuário na mesma transação; falha provoca rollback e resposta sem dado.
- [ ] Exportação DPO contém metadados de origem e integridade, inclui IP/agente do usuário disponíveis e exige `lgpd:gerir`/escopo autorizado.
- [ ] Busca automatizada de logs e payloads de erro não encontra nome/documento de acompanhante, PII nem texto clínico; falha do provedor não grava conteúdo em `/tmp`.
- [ ] Solicitação do titular percorre estados rastreáveis e produz exportação auditada; política de retenção está aprovada por jurídico antes do uso real.

### F4 — Governança de IA e consentimento da escuta ambiental

**Implementação**

1. Adicionar metadados versionados `modelo_nome`, `modelo_versao` e `prompt_versao` a `analises_tissia_ai` e `sessoes_escuta`. Criar `decisoes_analise` append-only para decisões de sugestões; persistir também sugestões proativas em sintonia contratual com [[SPK-14-sugestao-de-codigos-e-indicacao-a-partir-da-evolucao|SPK-14]] e [[SPK-15-metadados-modelo-resumo-soap-e-tabelas-tuss-despesas|SPK-15]]. Não persistir conteúdo clínico em auditoria técnica.
2. Tornar `trilhaAnexada` obrigatória para evolução assistida. Em `Revisao.tsx`, revisão por item/achado, diferenças visíveis e confirmação antes do aceite em massa; recusa e edição são decisões persistidas com ator e horário.
3. Unificar `consentimentos_gravacao` com `pacientes.consentimentos`, finalidade `escuta_ambiental`; exibir termo v2 completo antes de gravar e manter revogação durante captura. Revogação interrompe captura e inicia expurgo dos `transcricao_segmentos` vinculados, com resultado auditado e sem apagar evidência mínima do consentimento/revogação.
4. Descartar segmentos `clinico: false` antes de persistir. Remover `MODELO_ORTOPEDIA_JOELHO` hardcoded. `CONSULTA_EXEMPLO` em `Voz.tsx` só pode executar em ambiente explicitamente de demonstração, validado no servidor; produção bloqueia fixtures e dados fictícios.
5. Confirmar com jurídico aplicabilidade da Resolução CFM nº 2.454/2026 `[validar]`; registrar versão de política e mapa de requisitos quando validados. Nenhuma sugestão de IA substitui revisão, decisão e assinatura humanas.

**Critérios de aceite F4**

- [ ] Cada análise e sessão persistida identifica modelo e versões de prompt, sem depender de inferência posterior.
- [ ] Toda sugestão exibida tem decisão append-only (aceita, editada, rejeitada ou pendente) vinculada a ator e atendimento; registros não admitem UPDATE/DELETE pela aplicação.
- [ ] Evolução assistida sem trilha anexada é rejeitada; aceite em massa não ocorre antes da revisão granular e confirmação.
- [ ] Termo v2 é mostrado; revogação durante gravação interrompe captura, impede novos segmentos e inicia expurgo dos vinculados com status verificável.
- [ ] Segmento `clinico: false` não chega ao banco; fixture e modelo de consulta fictícia são inacessíveis em produção.
- [ ] Evidências de conformidade são aprovadas pelo responsável clínico e jurídico após validação normativa.

## DDL SQL aditivo de referência

O trecho é um contrato de desenho, não substitui revisão contra o schema Drizzle vigente. Tipos, nomes e chaves estrangeiras devem ser alinhados à implementação antes da migração. Não renomear nem remover colunas em uso. Tabelas novas são aditivas; qualquer backfill ocorre antes de `NOT NULL` em tabelas existentes.

```sql
-- Novas tabelas: tenant_id obrigatório e RLS por tenant.
CREATE TABLE triagem_versoes (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  triagem_id uuid NOT NULL,
  atendimento_id uuid NOT NULL,
  autor_id uuid NOT NULL,
  versao integer NOT NULL,
  sinais_vitais jsonb NOT NULL,
  criado_em timestamptz NOT NULL DEFAULT now(),
  UNIQUE (tenant_id, triagem_id, versao)
);

CREATE TABLE decisoes_analise (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  analise_id uuid NOT NULL,
  atendimento_id uuid NOT NULL,
  sugestao_ref text NOT NULL,
  decisao text NOT NULL CHECK (decisao IN ('aceita', 'editada', 'rejeitada')),
  ator_id uuid NOT NULL,
  justificativa text,
  criado_em timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE analises_tissia_ai
  ADD COLUMN modelo_nome text,
  ADD COLUMN modelo_versao text,
  ADD COLUMN prompt_versao text;

ALTER TABLE sessoes_escuta
  ADD COLUMN modelo_nome text,
  ADD COLUMN modelo_versao text,
  ADD COLUMN prompt_versao text;

ALTER TABLE atendimentos
  ADD COLUMN fechado_por uuid;

ALTER TABLE triagem_versoes ENABLE ROW LEVEL SECURITY;
ALTER TABLE decisoes_analise ENABLE ROW LEVEL SECURITY;

CREATE POLICY triagem_versoes_tenant_policy ON triagem_versoes
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);

CREATE POLICY decisoes_analise_tenant_policy ON decisoes_analise
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);

REVOKE UPDATE, DELETE ON triagem_versoes FROM app_rw;
REVOKE UPDATE, DELETE ON decisoes_analise FROM app_rw;
```

A implantação real deve adicionar chaves estrangeiras e índices compostos por tenant conforme o schema vigente, grants do papel da aplicação e triggers que rejeitem UPDATE/DELETE, seguindo [[ADR-PEP-0003-postgres-rls-multitenant|ADR-PEP-0003]], [[ADR-PEP-0017-orm-drizzle-rls|ADR-PEP-0017]] e [[ADR-PEP-0019-drizzle-kit-migracoes-e-seed|ADR-PEP-0019]]. Migrações existentes não são reescritas.

## Rotas e contratos

Rotas abaixo são contratos-alvo; alinhar prefixos e nomes à API efetiva antes de publicar OpenAPI. Toda entrada Zod usa `.strict()`. Identidade, tenant e autoria são obtidos do contexto autenticado, não do payload.

| Operação | Acesso | Entrada/saída principal | Auditoria |
| :--- | :--- | :--- | :--- |
| `POST /atendimentos/:id/documentos/:documentoId/imprimir-checkout` | `documentos:imprimir_checkout` e escopo de atendimento do dia | Entrada estrita sem ator; saída PDF/payload sanitizado e assinado | `documento.impressao_checkout` |
| `GET /guias/:id/metadados` | `guias:ler_metadados` | Metadados administrativos mínimos; sem XML nem SOAP | `guia.metadados_visualizados` |
| `POST /lotes/:id/xml/download` | `lotes:baixar_xml` | Saída XML do lote autorizado | `lote.xml_baixado` |
| `GET /guias/:id/recurso-glosa` | `prontuario:ler_trecho_guia` + acesso à guia | Trecho clínico aprovado e mínimo; nunca evolução integral | `guia.trecho_clinico_visualizado` |
| `POST /documentos/:id/conflitos/override` | profissional autorizado no atendimento | `{ justificativa: string }` estrito; decisão e identificador da regra | `prescricao.conflito_override` |
| `POST /atendimentos/:id/fechamento` | titular ou coautor/delegado válido | Sem `profissionalId` de autoridade; resposta com `fechado_por` | `atendimento.fechado` |
| `POST /atendimentos/:id/escuta/revogacao` | paciente/representante ou operador autorizado conforme fluxo | `{ consentimentoId: string, motivo?: string }` estrito | `escuta.consentimento_revogado` |
| `POST /lgpd/solicitacoes` / `GET /lgpd/solicitacoes/:id/exportacao` | `lgpd:gerir` e verificação do titular | tipo/escopo estritos; exportação autenticada e auditada | `lgpd.solicitacao_criada` / `lgpd.exportacao` |

Os schemas Zod de cada endpoint declaram `.strict()`, limites de tamanho, enums fechados e refinamentos de autorização. Campos como `tenantId`, `atorId`, `profissionalId`, `assinadoPor` e `ip` não são aceitos do cliente para definir identidade; são derivados do contexto confiável. IDs recebidos são revalidados dentro do tenant.

## Matriz RBAC atualizada

`—` significa sem permissão padrão. Escopo de recurso, vínculo profissional e autorização do backend são sempre aplicados. Admin administrativo não ganha acesso clínico por conveniência nem pode autoescalar.

| Ação/permissão | Médico titular | Enfermagem | Recepção | Faturista | Gestor | Admin | DPO |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Editar evolução/fechar atendimento | Próprio atendimento | — | — | — | — | — | — |
| Registrar triagem | — | Atendimento autorizado | — | — | — | — | — |
| Coassinar/delegar plantão | Concessão auditada | Conforme credencial/concessão | — | — | — | Não para si | — |
| `documentos:imprimir_checkout` | Atendimento próprio | — | Atendimento do dia, documento assinado | — | — | — | — |
| `guias:ler_metadados` | Guias vinculadas | — | — | Guias autorizadas | Metadados agregados/autorizados | Metadados operacionais | Conforme finalidade |
| `prontuario:ler_trecho_guia` | Trecho de atendimento próprio | — | — | Recurso de glosa autorizado | — | — | Conforme solicitação formal |
| `lotes:baixar_xml` | — | — | — | Lotes autorizados | — | — | — |
| Ver alergias | Atendimento assistencial próprio | Triagem assistencial autorizada | — | — | — | — | Apenas fluxo formal autorizado |
| `auditoria:ler` clínico identificável | Escopo próprio se concedido | — | — | — | — | Operacional mínimo | Escopo de DPO |
| `lgpd:gerir` | — | — | — | — | — | Gestão sem autoelevação | Solicitações/exportação autorizadas |
| Aprovar `DadosProfissionais` | Solicita/consulta próprio | Solicita/consulta próprio | — | — | — | Curadoria com trilha | — |

Perfis e escopos finais devem ser materializados em `PERMISSOES_POR_PERFIL`, guardas de rota e testes de matriz; esta tabela é o requisito funcional de referência.

## Fluxo de dados e auditoria

```mermaid
sequenceDiagram
    actor Profissional
    participant UI as Interface clínica
    participant API as API PEP
    participant DB as PostgreSQL com RLS
    participant AUD as auditoria_eventos
    participant IA as tissia-ia

    Profissional->>UI: Abre atendimento e revisa contexto
    UI->>API: Leitura autenticada com atendimentoId
    API->>DB: Valida tenant, vínculo e escopo
    API->>AUD: Registra leitura na mesma transação
    API->>UI: Retorna somente dados autorizados
    Profissional->>UI: Solicita sugestão ou prescrição
    UI->>API: Payload estrito sem autoria declarada
    API->>DB: Valida alergias, regra e estado do atendimento
    API->>IA: Envia contexto minimizado conforme contrato
    IA-->>API: Sugestão e metadados versionados
    API->>DB: Persiste sugestão/decisão append-only
    API->>AUD: Registra evento na transação de negócio
    API-->>UI: Apresenta para revisão humana
    Profissional->>UI: Aceita, edita ou rejeita
    UI->>API: Decisão granular e justificativa quando exigida
    API->>DB: Persiste decisão; fecha ou assina conforme autorização
    API->>AUD: Registra autoria e resultado sincronamente
```

## Dependências e ordem de entrega

1. F1 pode começar após confirmação do modelo de atendimento e fonte de princípio ativo com curadoria clínica.
2. F2 depende dos contratos de identidade, prontuário e assinatura; compatibilizar rollout de coluna `fechado_por` e backfill.
3. F3 depende de `Db.comTenant`, RLS e capacidade transacional de `AuditoriaService`; separar privilégio administrativo de acesso assistencial.
4. F4 depende dos contratos vigentes SPK-14/SPK-15 e do fluxo de consentimento da RFC-PEP-013. Mudança de API externa exige atualização contract-first e snapshot correspondente.
5. Cada fatia deve ter migração aditiva, contrato, matriz de autorização, evidência de auditoria e atualização documental antes de habilitação em produção.

## Riscos

- Classificação de alergia incompleta ou fonte desatualizada pode gerar falso negativo; começar conservador, explicitar desconhecidos e submeter catálogo à curadoria clínica.
- Auditoria síncrona adiciona latência e pode aumentar indisponibilidade. Medir desempenho, particionamento e capacidade sem permitir leitura clínica sem trilha.
- Backfill de `fechado_por`, autoria de documentos e dados profissionais pode não ter evidência histórica suficiente. Preservar legado como desconhecido, sem inferir identidade.
- Cifragem de campos e exportação LGPD exigem gestão de chaves, rotação, controles de acesso e análise de impacto operacional.
- Expurgo após revogação pode conflitar com obrigação de guarda ou evidência; definir retenção mínima com jurídico, preservando metadados estritamente necessários.
- Resolução CFM nº 2.454/2026 e prazo de guarda de prontuário: **[validar]** vigência, alcance e interpretação jurídica antes de alegar conformidade.
- Nomes de rotas, colunas e relações deste desenho precisam ser conferidos no schema e OpenAPI vigentes antes de gerar migração ou contrato publicado.

## Perguntas abertas

| # | Pergunta | Responsável |
| :--- | :--- | :--- |
| Q1 | Qual fonte oficial e processo de atualização serão usados para princípio ativo e classe terapêutica? | CMO/Farmácia clínica |
| Q2 | Quais atos admitem delegação/coassinatura de plantão e como validar conselho, escopo e vigência? | CMO/Jurídico |
| Q3 | Qual formato e prazo de guarda para áudio, segmentos, sugestões de IA e evidência mínima de revogação? | Jurídico/DPO |
| Q4 | A Resolução CFM nº 2.454/2026 existe, está vigente e se aplica a cada fluxo descrito? | Jurídico/Compliance |
| Q5 | Qual estratégia de expurgo de transcrições já processadas respeita revogação e guarda legal? | DPO/Engenharia |
| Q6 | O modelo atual de `auditoria_eventos` comporta volume de leitura síncrona, IP/agente e consulta/exportação por DPO? | Arquitetura/SRE |
| Q7 | Qual política de validação, expiração e recertificação de CRM/COREN/CBO será fonte de bloqueio de assinatura? | Operações clínicas |

## Referências

- [[ADR-PEP-0032-autoria-clinica-sigilo-e-trilha-selada-obrigatoria|ADR-PEP-0032]] e [[ADR-PEP-0033-governanca-ia-cfm-2454-e-seguranca-prescricao|ADR-PEP-0033]].
- [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]], [[RFC-PEP-024-ux-escuta-revisao-e-documentos-clinicos|RFC-PEP-024]] e [[modelo-de-dados-pep|modelo de dados PEP]].
- Lei nº 13.787/2018 e LGPD: interpretação e prazos específicos sujeitos a validação jurídica.

## Notas de implementação

**Entrega consolidada — 21/11/2026 (PR `pedro2s/tissia#34`, branch `feat/rfc-026-integrada`, commit `fc340a6`):**

- **Migrações Drizzle sincronizadas (`db:generate` sem drift):**
  - `0051_rfc026_schema.sql` (`fechado_por`, `triagem_versoes`, `lgpd_solicitacoes`, `decisoes_analise_log`, `sugestoes_ia_log`, colunas `modelo_nome`/`modelo_versao`/`prompt_versao`/`consentimento_revogado_em` e constraint `sessao_estado` com `'cancelada'`).
  - `0052_rfc026_triggers_imutabilidade.sql` (triggers imutáveis em `triagem_versoes`, `atendimentos`, `pedidos_exame`, `documentos` incluindo `resumo_paciente` liberado, `decisoes_analise_log` e `sugestoes_ia_log`, além de `REVOKE UPDATE, DELETE, TRUNCATE ... FROM app_rw`). RLS aplicada por convenção em todas as novas tabelas via `drizzle/rls/001_tenant_policies.sql`.
- **Alinhamentos e desvios pontuais em relação às rotas-alvo:**
  - Mantidos os paths já existentes na API para preservar compatibilidade retroativa com `@pep/ui`: `GET /api/v1/agenda/checkout/documentos/:documentoId/impressao` (agora exigindo `documentos:imprimir_checkout`), `GET /api/v1/lotes/:id/xml` (agora exigindo `lotes:baixar_xml`), `GET /api/v1/guias/:id/metadados` (`guias:ler_metadados`), `POST /api/v1/escuta/:sid/revogar`, `GET/POST /api/v1/lgpd/solicitacoes`, `PATCH /api/v1/lgpd/solicitacoes/:id` e `GET /api/v1/lgpd/solicitacoes/:id/exportacao`.
  - O override de conflito de alergia é enviado no campo aditivo `overrideAlergiaJustificativa` de `ReceitaSchema` (validado >= 10 chars no servidor em `assinar` com auditoria síncrona `prescricao.conflito_override`).
  - Delegação/coassinatura formal de plantão entre médicos e fluxo de aprovação assíncrona de `DadosProfissionais` por fila de admin ficam como evolução incremental após o bloqueio imediato de titularidade, perfil × conselho e anti-auto-escalação já entregues nesta RFC.
- **Verificação observada (PostgreSQL zerado + CI-equivalente):** `pnpm --filter @pep/api db:generate` (zero drift), `db:setup` + RLS, `pnpm typecheck`, `pnpm deps` (0 violações), `pnpm build` e `pnpm test` completo (**664/664 testes**: `@pep/contracts` 7/61, `@pep/backoffice` 1/3, `@pep/ui` 23/125, `@pep/api` 50/475 incluindo `rfc026-f1..f4` e `openapi.spec.ts`).

### Atualização de 26/11/2026 — auditoria clínica (PR `pedro2s/tissia#74`, em draft)

- **Ato médico:** quem não é o médico titular recebe `403` (RFC 9457) ao registrar aceite, edição ou rejeição de itens de diagnóstico (`sug_cid:`, `cid:`, `ia_cid:`). O faturista continua podendo sinalizar divergência em justificativa e alertas.
- **Human-in-the-loop de CID:** `salvarEvolucao` rejeita com `422` (`conteudo.cids`) CID sugerido pela IA sem decisão humana registrada em `decisoes_sugestao_ia`. A escuta registra o aceite dos CIDs selecionados na mesma transação.
- **Autoria:** marcas já gravadas e as geradas no servidor nunca são removidas; as enviadas pelo cliente só acrescentam. Isso preserva a origem IA do fluxo do editor, mas **não impede** que um cliente adulterado deixe de marcar um trecho novo.
- **Migração `0083_cids_sugeridos_log.sql`** (aditiva): `sugestoes_ia_log.cids` guarda só códigos, sem texto clínico.
- **Risco aberto:** o UI posta a decisão do CID de forma assíncrona; se o autosave da evolução chegar antes, o médico recebe `422`. Verificar o fluxo no UI antes do merge.
- Sem mudança de contrato com a `tissia-ia`.
