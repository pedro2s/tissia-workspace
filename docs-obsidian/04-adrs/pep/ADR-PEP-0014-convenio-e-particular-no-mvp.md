---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0014-convenio-e-particular-no-mvp.md
---

# ADR-PEP-0014 — MVP com convênio completo e particular simples

**Status:** Aceita — 13/11/2026 (proposta de 07/11/2026)

## Contexto

Decisão em aberto nº 2 do design de produto: particular e convênio juntos desde o MVP? O particular simplifica a entrada de clientes (não depende de TISS), mas é no convênio que a Spike gera valor (glosa). Clínicas-alvo do piloto atendem os dois (a agenda do design já mostra "Particular · Sem convênio").

## Decisão

- **Convênio completo na P1:** carteirinha, elegibilidade (manual na P1), guia de consulta e SP/SADT, Spike, lote XML 4.03 ([[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]]).
- **Particular simples na P1:** atendimento marcado como particular, valor da tabela da clínica, registro de recebimento (forma de pagamento, valor) e recibo em PDF. Sem conta a receber parcelada, sem NFS-e, sem maquininha.
- Financeiro completo (contas a pagar/receber, conciliação, repasse) e NFS-e na P2 ([[RFC-PEP-010-financeiro-e-repasse|RFC-PEP-010]]).
- A Spike não é chamada para atendimento particular (não há guia nem glosa).
- **Nota fiscal fora do MVP, por decisão:** os pacientes particulares costumam exigir nota fiscal (declaração de imposto de renda), mas a integração de NFS-e é complexa e a clínica já tem outros meios de emiti-la. O PEP entrega só o recibo em PDF; NFS-e pode voltar como item da RFC-PEP-010 depois do piloto.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Só convênio no MVP | A clínica-piloto continuaria com outro sistema para particular e não migraria a agenda |
| Particular completo + convênio | Financeiro completo dobra o escopo da P1 sem ajudar a provar a tese da Spike |
| Só particular no MVP | Não prova o diferencial (glosa evitada) |

## Consequências

- O recebimento particular da P1 precisa migrar para o modelo financeiro da P2 sem perda (mesma tabela de lançamentos, campos adicionais depois).
- Relatório do piloto separa receita convênio e particular.

**Aceite (13/11/2026):** aceita pelo responsável do produto sem confirmação com as clínicas-piloto: recibo simples basta na P1 e NFS-e fica fora do MVP. A mensagem de venda deve dizer isso com clareza ("o PEP gera recibo; a nota fiscal continua no seu emissor").
