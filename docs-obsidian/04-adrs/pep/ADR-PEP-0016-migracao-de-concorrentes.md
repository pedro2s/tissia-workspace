---
projeto: PEP
tipo: adr
status: proposta
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0016-migracao-de-concorrentes.md
---

# ADR-PEP-0016 — Migração de concorrentes por importadores de arquivo

**Status:** Proposta — 07/11/2026

## Contexto

Decisão em aberto nº 5 do design de produto: como trazer clientes de iClinic, Feegow, Amplimed, HiDoctor, MV, Tasy. Migração pesada é barreira de entrada (§10 do design de produto). Os concorrentes não oferecem API de exportação padronizada **[validar por concorrente]**, e prontuário legado não tem a estrutura (SOAP, versões, autoria IA) do nosso.

## Decisão

- **Importadores de arquivo** (CSV/XLSX/exports nativos de cada concorrente) na **P3**, no módulo `integracoes`, com pré-visualização, validação e relatório de erros antes de gravar. Sem integração direta com o banco ou a API do concorrente.
- **Ordem:** pacientes e beneficiários (carteirinhas) → agenda futura → contratos/tabelas de preço → prontuário legado.
- **Prontuário legado** entra como **documento anexo somente leitura** (PDF/HTML por atendimento) na linha do tempo, marcado com a origem; não é convertido em evolução estruturada nem versionado como registro nosso.
- Deduplicação de paciente por CPF + data de nascimento, com revisão humana em caso de dúvida.
- Cada importação é um job auditado (quem, quando, arquivo, contagens), reversível até o primeiro uso dos dados.
- A extensão Chrome (Spike F5) segue como porta de entrada antes da migração, sem depender dela.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Integração direta com a API/banco do concorrente | Frágil, possivelmente contrária aos termos de uso **[validar]** |
| Converter prontuário legado em evolução estruturada | Risco de alterar registro médico de terceiro; sem autoria verificável |
| Migração manual por serviço | Não escala; pode ser oferecida como serviço pago à parte |

## Consequências

- Escopo da P3 inclui um importador por concorrente-alvo, priorizado pela base de vendas.
- Prontuário legado não alimenta a Spike automaticamente (sem estrutura nem consentimento original).

**Para aceitar:** levantar com Thiago/vendas quais concorrentes predominam na base-alvo e que formatos de export cada um oferece; jurídico validar tratamento do prontuário legado (controladora = clínica).
