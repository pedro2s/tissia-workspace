---
projeto: PEP
tipo: rfc
status: rascunho
tags:
  - tissia
  - pep
  - rfc
  - especificacao
origem: tissia/docs/rfc/RFC-PEP-016-expansao.md
---

# RFC-PEP-016 — Expansão (P3)

**Fase:** P3 · **Status:** Rascunho · **ADRs:** ADR-PEP-0012, ADR-PEP-0016
**Telas:** sem tela no design atual (exceto laboratórios e API em 25)
**Requisitos:** RF-PLT-*, RF-BI-*, novos RF-INT-*, RF-TEL-*
**Depende de:** P1 e P2 concluídas; RFC-PEP-014 (API pública)

## Objetivo

Agrupar o que o roadmap põe na Fase 3. Cada item vira RFC própria quando a P2 terminar; aqui ficam escopo, dependências e o que precisa ser decidido antes.

## Itens

### 1. Internação

- Guias de solicitação e resumo de internação, honorários; prorrogação. A Spike já aceita `tipo_guia: "internacao"` em `POST /v1/guias/analisar`.
- Censo de leitos, evolução multiprofissional (enfermagem, prescrição hospitalar) **[validar escopo: hospital pequeno vs. day-hospital]**.
- Depende de: modelo de atendimento com episódio longo (estender RFC-PEP-004).

### 2. Telemedicina integrada

- Videochamada no atendimento, consentimento específico, registro na evolução **[validar Res. CFM vigente de telemedicina]**.
- Voz ambiental sobre o áudio da chamada (RFC-PEP-013) — reaproveita o Scribe.
- Decisão: provedor de vídeo (WebRTC gerenciado) com residência de dados no Brasil.

### 3. Laboratórios e imagem

- Pedido eletrônico e resultado na linha do tempo; padrões HL7 FHIR R4 (`ServiceRequest`, `DiagnosticReport`, `Observation`) e DICOM/DICOMweb para imagem **[validar adoção pelos parceiros]**.
- Tela 25 já prevê "Laboratórios — 2 parceiros".
- Marketplace de integrações (item 6).

### 4. BI avançado

- Exportação para armazém (BigQuery na mesma região), cubos por operadora/TUSS/médico, previsão de recebíveis, benchmarks anonimizados entre clínicas (exige base legal e anonimização real, não pseudonimização).

### 5. Certificação SBIS/CFM (NGS1/NGS2)

- Requisitos de segurança e de prontuário sem papel **[validar manual de certificação vigente]**: controle de acesso, auditoria, assinatura ICP (RFC-PEP-011), backup, integridade.
- Gap analysis no início da P3; muitos requisitos já vêm da P0/P1 (RLS, trilha encadeada, MFA).
- Meta comercial: vender para hospitais.

### 6. API pública e marketplace

- Detalhado como Rascunho na [[RFC-PEP-014-plataforma-lgpd-auditoria|RFC-PEP-014]] §P3: OAuth2 client credentials, escopos, webhooks assinados (`guia.validada`, `lote.protocolado`, `glosa.recebida`), sandbox, portal de desenvolvedor.
- Marketplace: catálogo de integrações (labs, imagem, farmácia, ERP/contábil, NFS-e) instaladas por clínica.

### 7. Migração de concorrentes

- Importadores por arquivo (ADR-PEP-0016) para iClinic, Feegow, Amplimed, HiDoctor: pacientes, agenda futura, histórico clínico como documento legado somente leitura (não vira evolução editável).
- Validação e relatório de importação; deduplicação de pacientes; consentimento e base legal da migração (clínica controladora).
- Extensão Chrome da Spike (Spike RFC-010) como porta de entrada que leva ao PEP.

## Riscos

- Escopo hospitalar (internação) multiplica requisitos regulatórios e de UX; pode merecer produto separado.
- Formatos de exportação dos concorrentes mudam sem aviso; contratos de saída de dados variam.
- Certificação SBIS tem custo e prazo longos; começar o gap analysis cedo.

## Perguntas abertas

| # | Pergunta | Dono |
| :--- | :--- | :--- |
| Q1 | Internação entra no PEP ou é produto à parte? | Sócios |
| Q2 | Quais laboratórios/redes de imagem atendem as clínicas piloto e que padrão expõem? | Produto |
| Q3 | NGS1 ou NGS2 como meta inicial? | Sócios **[validar]** |
| Q4 | Quais concorrentes migrar primeiro (base do Thiago, mercado-alvo)? | Comercial |

## Decisões a tomar

- Ordem dos itens na P3, por receita esperada.
- Provedor de vídeo e armazém analítico (novas ADRs).
