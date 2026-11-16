---
projeto: tissia-ia
tipo: prd
status: em-refinamento
tags:
  - tissia
  - tissia-ia
  - prd
  - produto
aliases:
  - PRD-001-tissia-ia
origem: tissia-ai/docs/prd/PRD-001-tissia-ia.md
---

# PRD-001 — Tissia IA

**Versão:** 3.0 (substitui o PRD v2.0 em `.docs/` como referência de implementação)
**Data:** 29/10/2026
**Autor:** Pedro Santana
**Status:** Aprovado para F0/F1; F2+ em refinamento

## 1. O que mudou em relação ao PRD v2.0

A reunião com Thiago Polydoro (21/10/2026) trouxe fatos que reordenam o produto:

1. **O PEP do Thiago já tem um assistente de IA** que grava a consulta, transcreve, separa os locutores e resume em campos configuráveis depois que a consulta termina (sem streaming). O Scribe não é a lacuna mais urgente dessa base.
2. **Quem fatura é a faturista/secretária, não o médico.** O alerta anti-glosa precisa chegar a quem monta a guia, no momento em que ela é montada, e não necessariamente durante a consulta.
3. **O maior valor percebido é o anti-glosa:** glosas, guias esquecidas e procedimentos acessórios não lançados (ex.: Doppler junto ao ultrassom). "Se recuperar uma glosa por mês, já paga o sistema."
4. **Validação com clientes-cobaia sem custo** antes de vender, que é o processo que o Thiago já usa.
5. **Há dados históricos** (lotes de guias pagas/glosadas, PDFs de contratos com operadoras) que permitem uma análise preditiva depois.
6. **A parceria ainda depende do aval dos sócios do Thiago.** O produto precisa ter um contrato de API próprio e independente do código dele.
7. O Thiago sugeriu avaliar o **Jev** (TypeSafe AI) para a análise. Ver [[ADR-IA-0004-estrategia-de-modelos|ADR-0004]].

Consequência: **o anti-glosa vira a Fase 1** (regras, DUT, justificativa e sugestão TUSS/CID) e o Scribe vai para a F4 ([[ADR-IA-0001-anti-glosa-antes-do-scribe|ADR-0001]]).

## 2. Problema

- **Perda financeira:** clínicas perdem 5–12% do faturamento bruto com glosas (fonte: PRD v2.0; **[validar]** com dados reais da base do Thiago no baseline da F2). As causas são código TUSS errado, falta de justificativa clínica na guia, divergência entre evolução e procedimento solicitado, não conformidade com a DUT e acessórios esquecidos.
- **Carga do faturamento:** a auditoria manual das guias, antes do envio, consome a equipe de faturamento.
- **Burnout de documentação do médico** (35–50% do tempo da consulta no PEP): é real, mas fica para a F4.

## 3. Personas e usuários por fase

| Persona | Fase | O que ganha |
| :--- | :--- | :--- |
| **Faturista / secretária** (usa o PEP do Thiago) | F1–F3 | Alertas antes de enviar a guia, justificativa SP/SADT pronta e acessórios lembrados. |
| **Gestor financeiro da clínica** | F2–F3 | Redução de glosa mensurável e relatório de risco por operadora. |
| **Integrador — software house do Thiago** | F1+ | Módulo premium para upsell (+R$ 397–697/médico/mês) sem refatorar o legado. |
| **Médico especialista** | F4–F5 | Evolução estruturada automática; extensão para PEPs de terceiros. |

## 4. Objetivos e métricas

| Objetivo | Métrica | Meta | Medido em |
| :--- | :--- | :--- | :--- |
| Reduzir glosa técnica/administrativa nas clínicas-piloto | Taxa de glosa (R$ glosado / R$ faturado) vs. baseline de 3 meses | −30% no piloto (meta comercial do PRD v2: −60 a −80%, a provar) | F2 |
| Alertas úteis, sem fadiga | Precisão dos alertas (aceitos / exibidos) | ≥ 70% no piloto; ≥ 80% no golden set | F1 (eval) / F2 (produção) |
| Pegar o que importa | Recall de critério DUT ausente no golden set | ≥ 85% | F1 |
| Justificativa utilizável | Justificativas aceitas sem edição ou com edição leve | ≥ 80% | F2 |
| Sugestão de código correta | Acurácia top-3 no golden set (código esperado entre os 3 primeiros) | TUSS ≥ 80%; CID (subcategoria) ≥ 80% **[validar após baseline]** | F1 |
| Sem código inventado | Sugestões com código inexistente na tabela vigente | 0 (garantido por construção) | F1 |
| Não atrapalhar o fluxo | Latência p95 de `POST /v1/guias/analisar` | ≤ 1,5 s só com regras + DUT; ≤ 5 s com justificativa e/ou sugestão de códigos | F1 |
| Viabilidade econômica | Custo de IA por guia analisada | ≤ R$ 0,05 **[validar após eval]** | F1 |

## 5. Escopo por fase

### F0 — Fundações

- Repositório, CI, ambientes `staging` e `prod` no GCP São Paulo.
- Autenticação por API key por integrador; cadastro de clínicas com feature flags.
- Minimização/scrubbing de PII.
- Bases versionadas: TUSS (tabela 22), CID-10, correlação TUSS×Rol, DUTs das especialidades-piloto.
- Harness de avaliação de modelos (Jev × Gemini × modelos via OpenRouter, este último só com dados sintéticos).

### F1 — Motor Anti-Glosa MVP

- **[RF-AG-01] Análise síncrona de guia:** `POST /v1/guias/analisar` recebe a evolução clínica, os CIDs, os procedimentos TUSS, a operadora e o tipo de guia (Consulta e SP/SADT). Devolve alertas, conformidade DUT por procedimento e justificativa sugerida.
- **[RF-AG-02] Regras determinísticas:** campos obrigatórios da guia, incompatibilidade CID × sexo, código TUSS inexistente ou vencido, procedimento acessório ausente, regra específica de operadora (quando cadastrada).
- **[RF-AG-03] Validador DUT:** para procedimentos com DUT, verifica na evolução cada critério da diretriz e classifica como `conforme`, `nao_conforme` ou `incerto`, citando o critério e a fonte (nº da DUT).
- **[RF-AG-04] Justificativa SP/SADT:** redige o campo "indicação clínica" usando **apenas** fatos presentes na evolução, dentro do limite de caracteres do campo TISS. Se faltar critério DUT, a justificativa não o inventa: o alerta pede ao humano que confirme.
- **[RF-AG-05] Feedback:** `POST /v1/analises/{id}/feedback` registra se cada alerta, justificativa e sugestão de código foi aceito, rejeitado ou editado.
- **[RF-AG-07] Sugestão TUSS/CID** (RF06 do PRD v2): a partir da evolução, sugere os CIDs e os procedimentos TUSS, com confiança e trecho de evidência. Sinaliza procedimento citado na evolução mas não lançado na guia, e CID informado divergente do sugerido. Funciona também com a guia vazia, para quando a faturista ainda não lançou nada. Só sugere código que existe na tabela vigente ([[RFC-IA-006-sugestao-tuss-cid|RFC-006]]).
- **[RF-AG-06] Degradação graciosa:** se a IA falhar ou estourar o timeout, a resposta traz os alertas determinísticos e `degradado: true`; nunca um erro 5xx por causa da IA.
- **Sem UI própria:** a UI é do PEP do Thiago. Nós entregamos OpenAPI e exemplos ([[ADR-IA-0005-integracao-rest-sincrona|ADR-0005]]).

### F2 — Piloto fechado (rascunho: [[RFC-IA-007-piloto-e-desfecho|RFC-007]])

- 3–5 clínicas da base do Thiago, sem custo, mais o consultório do CMO se ele usar um PEP compatível.
- Modo sombra → modo visível ([[release-plan-ia|release plan]]).
- **[RF-AG-08] Desfecho:** `POST /v1/analises/{id}/desfecho` com pago/glosado e motivo (tabela TUSS de mensagens).
- Relatório de piloto: glosa antes/depois, precisão dos alertas, acurácia das sugestões de código, tempo da faturista.

### F3 — Preditivo (rascunho: [[RFC-IA-008-preditivo-historico-e-contratos|RFC-008]])

- Ingestão de lotes XML TISS históricos com o retorno da operadora; score de risco de glosa por guia/operadora.
- Leitura de contratos (PDF) com operadoras para comparar com as regras cadastradas no PEP.

### F4 — Scribe (rascunho: [[RFC-IA-009-scribe|RFC-009]])

- RF01–RF05 do PRD v2: captura, diarização, dicionário médico BR, SOAP e documentos.
- Primeiro: consumir a transcrição que o PEP do Thiago já gera. Depois: streaming próprio.

### F5 — Extensão standalone (rascunho: [[RFC-IA-010-extensao-chrome|RFC-010]])

- RF10 do PRD v2: extensão Chrome "Universal Autofill" para iClinic, Feegow, Amplimed, HiDoctor etc., como canal de aquisição.

### Fora do roadmap (backlog)

- App mobile nativo; assinatura ICP-Brasil; envio por WhatsApp; CID-11; receituário estruturado.

## 6. Requisitos não-funcionais

| # | Requisito |
| :--- | :--- |
| RNF-01 | **LGPD Art. 11 (dado sensível de saúde):** contrato de API sem identificadores diretos do paciente; scrubber de PII em texto livre antes de qualquer provedor de IA; nenhum dado de saúde em logs ([[ADR-IA-0007-minimizacao-de-dados\|ADR-0007]]). |
| RNF-02 | **Residência de dados:** processamento e armazenamento em `southamerica-east1`. Envio a provedor fora do Brasil (ex.: Jev) só com base legal de transferência internacional (cláusulas-padrão ANPD) e dados pseudonimizados ([[ADR-IA-0004-estrategia-de-modelos\|ADR-0004]]). |
| RNF-03 | **Human-in-the-loop:** o sistema só sugere; não envia guia, não assina, não altera dado no PEP sem ação humana ([[ADR-IA-0008-human-in-the-loop\|ADR-0008]]). |
| RNF-04 | **Rastreabilidade:** toda análise registra a versão das bases (TUSS/DUT/regras), o modelo e o prompt usados. |
| RNF-05 | **Disponibilidade:** 99,5% mensal na F1/F2 (Cloud Run, uma região). A API do integrador deve tolerar indisponibilidade da nossa API. |
| RNF-06 | **Latência:** ver metas da §4. |
| RNF-07 | **Auditabilidade clínica:** todo alerta DUT cita o critério e o número da DUT; toda justificativa é marcada como gerada por IA. |

## 7. Contrato de API

Definido em [[RFC-IA-002-api-analise-de-guia|RFC-002]]. O payload do PRD v2 §4 (consulta completa com prontuário + TISS + documentos do paciente) passa a ser a referência da F4. Na F1 a entrada é a guia.

## 8. Modelo de negócio (inalterado, para contexto)

- Upsell na base do Thiago: +R$ 397–697/médico/mês, com revenue share a definir no contrato de parceria.
- Plano Pro standalone (F5): R$ 497/mês. Clínica Blindada: R$ 797/médico/mês.
- O preço da F2 é zero (piloto).

## 9. Dependências externas e riscos

| Risco | Impacto | Mitigação |
| :--- | :--- | :--- |
| Sócios do Thiago não aprovam a parceria | Sem canal de piloto | API independente; plano B = clínicas indicadas pelo CMO, com integração manual (Swagger/CSV). |
| Regras das operadoras não são públicas | Validador limitado à DUT ANS | Importar regras já cadastradas no sistema do Thiago e envolver o consultor de faturamento indicado por ele. |
| Jev em early access, sem documentação de privacidade/região | Bloqueia uso com dado real | Gemini como decisor alternativo, com a mesma interface ([[ADR-IA-0004-estrategia-de-modelos\|ADR-0004]]). |
| Fadiga de alertas | Faturista ignora o produto | Meta de precisão + limiar calibrado + severidade; o modo sombra mede antes de exibir. |
| Justificativa "inventa" achado clínico | Risco ético/jurídico (fraude) | Geração ancorada em fatos da evolução + checagem cruzada + revisão humana obrigatória. |
| Exemplo do PRD v2 (RM de joelho exigindo Lachman via DUT) | Pode não haver DUT para esse caso; a exigência pode ser da operadora | CMO valida cada DUT usada; separar a fonte `ANS_DUT` da fonte `OPERADORA` nos alertas. |

## 10. Perguntas abertas

| # | Pergunta | Dono | Bloqueia |
| :--- | :--- | :--- | :--- |
| Q1 | O PEP do Thiago envia o texto da evolução junto da guia? Em que momento a faturista monta a guia? | Thiago | RFC-002 (validação do contrato) |
| Q2 | Qual versão TISS o sistema dele usa e que operadoras predominam na base? | Thiago | RFC-001 |
| Q3 | Quais especialidades predominam nas clínicas-piloto? (default do PRD: Ortopedia e Cardiologia) | Thiago + CMO | Curadoria de DUT (RFC-001) |
| Q4 | As regras de faturamento por operadora já cadastradas no sistema dele são exportáveis? Em que formato? | Thiago | RFC-003 (fonte `OPERADORA`) |
| Q5 | Há histórico de glosas por guia (com motivo) para baseline e golden set? | Thiago | F2 baseline, F3 |
| Q6 | Papéis LGPD: clínica = controladora; software house = operadora; Tissia IA = suboperadora? | Jurídico | Uso de dado real (F2) |
| Q7 | Acesso ao early access do Jev e termos de tratamento de dados da TypeSafe | Pedro | ADR-0004 |
