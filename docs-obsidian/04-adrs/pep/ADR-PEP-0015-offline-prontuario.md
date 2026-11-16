---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0015-offline-prontuario.md
---

# ADR-PEP-0015 — Offline do prontuário: PWA + fila local; conflito vira adendo

**Status:** Aceita — 07/11/2026

## Contexto

O design de produto pede modo offline para o prontuário (fila local com sincronização). Consultórios e tablets perdem conexão; o médico não pode perder a evolução nem parar o atendimento. Ao mesmo tempo, o prontuário é imutável ([[ADR-PEP-0007-prontuario-imutavel|ADR-PEP-0007]]) e o faturamento depende de dados consistentes no servidor.

## Decisão

- **PWA com service worker** ([[ADR-PEP-0004-frontend|ADR-PEP-0004]]): shell e telas do atendimento em cache.
- **Escopo offline:** só o atendimento já aberto no dispositivo: ler a linha do tempo baixada, escrever a evolução e rascunhos de documentos. Agenda, cadastro, faturamento, assinatura e Spike exigem conexão.
- **IndexedDB + fila de comandos** (`SalvarVersaoEvolucao`, `CriarRascunhoDocumento`), cada um com `comando_id` (UUID gerado no cliente) e `versao_base`. Reenvio idempotente por `comando_id`.
- **Servidor é a fonte de verdade.** Se a `versao_base` não for a atual (outro dispositivo gravou), o servidor **não sobrescreve**: grava o conteúdo offline como **adendo** marcado "registrado offline", com autor e horário do dispositivo e do servidor.
- Atendimento não fecha offline (fechar dispara a Spike e a guia); a UI mostra o estado "aguardando conexão para finalizar".
- Dados locais criptografados com chave derivada da sessão e apagados no logout e após X horas sem uso **[definir X na RFC-PEP-015]**.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Sem offline | Perda de evolução em queda de rede é inaceitável no consultório |
| Offline total com CRDT/merge | Merge automático de texto clínico não é seguro juridicamente |
| Faturamento offline | Guia depende de elegibilidade, contrato e Spike, todos online |

## Consequências

- Dado de saúde fica no dispositivo por um tempo: exige bloqueio de tela e limpeza; constar no RIPD ([[conformidade-regulatoria|conformidade]]).
- Testes de sincronização e conflito obrigatórios ([[test-plan-pep|test plan]]); detalhes na [[RFC-PEP-015-frontend|RFC-PEP-015]].
