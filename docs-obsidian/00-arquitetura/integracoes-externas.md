---
projeto: PEP
tipo: arquitetura
status: ativo
tags:
  - tissia
  - pep
  - arquitetura
  - integracoes
  - keycloak
  - tiss
aliases:
  - integracoes
origem: tissia/docs/integracoes.md
---

# Catálogo de integrações externas

Toda integração externa do backend vive no módulo `integracoes` (um submódulo por parceiro), exceto a Spike, que tem módulo próprio (`tissia-ai`, o SpikeAdapter da [[ADR-PEP-0005-spike-adapter|ADR-PEP-0005]]). Nenhum módulo de domínio chama HTTP de terceiro diretamente.

> **Convenção de nomenclatura.** *Spike* foi apenas o nome da PoC inicial e *MedScribe* o nome anterior do produto; o motor oficial é o **Tissia AI** (repositório `tissia-ai`). O contrato oficial com o Tissia AI usa **exclusivamente** chaves de API `tsk_test_` (sandbox, local e staging) e `tsk_live_` (produção) e headers `X-Tissia-*` (`X-Tissia-Cota-Restante`, `X-Tissia-Assinatura`, `X-Tissia-Evento`, `X-Tissia-Entrega`). Os prefixos e headers antigos (os que começavam com `msk` e com `X-Spike`) foram removidos do `tissia-ai` e não valem mais. No código e no banco deste repositório tudo usa o nome novo: módulo `tissia-ai`, variáveis `TISSIA_AI_*`, tabelas `analises_tissia_ai` e `tissia_ai_config_*`, evento `AnaliseTissiaAiConcluida`, origem de revisão `sugestao_tissia_ai` e consumidor `tissia-ai.feedback` (migração `0034`, que migra também as linhas existentes). Seguem com o nome da época só o que não se reescreve ou depende de fora do repositório: as ADRs aceitas (imutáveis), o nome do arquivo de migrações já aplicadas (`0004_analises_spike`), o segredo de CI `SPIKE_DEPLOY_KEY` (o workflow já aceita `TISSIA_AI_DEPLOY_KEY`, que precisa ser criado no GitHub) e os nomes de arquivo da documentação (`docs/spike/`, `*-spike-*`), a renomear junto com os links.

## 1. Catálogo

| Integração | Fase | Dono (módulo) | Protocolo | Autenticação | Dados trafegados | PII? | Falha | RFC / ADR |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Spike** (anti-glosa) | P1 (`analisar`, `feedback`); P2 (demais) | `tissia-ai` | REST JSON, `/v1` | API key `tsk_test_…` (sandbox, local, staging) ou `tsk_live_…` (produção), uma por ambiente | `paciente.ref` opaco, sexo, idade, evolução, códigos, valores | Não direta; evolução é dado sensível de saúde | Timeout 6 s; segue sem alertas; `degradado` → análise parcial; fila com retry nas rotas assíncronas | [[RFC-PEP-005-spike-adapter\|RFC-PEP-005]], [[mapeamento-pep-tissia-ia\|spike/README]] |
| **Spike — Scribe** (voz) | P2 | `tissia-ai` | REST + WebSocket | API key no backend; token de sessão curto no navegador | Áudio (sem retenção), transcrição | Voz e conteúdo clínico | Escuta cai → evolução por digitação continua | [[RFC-PEP-013-voz-ambiental\|RFC-PEP-013]], [[SPK-04-requisitos-do-scribe\|SPK-04]] |
| **Operadoras — webservices TISS** (elegibilidade, autorização, envio de lote, demonstrativo) | P1: só geração de XML e envio manual (portal da operadora); P2: webservices via parceiro | `integracoes/tiss` (gateway) | SOAP/XML TISS 4.03.00 (vigente; versão anterior só se a operadora exigir **[validar]**), via parceiro homologado | Certificado digital do prestador e/ou credenciais por operadora, guardados no parceiro ou no Secret Manager | Nome, carteirinha, CNS do beneficiário, procedimentos, valores | Sim | Elegibilidade sem resposta → agendamento segue com status "verificar"; envio de lote com retry e protocolo; falha visível na tela 18 | [[RFC-PEP-006-guias-e-lotes-tiss\|RFC-PEP-006]], [[ADR-PEP-0012-motor-tiss\|ADR-PEP-0012]] |
| **PSC ICP-Brasil** (assinatura em nuvem) | P2 | `integracoes/assinatura` | REST do PSC (padrão de assinatura remota) **[validar fornecedor: ex. BirdID, VIDaaS, SafeID]** | OAuth do PSC + autorização do titular (PIN/app) | Hash do documento (PAdES); não o documento inteiro, quando o PSC permitir | Hash não; identificação do signatário sim | Assinatura pendente não bloqueia salvar; documento fica "aguardando assinatura" | [[RFC-PEP-011-assinatura-icp-brasil\|RFC-PEP-011]], [[ADR-PEP-0011-assinatura-icp-brasil\|ADR-PEP-0011]] |
| **WhatsApp Business Platform** (lembrete, confirmação, resumo ao paciente) | P1 (lembrete e confirmação); P2 (resumo) | `integracoes/mensageria` | API oficial (Cloud API da Meta ou BSP) **[validar provedor]**, templates aprovados | Token do BSP no Secret Manager | Telefone, nome, data/hora; resumo da consulta (P2) | Sim | Falha → reenvio com backoff; status "sem resposta" na agenda (tela 03); opt-out respeitado | [[RFC-PEP-003-agenda\|RFC-PEP-003]], [[RFC-PEP-013-voz-ambiental\|RFC-PEP-013]] |
| **E-mail e SMS** (convites, recuperação de senha, alertas de sistema) | P0 | `integracoes/mensageria` | SMTP/API transacional **[validar provedor]** | Chave de API | E-mail/telefone do usuário | Sim (do usuário, não do paciente) | Fila com retry; recuperação de senha mostra "tente de novo" | [[RFC-PEP-001-identidade-e-acesso\|RFC-PEP-001]] |
| **Receita digital** (prescrição com validação em farmácia) | P2 | `integracoes/prescricao` | API de plataforma de receita digital **[validar: avaliar plataformas do mercado vs. PDF assinado próprio]** | OAuth/chave do parceiro | Paciente, prescritor, medicamentos | Sim | Falha → receita em PDF assinado (fallback) | [[RFC-PEP-004-prontuario\|RFC-PEP-004]], [[RFC-PEP-011-assinatura-icp-brasil\|RFC-PEP-011]] |
| **Laboratórios e imagem** (pedido e resultado) | P3 | `integracoes/diagnostico` | FHIR R4 (ServiceRequest/DiagnosticReport), HL7 v2 (ORM/ORU), DICOMweb para imagem **[validar por parceiro]** | mTLS ou OAuth por parceiro | Pedido com identificação do paciente; laudos | Sim | Resultado chega por fila; pendência visível na linha do tempo | [[RFC-PEP-016-expansao\|RFC-PEP-016]] |
| **Bancos** (conciliação, recebimento) | P2 | `integracoes/bancos` | Arquivos OFX/CNAB 240 **[validar layout por banco]**; PIX via API do PSP | Upload manual (OFX/CNAB) no início; API do PSP com certificado | Lançamentos financeiros, CPF/CNPJ de pagadores | Sim (particular) | Importação idempotente por hash do arquivo | [[RFC-PEP-010-financeiro-e-repasse\|RFC-PEP-010]] |
| **SSO de organizações** | P0 (federação disponível); uso sob demanda | Keycloak (fora do backend) | OIDC/SAML 2.0 | Federação configurada por organização | Identidade do usuário | Sim (do usuário) | Login por senha + MFA continua disponível para admins | [[RFC-PEP-001-identidade-e-acesso\|RFC-PEP-001]], [[ADR-PEP-0010-identidade-keycloak\|ADR-PEP-0010]] |
| **Armazenamento de anexos** | P1 | `anexos` | Object storage (GCS `southamerica-east1`), URL assinada de curta duração | Service account do backend | Documentos clínicos, laudos, imagens | Sim | Upload retomável; anexo sem confirmação de upload não entra no prontuário | [[RFC-PEP-004-prontuario\|RFC-PEP-004]], [[ADR-PEP-0009-hospedagem-gcp-sao-paulo\|ADR-PEP-0009]] |

## 2. Padrões comuns

| Padrão | Regra |
| :--- | :--- |
| **Isolamento** | Um submódulo por parceiro em `integracoes/`, com DTOs próprios. O domínio só conhece portas (interfaces) como `GatewayTiss`, `ServicoAssinatura`, `Mensageria` |
| **Outbox** | Toda chamada disparada por mudança de estado do domínio (ex.: `GuiaAssinada` → enviar lote; `AgendamentoCriado` → lembrete) é gravada na outbox na mesma transação e executada por worker BullMQ ([[ADR-PEP-0008-eventos-outbox-bullmq\|ADR-PEP-0008]]) |
| **Síncrono só quando o usuário espera** | Síncrono: `analisar` (6 s), elegibilidade no agendamento (5 s, depois "verificar"), autorização de assinatura. O resto vai para a fila |
| **Retry** | Backoff exponencial com jitter, máx. 8 tentativas em 24 h; depois, dead letter visível em Configurações › Integrações (tela 25) |
| **Idempotência** | Chave por operação (`lote_ref`, `guia_ref:revisao`, hash do arquivo). Reprocessar nunca duplica efeito |
| **Circuit breaker** | Por parceiro e operadora: 5 falhas seguidas abrem por 60 s; UI mostra o estado degradado (ex.: chip "Análise parcial" no topo, tela 14) |
| **Segredos** | Secret Manager, por ambiente; nunca no banco nem em variável de build da UI. Rotação documentada por parceiro |
| **Ambientes** | Cada parceiro tem sandbox/homologação ligada ao `staging`; `prod` só com credenciais de produção. Sem fallback de `prod` para sandbox |
| **Logs** | Só ids, códigos, status e latência. Nunca corpo de requisição com PII ou texto clínico (mesma regra da Spike ADR-0007) |
| **Observabilidade** | Métricas por parceiro: latência p95, taxa de erro, taxa de degradação, profundidade de fila, idade da mensagem mais antiga |
| **Minimização** | Mandar ao parceiro só o que ele precisa. Exemplo: o XML enviado à Spike para ingestão vai sem as tags de identificação direta ([[mapeamento-pep-tissia-ia\|spike/README §3.5]]) |
| **Contrato** | Para cada parceiro com API própria, teste de contrato contra o schema/OpenAPI publicado ou fixture gravada ([[test-plan-pep\|test plan]]) |
