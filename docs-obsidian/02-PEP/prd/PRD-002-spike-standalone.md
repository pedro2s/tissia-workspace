---
projeto: PEP
tipo: prd
status: em-refinamento
tags:
  - tissia
  - pep
  - prd
  - produto
origem: tissia/docs/prd/PRD-002-spike-standalone.md
---

# PRD-002 — Spike como produto standalone

**Versão:** 1.0
**Data:** 07/11/2026
**Autor:** Pedro Santana
**Status:** Em refinamento (depende de SPK-01..05)

A Spike (repo `tissia-ai`, API `Tissia IA v1`) é o motor de IA anti-glosa usado pelo PEP Tissia. Este PRD trata a venda dela **à parte**, para quem não usa o PEP. Produto, contrato e fases internas estão na [[PRD-IA-001-tissia-ia|Spike PRD-001]]; aqui só o que muda para vendê-la.

## 1. Segmentos

| Segmento | Como consome | Prioridade |
| :--- | :--- | :--- |
| **PEP Tissia** (cliente nativo) | SpikeAdapter no backend do PEP ([[RFC-PEP-005-spike-adapter\|RFC-PEP-005]]) | 1 — primeiro cliente e vitrine |
| **PEPs e sistemas de faturamento de terceiros** | API REST `v1` integrada pelo fornecedor do sistema (integrador) | 2 — receita com pouco custo de atendimento |
| **Clínicas em PEPs de terceiros sem integração** | Extensão Chrome (Spike RFC-010, F5) | 3 — canal de aquisição; funil para o PEP |

## 2. Proposta de valor

- Para o **integrador**: módulo premium anti-glosa sem construir regras TISS/DUT, IA clínica nem bases versionadas. Contrato estável e documentado ([[manual-integracao-tissia-ia-v1|manual de integração]]).
- Para a **clínica**: menos glosa técnica e administrativa, justificativa do Campo 23 pronta, receita esquecida encontrada, prova de valor no modo silencioso antes de pagar.

## 3. O que já existe × o que falta

| Capacidade | Estado na Spike | Falta para vender |
| :--- | :--- | :--- |
| Análise de guia (regras, DUT, justificativa, TUSS/CID) | Concluída (F1) | — |
| Desfechos, relatório de piloto | Concluída (F2); relatório por script | Relatório acessível por API/portal ([[SPK-05-empacotamento-standalone\|SPK-05]]) |
| Preditivo, XML TISS, contratos, esquecidas | Concluída (F3) | Processamento assíncrono para arquivos grandes ([[SPK-03-processamento-assincrono\|SPK-03]]) |
| Cadastro de integrador e clínica | Scripts (`criar_integrador.py`, `configurar_clinica.py`); clínica autoprovisionada no 1º `analisar` com flags `{}` | API de clínicas e flags ([[SPK-01-api-de-clinicas-e-flags\|SPK-01]]) |
| Recurso de glosa | Não existe | [[SPK-02-recurso-de-glosa\|SPK-02]] |
| Scribe (estruturar consulta, resumo ao paciente) | Rascunho (Spike RFC-009) | [[SPK-04-requisitos-do-scribe\|SPK-04]] |
| Portal do integrador, medição de uso, limites, SLA | Não existe | [[SPK-05-empacotamento-standalone\|SPK-05]] |
| Extensão Chrome | Rascunho (Spike RFC-010) | Depende do Scribe e de auth de usuário final |

## 4. Regra de neutralidade

O PEP Tissia usa **só** o contrato público `v1`, igual a qualquer integrador:

- Sem endpoint privado, sem acesso ao banco da Spike, sem flag que só o PEP enxerga.
- Toda capacidade que o PEP precisar e o contrato não tiver vira proposta SPK e, quando aceita, entra no contrato público.
- O PEP tem uma API key de integrador como os demais.

Isso mantém a Spike vendável e evita que a evolução do PEP quebre clientes externos.

## 5. Modelo comercial

| Plano | Cobrança | Referência |
| :--- | :--- | :--- |
| Integrador (revenda) | Por médico ativo/mês ou por guia analisada, com revenue share | Upsell de R$ 397–697/médico/mês ao cliente final (Spike PRD-001 §8) **[validar]** |
| Por uso | Por guia analisada (faixas de volume) | Custo de IA alvo ≤ R$ 0,05/guia (Spike test plan §3) define o piso **[validar]** |
| Pro (extensão) | R$ 497/mês por médico | Spike PRD-001 §8 **[validar]** |
| Clínica Blindada | R$ 797/médico/mês | Spike PRD-001 §8 **[validar]** |

Piloto de 30 dias no modo silencioso sem custo; o relatório de glosa evitada é a peça de fechamento.

## 6. SLA e suporte

| Item | Meta |
| :--- | :--- |
| Disponibilidade mensal de `/v1/guias/analisar` | 99,5% (Spike RNF-05); 99,9% depois de multi-zona **[validar]** custo |
| Latência p95 | ≤ 1,5 s só regras + DUT; ≤ 5 s com justificativa e sugestão |
| Degradação | Nunca 5xx por IA; `degradado: true` |
| Mudança incompatível | `/v2` com no mínimo 90 dias de convivência (Spike ADR-0005) |
| Mudança de base (TUSS/DUT) | Nota de release com o diff resumido |
| Incidente com dado ou alerta errado | Aviso no mesmo dia; post-mortem |

## 7. Onboarding de um integrador

1. Contrato comercial e DPA (Spike como suboperadora).
2. API key de `staging` e acesso ao manual + OpenAPI.
3. Integração em homologação com dados fictícios; checklist do [[validacao-integracao-tissia-ia|roteiro de validação]].
4. Cadastro das clínicas e do modo (hoje por script; [[SPK-01-api-de-clinicas-e-flags|SPK-01]] torna self-service).
5. 30 dias silencioso → assistido; relatório de glosa evitada.

## 8. Métricas

| Métrica | Meta |
| :--- | :--- |
| Integradores ativos | Registrar; meta após o PEP estar em produção |
| Guias analisadas/mês | Registrar |
| Aceite de sugestões por integrador | ≥ 70% |
| Redução de glosa no piloto de cada clínica | −30% **[validar]** |
| Custo de IA por guia | ≤ R$ 0,05 **[validar]** |
| Churn de integradores | Registrar |

## 9. Riscos

| Risco | Impacto | Mitigação |
| :--- | :--- | :--- |
| Conflito de canal: vender a concorrentes do próprio PEP | Concorrente ganha o mesmo motor | O diferencial do PEP é a integração na origem (prontuário + guia + financeiro), não o motor; regra de neutralidade evita favorecer e evita perder clientes |
| Regras de operadora não públicas | Validação limitada à ANS | Regras customizadas por integrador (Spike RFC-007) e contratos (RFC-008) |
| LGPD: Spike como suboperadora de vários controladores | Exposição jurídica | Minimização no contrato (Spike ADR-0007); DPA padrão; RIPD por integrador **[validar]** |
| Inconsistência na doc: manual diz que a evolução nunca é armazenada, mas a Spike guarda o texto scrubbed com `guardar_entrada_para_avaliacao=true` (Spike ADR-0007) | Promessa contratual errada | Corrigir o manual antes de vender; ver [[mapeamento-pep-tissia-ia\|spike/README]] |
| Default do modo: release plan da Spike diz `sombra`, código usa `visivel` quando a flag falta (`app/api/guias.py:200`) | Cliente novo vê alertas antes do previsto | Decidir o default e alinhar código e doc ([[SPK-01-api-de-clinicas-e-flags\|SPK-01]]) |
| Dependência de IA de terceiros (Gemini, Jev) | Custo e disponibilidade | Degradação graciosa; decisor configurável (Spike ADR-0004) |

## 10. Perguntas abertas

| # | Pergunta | Dono |
| :--- | :--- | :--- |
| Q1 | Preço por guia ou por médico para integradores? | Comercial |
| Q2 | Quem faz o suporte de 1º nível ao cliente final de um integrador? | Comercial + integrador |
| Q3 | A extensão Chrome é da Spike ou do PEP? (hoje Spike RFC-010) | Pedro |
| Q4 | A Spike precisa de marca própria separada de "Tissia"? | Sócios |
