---
projeto: ecossistema
tipo: spk
status: ativo
tags:
  - tissia
  - ecossistema
  - spk
  - proposta-contrato
  - spike
origem: tissia/docs/spike/SPK-01-api-de-clinicas-e-flags.md
---

# SPK-01 — API de clínicas, flags e indicadores

**Status:** Implementada na `tissia-ia` (`app/api/clinicas.py` + testes em `tests/test_spk01_clinicas_e_indicadores.py`: `PUT/GET/DELETE /v1/clinicas`, `/v1/clinicas/{ref}/indicadores` e `PUT /v1/integrador/configuracao`); falta integrar no `PEP` (onboarding de unidades em [[RFC-PEP-018-backoffice-clinicas-e-onboarding|RFC-PEP-018]] / módulo `tissia-ai`) · **Necessária para:** onboarding de unidade no PEP (P1), venda standalone ([[PRD-002-spike-standalone|PRD-002]])

## Objetivo

Permitir que o integrador cadastre clínicas, altere feature flags e consulte indicadores por API, sem depender de script executado pela equipe da Spike.

## Motivação

- Flags (`modo`, `gerar_justificativa`, `sugerir_codigos`, `decisor`, `guardar_entrada_para_avaliacao`) só mudam por `scripts/configurar_clinica.py` ([[release-plan-ia|Spike release-plan §4]]). Isso não escala além do piloto.
- `POST /v1/guias/analisar` autoprovisiona a clínica com `flags={}` quando o `clinica_ref` é novo (`app/api/guias.py:40-60`). Um `clinica_ref` digitado errado cria uma clínica fantasma e espalha análises.
- O default real de `modo` é `visivel` (`app/api/guias.py:200`), mas o release plan diz `sombra`.
- O relatório do piloto só existe como script (`scripts/relatorio_piloto.py`). Clientes standalone não têm como ver o valor entregue.

## Contrato proposto

Autenticação igual à atual (API key do integrador). Todas as rotas filtram pelo integrador da chave. `extra="forbid"` em todos os corpos.

### Clínicas

| Método e rota | Corpo / resposta | Erros |
| :--- | :--- | :--- |
| `PUT /v1/clinicas/{clinica_ref}` | Cria ou atualiza (idempotente) | 422 |
| `GET /v1/clinicas/{clinica_ref}` | Clínica com flags efetivas | 404 |
| `GET /v1/clinicas?cursor=…&limite=100` | Lista paginada | — |
| `DELETE /v1/clinicas/{clinica_ref}` | Desativa (soft delete); análises novas → 404 | 404 |

```json
PUT /v1/clinicas/un_7f3a…
{
  "nome": "Unidade Centro",
  "flags": {
    "modo": "visivel",
    "gerar_justificativa": true,
    "sugerir_codigos": true,
    "decisor": null,
    "guardar_entrada_para_avaliacao": false
  }
}
```

```json
200 OK
{
  "clinica_ref": "un_7f3a…",
  "nome": "Unidade Centro",
  "ativa": true,
  "flags": { "modo": "visivel", "gerar_justificativa": true, "sugerir_codigos": true, "decisor": "gemini", "guardar_entrada_para_avaliacao": false },
  "atualizado_em": "2026-11-07T12:00:00Z"
}
```

- `flags` é merge parcial: chave ausente mantém o valor atual. `decisor: null` = default global.
- `guardar_entrada_para_avaliacao: true` exige `"aceite_contratual_ref": "<id do termo>"` no mesmo corpo; sem ele, 422. Isso amarra o opt-in a um termo assinado (Spike ADR-0007).

### Política de integrador

`PUT /v1/integrador/configuracao` com `{"exigir_clinica_cadastrada": true, "modo_padrao": "visivel"}`.

- Com `exigir_clinica_cadastrada: true`, `analisar` com `clinica_ref` desconhecido retorna `404 {"detail": "clinica_nao_cadastrada"}` em vez de autoprovisionar.
- `modo_padrao` torna explícito o default hoje implícito. Default do campo para integradores existentes: `visivel` (comportamento atual do código).

### Indicadores

`GET /v1/clinicas/{clinica_ref}/indicadores?de=2026-10-01&ate=2026-10-31`

```json
{
  "periodo": { "de": "2026-10-01", "ate": "2026-10-31" },
  "analises": 1840,
  "degradadas": 22,
  "por_risco": { "alto": 210, "medio": 530, "baixo": 1100 },
  "feedback": { "aceito": 1210, "editado": 340, "rejeitado": 190 },
  "desfechos": { "pago": 1380, "glosado_parcial": 61, "glosado_total": 17, "valor_glosado": "18432.10" },
  "glosas_sinalizadas_previamente": { "quantidade": 49, "valor": "14120.00" },
  "versao_bases_atual": "2026.04.1"
}
```

`glosas_sinalizadas_previamente` = desfechos glosados cuja última análise antes do envio tinha alerta `bloqueante` ou `aviso` relacionado. É o número do relatório "glosas que teriam sido evitadas", hoje só no script.

## Impacto no modelo de dados da Spike

Migração aditiva:

- `clinicas.ativa boolean not null default true`, `clinicas.atualizado_em timestamptz`.
- `clinicas.aceite_contratual_ref text null`.
- `integradores.configuracao jsonb not null default '{}'`.
- Indicadores calculados por consulta sobre `analises`, `feedbacks` e `desfechos`. `# ponytail:` sem tabela materializada; criar uma visão materializada se a consulta passar de 2 s.

## LGPD e segurança

- Indicadores são agregados; nenhum `paciente_ref` ou texto clínico na resposta.
- Mudança de flag é auditada (quem, quando, valor anterior e novo) em tabela `clinicas_auditoria`.
- `guardar_entrada_para_avaliacao` só liga com referência ao termo; desligar apaga `entrada_scrubbed` futura, e a retenção de 180 dias da existente segue a ADR-0007.

## Compatibilidade

Só adições em `/v1`. Sem `exigir_clinica_cadastrada`, o autoprovisionamento continua como hoje. O script continua funcionando e passa a chamar a mesma camada de serviço.

## Critérios de aceite

- [ ] `PUT` idempotente; `GET` devolve flags efetivas (com defaults aplicados).
- [ ] Integrador A não lê nem altera clínica do integrador B (404).
- [ ] `exigir_clinica_cadastrada=true` → `analisar` com ref desconhecido = 404 e nenhuma linha nova em `clinicas`.
- [ ] Opt-in sem `aceite_contratual_ref` → 422.
- [ ] Indicadores batem com `scripts/relatorio_piloto.py` no mesmo período (teste com fixture).
- [ ] Snapshot OpenAPI atualizado e revisado.
- [ ] Release plan da Spike corrigido para o default real de `modo`.

## Perguntas abertas

1. O `decisor` por clínica deve ser exposto ao integrador ou ficar só para a equipe da Spike?
2. Indicadores por profissional (para o painel do gestor) exigem um `profissional_ref` no `analisar`. Vale adicionar como campo opcional?
3. Desativar clínica deve bloquear também `desfechos` de análises antigas?
