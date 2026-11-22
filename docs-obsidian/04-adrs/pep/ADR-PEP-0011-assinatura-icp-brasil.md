---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0011-assinatura-icp-brasil.md
---

# ADR-PEP-0011 — Assinatura ICP-Brasil via PSC em nuvem (PAdES)

**Status:** Aceita — 07/11/2026

## Contexto

Documentos clínicos (evolução final, receita, pedido de exame, atestado) precisam de assinatura com validade jurídica para dispensar o papel. A tela 07 mostra "assinatura ICP-Brasil só após decisão" sobre cada trecho da IA. O médico atende em consultório, tablet e celular: token A3 físico não funciona em todos esses lugares. Receita de controlados e antimicrobianos tem regras próprias de receituário digital **[validar: RDC ANVISA e Res. CFM vigentes]**.

## Decisão

- **Certificado em nuvem** emitido por **PSC credenciado pelo ITI** (ex.: BirdID, VIDaaS, SafeID, NeoID **[validar fornecedor e preço]**), integrado por API (OAuth do PSC + assinatura remota).
- **Formato:** PDF gerado pelo PEP e assinado em **PAdES** (AD-RB no mínimo, AD-RT com carimbo de tempo para documentos que precisam de prova temporal) **[validar política de assinatura]**. Evolução também pode ser assinada como CAdES sobre o JSON canônico da versão final.
- **Trilha de revisão anexada:** o PDF da evolução inclui (ou referencia por hash) a trilha: trechos de IA, decisão (aceito/editado/rejeitado), autor e horário ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]).
- **Bloqueio:** documento com sugestão de IA pendente de decisão não pode ser assinado (regra do design, tela 07).
- Evento `GuiaAssinada` / `DocumentoAssinado` após confirmação do PSC; falha do PSC não perde o documento (fica "aguardando assinatura").
- Verificação de assinatura disponível para terceiros (link/QR no documento) **[validar exigência para receita]**.
- Implementação na P2 ([[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]]); na P1 o documento sai com assinatura simples (login + MFA) e marcação de que não é ICP.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Token/cartão A3 local (Web PKI/extensão) | Não funciona em tablet/celular; exige instalação |
| Assinatura eletrônica avançada (sem ICP) | Não atende receita e documentos que exigem ICP **[validar]** |
| Plataforma de prescrição de terceiros (ex.: Memed) para receita | Pode complementar; avaliar como integração em [[integracoes-externas\|integracoes]] |

## Consequências

- Custo por assinatura/por médico do PSC entra no preço.
- Dependência externa no fechamento: o atendimento fecha mesmo com PSC fora; a assinatura fica pendente.

**Para aceitar:** jurídico/CMO validar requisitos (CFM, ANVISA para receita digital, prazo de guarda); escolher PSC após comparação de API, preço e SLA.
