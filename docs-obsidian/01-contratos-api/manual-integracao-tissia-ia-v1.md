---
projeto: tissia-ia
tipo: contrato-api
status: ativo
tags:
  - tissia
  - tissia-ia
  - contrato-api
  - api-v1
  - openapi
aliases:
  - integracao
  - manual-integracao-v1
origem: tissia-ai/docs/integracao.md
---

# Tissia IA — Manual de Integração da API (`v1`)

> **Público-alvo:** Equipes de engenharia de sistemas de Prontuário Eletrônico do Paciente (PEP), plataformas de gestão clínica/hospitalar e sistemas de faturamento no padrão TISS/ANS.
> **Versão da API:** `v1`

---

## 1. Visão Geral

O **Tissia IA** é uma API REST JSON de inteligência clínica e regulatória projetada para integração nativa a sistemas de Prontuário Eletrônico do Paciente (PEP) e plataformas de faturamento de saúde suplementar no Brasil.

Seu objetivo é atuar de forma preventiva antes do envio do faturamento às operadoras de saúde, reduzindo glosas administrativas e técnicas no padrão **TISS/ANS** e acelerando o ciclo de receita das clínicas por meio de seis capacidades integradas:

1. **Validação Regulatória e Operacional TISS/ANS:** verificação imediata de consistência de campos obrigatórios, compatibilidade de sexo e idade, regras de procedimentos acessórios/excludentes, quantidades máximas e regras específicas de cada operadora de saúde.
2. **Auditoria de Diretrizes de Utilização (DUT/ANS):** verificação automática dos critérios clínicos exigidos pela ANS para procedimentos de alta complexidade, destacando os trechos literais da evolução clínica que comprovam cada critério ou sinalizando pendências documentais.
3. **Elaboração de Justificativa Clínica SP/SADT (Campo 23 TISS):** redação estruturada e concisa (respeitando o limite de 500 caracteres do padrão TISS) que conecta o quadro clínico, tratamentos/exames prévios e a indicação de cada procedimento solicitado.
4. **Sugestão de Codificação TUSS e CID-10:** identificação de procedimentos ou diagnósticos descritos pelo profissional de saúde na evolução clínica que não foram lançados ou que divergem dos códigos preenchidos na guia.
5. **Análise Preditiva de Risco e Conciliação de Retornos:** cálculo da probabilidade de glosa (`score_risco` de `0.0` a `1.0`) com base no histórico de faturamento da clínica por operadora e código TUSS, alimentado por conciliação JSON ou ingestão direta de arquivos XML TISS.
6. **Auditoria Contratual e Prevenção de Receita Esquecida:** conferência entre valores negociados em contrato e tabelas cadastradas no sistema de gestão, além do cruzamento entre atendimentos realizados na agenda e guias efetivamente emitidas.

---

## 2. Arquitetura de Integração, Desempenho e Segurança

### 2.1 Fluxo Operacional de Integração

```
[Profissional de Saúde / Faturista no Sistema de Gestão]
       │  Aciona "Validar / Salvar Guia"
       ▼
  [Sistema Integrador] ── POST /v1/guias/analisar (Timeout HTTP: 6s) ──► [API Tissia IA]
                                                                               ├─► 1. Anonimização em memória de dados sensíveis
                                                                               ├─► 2. Validação de regras TISS, ANS, Operadora e Contrato
                                                                               ├─► 3. Cálculo preditivo de risco de glosa
                                                                               └─► 4. Análise clínica (Checklist DUT, Codificação TUSS/CID e Justificativa SP/SADT)
  [Interface do Usuário] ◄── 200 OK (Alertas + DUT + Justificativa + Score) ───┘
       │
       │  Usuário revisa e Aceita / Edita / Rejeita sugestões
       ▼
  [Sistema Integrador] ── POST /v1/analises/{analise_id}/feedback (204 No Content)
```

### 2.2 Tempo de Resposta e Continuidade Operacional

- **Gatilho de chamada:** recomenda-se acionar `POST /v1/guias/analisar` no momento em que o médico encerra o atendimento com solicitação de exames/procedimentos ou quando a pessoa faturista realiza a conferência da guia antes da geração do lote XML TISS.
- **Timeout HTTP recomendado no cliente:** **6 segundos**.
- **Continuidade operacional garantida (Modo Degradado):** o Tissia IA foi desenhado para nunca bloquear o fluxo de atendimento ou faturamento da clínica. Caso os motores de análise de texto clínico excedam o orçamento interno de tempo (4 segundos) ou enfrentem instabilidade momentânea, a API **não retorna erro 5xx**: ela responde `200 OK` com o indicador `"degradado": true`, entregando integralmente todas as validações de regras TISS/ANS, regras da operadora, regras contratuais e o score preditivo de risco.

### 2.3 Privacidade, Proteção de Dados (LGPD) e Validação Estrita de Schema

A API segue o princípio de **minimização estrita de dados pessoais** e utiliza validação estrita de contrato (`extra="forbid"` em todos os objetos JSON de entrada):

1. **Rejeição automática de campos não previstos (`422 Unprocessable Entity`):** qualquer campo adicional fora dos especificados nesta documentação — seja por erro de nomenclatura ou envio acidental de dados pessoais como `nome`, `cpf`, `cns`, `rg`, `telefone`, `email`, `endereco` ou `carteirinha` — é automaticamente rejeitado com código HTTP `422` (`type: "extra_forbidden"`).
2. **Dados permitidos do paciente:** envie exclusivamente `paciente.ref` (identificador interno opaco ou hash gerado pelo sistema integrador), `paciente.sexo` (`"M"`, `"F"` ou `"I"`) e `paciente.idade` (em anos completos).
3. **Sanitização automática em memória:** antes de qualquer processamento do campo `evolucao_clinica`, o serviço executa em memória a supressão automática de eventuais dados pessoais digitados no texto livre (CPFs, Cartão Nacional de Saúde, telefones, e-mails, números de conselho profissional e nomes próprios acompanhados de vocativos clínicos).
4. **Não retenção de prontuário bruto e opt-in contratual:** o texto bruto da evolução clínica nunca é armazenado em banco de dados ou arquivos de log. Por padrão (`guardar_entrada_para_avaliacao = false`), nenhuma entrada clínica é retida; apenas quando `guardar_entrada_para_avaliacao = true` estiver explicitamente habilitado por **opt-in contratual da clínica** (mediante `aceite_contratual_ref` em `PUT /v1/clinicas/{clinica_ref}`), o texto **já sanitizado pelo scrubber** pode ser retido para avaliação de qualidade por até **180 dias** ([[ADR-IA-0007-minimizacao-de-dados|ADR-0007]]).

### 2.4 Conformidade Regulatória (Resolução CFM nº 2.454/2026)

Em conformidade com as normas do Conselho Federal de Medicina para uso de inteligência artificial de apoio à decisão e documentação médica:

- Toda resposta de análise inclui o campo textual `aviso_legal`:
  > *"Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."*
- O Tissia IA atua estritamente como ferramenta de suporte: nenhuma justificativa clínica ou alteração de código deve ser transmitida à operadora sem confirmação explícita do profissional responsável na interface do sistema integrador.

---

## 3. Modalidades de Implantação em Clínicas

Para facilitar a adoção gradual e a aferição de resultados financeiros nas clínicas clientes, recomenda-se que o sistema integrador suporte duas modalidades de operação controladas por configuração (*feature flag*) por clínica:

| Modalidade | Comportamento no Sistema Integrador | Objetivo |
| :--- | :--- | :--- |
| **Operação Silenciosa (Baseline)** | O backend do sistema integrador envia as guias para `POST /v1/guias/analisar` e armazena o `analise_id` retornado, porém **não exibe os alertas nem as sugestões** na tela do usuário. | Medir a linha de base de glosas da clínica e aferir preventivamente quantas glosas seriam evitadas sem alterar a rotina da equipe. |
| **Operação Assistida (Ativa)** | O sistema integrador exibe em tempo real os alertas, o checklist de critérios da DUT, a sugestão de justificativa SP/SADT e as sugestões de códigos TUSS/CID-10, enviando as decisões do usuário para `POST /v1/analises/{analise_id}/feedback`. | Reduzir ativamente o índice de glosas na origem, acelerar o fechamento de guias e medir a taxa de autocorreção antes do envio à operadora. |

---

## 4. Autenticação e Padrões de Comunicação

### 4.1 Headers Obrigatórios e Opcionais

Todas as chamadas aos endpoints `/v1/*` devem trafegar sob HTTPS (limite máximo de corpo de requisição de **64 KB** nas rotas clínicas `POST /v1/guias/analisar` e `POST /v1/glosas/recurso`) e incluir a chave de API do integrador no cabeçalho `Authorization`:

```http
Authorization: Bearer tsk_live_<chave_do_integrador>
Content-Type: application/json
X-Request-Id: <uuid-opcional-para-rastreamento>
```

> **Convenção de nomenclatura:** O motor oficial chama-se **Tissia AI** (antigo codinome de PoC *Spike* e antigo nome *MedScribe*). As chaves de API utilizam exclusivamente os prefixos `tsk_test_` (sandbox) e `tsk_live_` (produção), e todos os headers HTTP proprietários utilizam o prefixo `X-Tissia-*`:
> - `X-Tissia-Tempos-Ms` (resposta de `POST /v1/guias/analisar`): latências por etapa em milissegundos no formato `regras=<n>,dut=<n>,sugestao=<n>,justificativa=<n>,total=<n>`.
> - `X-Tissia-Cota-Restante` (respostas autenticadas em `/v1/*`): saldo restante da cota mensal configurada para o integrador (quando aplicável).
> - `X-Tissia-Assinatura`, `X-Tissia-Evento`, `X-Tissia-Entrega` (entregas de webhook): assinatura HMAC-SHA256 (`t=<ts>,v1=<hex>`), tipo do evento e UUID da entrega.

### 4.2 Códigos de Status HTTP

| Código HTTP | Significado | Tratamento Recomendado no Integrador |
| :--- | :--- | :--- |
| `200 OK` | Requisição processada com sucesso. | Renderizar resposta normalmente (verificar flag booleana `degradado`). |
| `201 Created` | Novo recurso criado (ex.: emissão de chave de integrador via `POST /v1/integrador/chaves`). | Persistir ou exibir com segurança as credenciais retornadas uma única vez. |
| `202 Accepted` | Requisição assíncrona aceita e enfileirada (`Prefer: respond-async`). | Ler `Location` (`/v1/jobs/{job_id}`) e acompanhar por polling ou webhook. |
| `204 No Content` | Registro de feedback, desfecho ou exclusão lógica concluído com sucesso. | Seguir fluxo normal sem necessidade de leitura de corpo de resposta. |
| `401 Unauthorized` | Chave de API ausente, malformada ou inválida. | Verificar a configuração do cabeçalho `Authorization`. |
| `403 Forbidden` | Chave de API inativa, revogada ou sem escopo necessário (ex.: rota `admin`). | Contatar a equipe de suporte do Tissia IA para renovação ou ajuste de escopo da credencial. |
| `404 Not Found` | Recurso referenciado (`analise_id`, `estruturacao_id`, `clinica_ref`, `job_id`) não localizado para o integrador autenticado. | Verificar se o identificador existe e pertence à mesma credencial de integração. |
| `409 Conflict` | Conflito de idempotência (`Idempotency-Key` reutilizada com payload divergente). | Gerar nova chave de idempotência para payloads distintos ou reenviar exatamente o payload original. |
| `413 Content Too Large` | Corpo da requisição acima do limite de 64 KB. | Reduzir ou truncar textos anexos antes do envio. |
| `422 Unprocessable Entity` | Payload JSON inválido ou presença de campos extras não previstos no schema (`extra_forbidden`). | Enviar estritamente os campos listados nas tabelas de contrato abaixo. |
| `429 Too Many Requests` | Limite de taxa por segundo ou cota mensal do plano excedida. | Respeitar o cabeçalho `Retry-After` / `retry_after_s` e aplicar backoff exponencial. |

---

## 5. Referência de Endpoints (`v1`)

### 5.1 `POST /v1/guias/analisar` — Análise Completa de Guia TISS

Analisa uma guia de **Consulta** (`consulta`), **SP/SADT** (`sp_sadt`) ou **Internação** (`internacao`) antes do faturamento.

#### Rastreamento de Revisões da Mesma Guia (`guia_ref`)
Sempre envie o identificador interno da guia no campo `guia_ref`. Quando o usuário corrigir uma pendência apontada pela API e salvar a guia novamente, envie a nova chamada mantendo o mesmo `guia_ref`: o Tissia IA reconhece automaticamente a nova revisão da guia e vincula o histórico de evolução da qualidade documental até o envio final à operadora.

#### Estrutura Estrita do Request (`extra="forbid"`)

> **Atenção:** Todos os objetos abaixo rejeitam propriedades extras com `422 Unprocessable Entity`. Envie apenas as chaves documentadas na tabela.

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `clinica_ref` | `string` | Sim | Identificador opaco da clínica/prestador no sistema integrador (1 a 128 caracteres). |
| `guia_ref` | `string \| null` | Não | Identificador interno da guia no sistema integrador (até 128 caracteres). |
| `tipo_guia` | `string` | Sim | `"consulta"`, `"sp_sadt"` ou `"internacao"`. Em `"internacao"`, aplicam-se as regras TISS/Rol/DUT dos procedimentos, justificativa clínica quando habilitada e coerência CID-procedimento. |
| `operadora` | `object` | Sim | Objeto contendo exclusivamente `registro_ans` (`string` de 1 a 20 caracteres, ex.: `"005711"` ou `"326305"`). |
| `especialidade` | `string \| null` | Não | Nome da especialidade médica (até 64 caracteres, ex.: `"cardiologia"`, `"ortopedia"`). |
| `paciente` | `object` | Sim | Objeto contendo exclusivamente: `ref` (`string`, 1 a 128 chars), `sexo` (`"M"`, `"F"` ou `"I"`) e `idade` (`integer` de `0` a `130`). |
| `evolucao_clinica` | `string \| null` | Não | Texto da evolução clínica / anamnese registrada no prontuário (até 20.000 caracteres). |
| `cids` | `string[]` | Não | Lista de códigos CID-10 informados na guia (ex.: `["I209"]` ou `["M51.1"]`, padrão: `[]`). |
| `procedimentos` | `object[]` | Não | Lista de procedimentos da guia contendo exclusivamente: `codigo_tuss` (`string`, 1 a 16 chars) e `quantidade` (`integer` de `1` a `100`, padrão: `1`). |
| `indicacao_clinica` | `string \| null` | Não | Texto atual do Campo 23 TISS (Indicação/Justificativa Clínica) preenchido na guia (até 5.000 caracteres). |
| `opcoes` | `object` | Não | Objeto opcional contendo: `gerar_justificativa` (`boolean`, padrão `true`) e `sugerir_codigos` (`boolean`, padrão `true`). |

#### Exemplo de Request (`application/json`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "guia_ref": "guia-2026-009841",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "005711"
  },
  "especialidade": "cardiologia",
  "paciente": {
    "ref": "pac-hash-99281a",
    "sexo": "M",
    "idade": 58
  },
  "evolucao_clinica": "Paciente com dor torácica atípica há 3 semanas e probabilidade pré-teste intermediária para doença arterial coronariana. Realizou teste ergométrico prévio inconclusivo há 7 dias. Solicito angiotomografia coronariana e ecocardiograma transtorácico.",
  "cids": ["I209"],
  "procedimentos": [
    {
      "codigo_tuss": "41001230",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": null,
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

#### Exemplo de Response (`200 OK`)

```json
{
  "analise_id": "8f14e45f-ceea-467f-a8c9-3189f4e2109b",
  "versao_bases": "2026.04.1",
  "risco": "alto",
  "score_risco": 0.85,
  "degradado": false,
  "origem_ia": {
    "decisor": "llm",
    "extracao": "llm",
    "justificativa": "llm"
  },
  "alertas": [
    {
      "id": "TISS-SADT-INDICACAO",
      "tipo": "campo_obrigatorio",
      "severidade": "bloqueante",
      "procedimento_tuss": null,
      "mensagem": "Guia SP/SADT sem Indicação Clínica preenchida (Campo 23 TISS).",
      "fonte": {
        "tipo": "tiss",
        "referencia": "TISS-SADT-INDICACAO"
      },
      "confianca": 1.0
    },
    {
      "id": "OP-005711-AUT-PREVIA-ALTO-CUSTO",
      "tipo": "exige_autorizacao_previa",
      "severidade": "aviso",
      "procedimento_tuss": "41001230",
      "mensagem": "A operadora 005711 exige autorização prévia e laudo de exame prévio anexo para Angiotomografia Coronariana (41001230).",
      "fonte": {
        "tipo": "operadora",
        "referencia": "005711"
      },
      "confianca": 1.0
    }
  ],
  "procedimentos": [
    {
      "codigo_tuss": "41001230",
      "termo_tuss": "Angiotomografia coronariana",
      "dut": {
        "numero": 3,
        "conformidade": "conforme",
        "criterios": [
          {
            "id": "dut_3_c1",
            "texto": "Dor torácica aguda ou crônica com probabilidade pré-teste baixa a intermediária para DAC",
            "status": "presente",
            "probabilidade": 0.95,
            "evidencia": "Paciente com dor torácica atípica há 3 semanas e probabilidade pré-teste intermediária para doença arterial coronariana."
          },
          {
            "id": "dut_3_c2",
            "texto": "Teste ergométrico ou cintilografia prévia inconclusiva ou impossibilidade de realizar teste funcional",
            "status": "presente",
            "probabilidade": 0.95,
            "evidencia": "Realizou teste ergométrico prévio inconclusivo há 7 dias."
          }
        ]
      },
      "justificativa_sugerida": {
        "texto": "Quadro clínico: dor torácica atípica com probabilidade pré-teste intermediária para DAC. Exames prévios: teste ergométrico prévio inconclusivo. Indicação: solicita-se Angiotomografia coronariana (41001230) sob CID I209 conforme critérios da DUT ANS nº 3.",
        "procedimentos_cobertos": ["41001230"],
        "gerada_por_ia": true
      }
    }
  ],
  "justificativa_sugerida": {
    "texto": "Quadro clínico: dor torácica atípica com probabilidade pré-teste intermediária para DAC. Exames prévios: teste ergométrico prévio inconclusivo. Indicação: solicita-se Angiotomografia coronariana (41001230) sob CID I209 conforme critérios da DUT ANS nº 3.",
    "procedimentos_cobertos": ["41001230"],
    "gerada_por_ia": true
  },
  "sugestoes_codigos": {
    "cids": [
      {
        "codigo": "I209",
        "descricao": "Angina pectoris, não especificada",
        "confianca": 0.92,
        "evidencia": "dor torácica atípica há 3 semanas e probabilidade pré-teste intermediária para doença arterial coronariana",
        "situacao": "ja_informado",
        "alternativas": []
      }
    ],
    "procedimentos": [
      {
        "codigo_tuss": "40901106",
        "termo": "Ecocardiograma transtorácico",
        "confianca": 0.9,
        "evidencia": "Solicito angiotomografia coronariana e ecocardiograma transtorácico.",
        "situacao": "ausente_na_guia",
        "alternativas": []
      }
    ]
  },
  "aviso_legal": "Sugestões geradas por IA de apoio. Revisão humana obrigatória antes do envio à operadora."
}
```

---

### 5.2 `POST /v1/analises/{analise_id}/feedback` — Registro de Decisão do Usuário

Registra as decisões tomadas pelo médico ou faturista sobre os itens sugeridos na análise.

#### Estrutura Estrita do Request (`extra="forbid"`)

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `papel` | `string` | Sim | `"faturista"` ou `"medico"`. |
| `itens` | `object[]` | Sim | Lista (1 a 100 itens) contendo exclusivamente: `item_id` (`string`, 1 a 128 chars), `acao` (`"aceito"`, `"editado"` ou `"rejeitado"`) e `texto_final` (`string \| null`, até 5.000 chars, utilizado quando `acao == "editado"`). |

#### Convenção de Identificadores (`item_id`)
- **Alertas:** utilizar o próprio `id` do alerta retornado em `alertas[].id` (ex.: `"TISS-SADT-INDICACAO"`).
- **Justificativa Clínica SP/SADT:** utilizar `"<codigo_tuss>:justificativa"` (ex.: `"41001230:justificativa"`) ou `"guia:justificativa"`.
- **Sugestão de CID-10:** utilizar `"sug_cid:<codigo>"` (ex.: `"sug_cid:I209"`).
- **Sugestão de Procedimento TUSS:** utilizar `"sug_tuss:<codigo_tuss>"` (ex.: `"sug_tuss:40901106"`).

#### Exemplo de Request (`application/json`)

```json
{
  "papel": "faturista",
  "itens": [
    {
      "item_id": "TISS-SADT-INDICACAO",
      "acao": "aceito"
    },
    {
      "item_id": "41001230:justificativa",
      "acao": "editado",
      "texto_final": "Quadro clínico: dor torácica atípica há 3 semanas com probabilidade pré-teste intermediária para DAC. Exames prévios: teste ergométrico inconclusivo há 7 dias. Solicita-se Angiotomografia coronariana (41001230) sob CID I209."
    },
    {
      "item_id": "sug_tuss:40901106",
      "acao": "aceito"
    }
  ]
}
```

- **Resposta esperada:** `204 No Content`.

---

### 5.3 `POST /v1/desfechos` e `POST /v1/analises/{analise_id}/desfecho` — Conciliação de Retorno de Faturamento

Quando a clínica recebe o demonstrativo de análise de conta da operadora, o sistema integrador pode informar o desfecho real (`pago`, `glosado_parcial` ou `glosado_total`) individualmente ou em lote (até 1.000 itens por requisição).

#### 5.3.1 Em Lote (`POST /v1/desfechos`)

Cada item de `itens[]` deve informar `analise_id` **ou** o par `("clinica_ref", "guia_ref")` (que associa automaticamente o resultado à última revisão da guia analisada antes do envio).

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `lote_ref` | `string \| null` | Não | Identificador do lote ou demonstrativo de retorno (até 128 caracteres). |
| `itens[].analise_id` | `uuid \| null` | Condicional | UUID da análise retornada por `/v1/guias/analisar` (obrigatório se `clinica_ref` e `guia_ref` não forem informados). |
| `itens[].clinica_ref` | `string \| null` | Condicional | Identificador da clínica (obrigatório se `analise_id` não for informado). |
| `itens[].guia_ref` | `string \| null` | Condicional | Identificador da guia (obrigatório se `analise_id` não for informado). |
| `itens[].status` | `string` | Sim | `"pago"`, `"glosado_parcial"` ou `"glosado_total"`. |
| `itens[].valor_cobrado` | `number \| null` | Não | Valor total cobrado na guia (`>= 0`). |
| `itens[].valor_glosado` | `number \| null` | Não | Valor glosado pela operadora (`>= 0`). |
| `itens[].codigo_glosa_tuss` | `string \| null` | Não | Código de glosa da Tabela 38 TUSS (até 16 caracteres, ex.: `"1706"`). |
| `itens[].motivo_glosa_texto` | `string \| null` | Não | Descrição textual do motivo da glosa enviado pela operadora (até 2.000 caracteres). |

##### Exemplo de Request (`POST /v1/desfechos`)

```json
{
  "lote_ref": "demonstrativo-operadora-2026-11",
  "itens": [
    {
      "clinica_ref": "clinica-ortopedia-sp-01",
      "guia_ref": "guia-2026-009841",
      "status": "pago",
      "valor_cobrado": 680.00,
      "valor_glosado": 0.00
    },
    {
      "analise_id": "8f14e45f-ceea-467f-a8c9-3189f4e2109b",
      "status": "glosado_parcial",
      "valor_cobrado": 850.00,
      "valor_glosado": 320.00,
      "codigo_glosa_tuss": "1706",
      "motivo_glosa_texto": "Ausência de detalhamento de tratamento conservador prévio."
    }
  ]
}
```

##### Exemplo de Response (`200 OK`)

```json
{
  "processados": 2,
  "nao_encontrados": []
}
```

#### 5.3.2 Individual (`POST /v1/analises/{analise_id}/desfecho`)

```json
{
  "status": "pago",
  "valor_cobrado": 680.00,
  "valor_glosado": 0.00,
  "codigo_glosa_tuss": null,
  "motivo_glosa_texto": null,
  "lote_ref": "demonstrativo-operadora-2026-11"
}
```

##### Exemplo de Response (`200 OK`)

```json
{
  "desfecho_id": "c42d9b10-7a11-4c9a-9e81-112233445566",
  "analise_id": "8f14e45f-ceea-467f-a8c9-3189f4e2109b",
  "status": "pago"
}
```

---

### 5.4 `POST /v1/tiss/lotes/ingerir` — Ingestão Direta de Arquivos XML TISS

Para sistemas integradores que geram lotes XML TISS de envio (`loteGuiasWS`) e importam demonstrativos XML de retorno das operadoras (`demonstrativoRetornoWS`) nas versões `3.05.00`, `4.00.00`, `4.01.00`, `4.02.00` ou `4.03.00`, este endpoint extrai automaticamente o número do lote, a operadora (`registroANS`), as guias e os resultados de pagamento/glosa diretamente do XML (exigindo tag `<ans:versaoPadrao>` ou `<ans:Padrao>` explícita e sem extração de CID em guias `SP/SADT` ou `Consulta`):

#### Estrutura Estrita do Request (`extra="forbid"`)

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `clinica_ref` | `string` | Sim | Identificador opaco da clínica no sistema integrador (1 a 128 caracteres). |
| `xml_conteudo` | `string` | Sim | Conteúdo integral do arquivo XML TISS (10 a 5.000.000 caracteres no modo síncrono; até **20.000.000** caracteres no modo assíncrono com `Prefer: respond-async`). |

> **Modo assíncrono (`SPK-03`):** Este endpoint — assim como `POST /v1/desfechos`, `POST /v1/contratos/auditar` e `POST /v1/guias/esquecidas` — aceita `Prefer: respond-async` e `Idempotency-Key` (ver `§5.14`). No modo síncrono, XMLs acima de 5.000.000 caracteres retornam `422`.

#### Exemplo de Request (`application/json`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "xml_conteudo": "<?xml version=\"1.0\" encoding=\"UTF-8\"?><ans:mensagemTISS xmlns:ans=\"http://www.ans.gov.br/padroes/tiss/schemas\">...</ans:mensagemTISS>"
}
```

#### Exemplo de Response (`200 OK`)

```json
{
  "tipo_mensagem": "demonstrativo_retorno",
  "versao_tiss": "4.01.00",
  "lote_ref": "PROT-2026-991",
  "registro_ans": "005711",
  "itens_ingeridos": 68,
  "desfechos_vinculados": 39
}
```

---

### 5.5 `POST /v1/contratos/auditar` — Auditoria de Tabela Contratual e Autorização Prévia

Compara os valores acordados em contrato entre a clínica e a operadora contra os valores e parâmetros configurados no sistema integrador. Quando `"provisionar_regras_autorizacao": true` (padrão), a API armazena a tabela contratual da clínica e cria automaticamente regras ativas de exigência de autorização prévia para as próximas chamadas de `POST /v1/guias/analisar`.

#### Estrutura Estrita do Request (`extra="forbid"`)

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `clinica_ref` | `string` | Sim | Identificador da clínica (1 a 128 caracteres). |
| `registro_ans` | `string` | Sim | Código ANS da operadora (1 a 20 caracteres). |
| `itens_contrato` | `object[]` | Sim | Lista (1 a 2.000 itens) contendo: `codigo_tuss` (`string`), `valor_contratado` (`number \| null`), `exige_autorizacao_previa` (`boolean`, padrão `false`) e `observacoes` (`string \| null`). |
| `itens_pep` | `object[]` | Não | Lista (até 2.000 itens) contendo: `codigo_tuss` (`string`), `valor_configurado` (`number \| null`) e `exige_autorizacao_previa` (`boolean`, padrão `false`). |
| `provisionar_regras_autorizacao` | `boolean` | Não | Quando `true` (padrão), provisiona regras automáticas de alerta de autorização prévia para a clínica. |

#### Exemplo de Request (`application/json`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "registro_ans": "005711",
  "provisionar_regras_autorizacao": true,
  "itens_contrato": [
    {
      "codigo_tuss": "41001230",
      "valor_contratado": 680.00,
      "exige_autorizacao_previa": true,
      "observacoes": "Exige pedido médico detalhado e teste funcional prévio"
    }
  ],
  "itens_pep": [
    {
      "codigo_tuss": "41001230",
      "valor_configurado": 620.00,
      "exige_autorizacao_previa": false
    }
  ]
}
```

#### Exemplo de Response (`200 OK`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "registro_ans": "005711",
  "total_itens_auditados": 1,
  "regras_autorizacao_criadas": 1,
  "divergencias": [
    {
      "codigo_tuss": "41001230",
      "tipo": "valor_abaixo_do_contrato",
      "mensagem": "Valor configurado no PEP (R$ 620.00) está abaixo do valor contratado com a operadora (R$ 680.00).",
      "valor_contrato": "680.00",
      "valor_pep": "620.00"
    },
    {
      "codigo_tuss": "41001230",
      "tipo": "autorizacao_previa_nao_configurada",
      "mensagem": "Contrato exige autorização prévia para o TUSS 41001230 na operadora 005711, mas o PEP está parametrizado como não exigido.",
      "valor_contrato": "680.00",
      "valor_pep": "620.00"
    }
  ]
}
```

---

### 5.6 `POST /v1/guias/esquecidas` — Conciliação de Atendimentos Realizados vs. Guias Emitidas

Cruza a lista de atendimentos realizados na agenda do sistema integrador contra as guias efetivamente analisadas ou enviadas em lotes XML TISS, identificando atendimentos sem guia emitida ou não localizados no faturamento.

#### Estrutura Estrita do Request (`extra="forbid"`)

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `clinica_ref` | `string` | Sim | Identificador da clínica (1 a 128 caracteres). |
| `atendimentos` | `object[]` | Sim | Lista (1 a 2.000 itens) contendo: `atendimento_ref` (`string`), `paciente_ref` (`string`), `data_atendimento` (`YYYY-MM-DD`), `registro_ans` (`string`), `codigo_tuss` (`string`), `guia_ref` (`string \| null`) e `valor_estimado` (`number \| null`). |

#### Exemplo de Request (`application/json`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "atendimentos": [
    {
      "atendimento_ref": "ag-2026-8812",
      "paciente_ref": "pac-hash-99281a",
      "data_atendimento": "2026-11-03",
      "registro_ans": "005711",
      "codigo_tuss": "41001230",
      "guia_ref": "guia-2026-009841",
      "valor_estimado": 680.00
    },
    {
      "atendimento_ref": "ag-2026-8819",
      "paciente_ref": "pac-hash-77410b",
      "data_atendimento": "2026-11-03",
      "registro_ans": "005711",
      "codigo_tuss": "10101012",
      "guia_ref": null,
      "valor_estimado": 180.00
    }
  ]
}
```

#### Exemplo de Response (`200 OK`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "total_atendimentos_verificados": 2,
  "total_guias_esquecidas": 1,
  "valor_total_em_risco": "180.00",
  "guias_esquecidas": [
    {
      "atendimento_ref": "ag-2026-8819",
      "paciente_ref": "pac-hash-77410b",
      "data_atendimento": "2026-11-03",
      "registro_ans": "005711",
      "codigo_tuss": "10101012",
      "valor_estimado": "180.00",
      "motivo": "Atendimento realizado na agenda sem guia TISS vinculada no PEP."
    }
  ]
}
```

### 5.7 Catálogo de Terminologia TUSS e CID-10 (`GET /v1/catalogos/*` — RFC-011 / SPK-09 / SPK-15)

Permite ao PEP listar, buscar incrementalmente (autocomplete por prefixo de código ou termo normalizado sem acento/caixa e com expansão de siglas) e consultar detalhes e vigência de códigos TUSS e CID-10 a partir da mesma base vigente que alimenta o motor anti-glosa e o RAG.

- `GET /v1/catalogos/tuss?q=<texto|código>&limit=20&tabela=22&data_referencia=YYYY-MM-DD&incluir_nao_vigentes=false` → `{ "itens": [{ "codigo_tuss", "termo", "grupo", "tem_dut", "tabela", "vigencia_inicio", "vigencia_fim", "vigente_na_data", "unidade_medida_padrao" }], "versao_bases" }` (`limit` de `1` a `50`; `tabela` aceita `"18" | "19" | "20" | "22" | "23" | "36" | "38" | "50"`; `data_referencia` opcional em formato `YYYY-MM-DD`, padrão data atual em `America/Sao_Paulo`; `incluir_nao_vigentes` booleano, padrão `false`)
- `GET /v1/catalogos/tuss/{codigo}?tabela=22&data_referencia=YYYY-MM-DD&incluir_nao_vigentes=false` → `{ "codigo_tuss", "termo", "grupo", "tem_dut", "numero_dut", "titulo_dut", "criterios_dut": [{ "id", "texto" }], "tabela", "vigencia_inicio", "vigencia_fim", "vigente_na_data", "unidade_medida_padrao", "versao_bases" }`
- `GET /v1/catalogos/cid?q=<texto|código>&limit=20` → `{ "itens": [{ "codigo_cid", "termo" }], "versao_bases" }`
- `GET /v1/catalogos/cid/{codigo}` → `{ "codigo_cid", "termo", "restricao_sexo", "versao_bases" }`

### 5.8 Governança e Proveniência das Bases (`GET /v1/bases/versoes` — RFC-013 / ADR-PEP-0024)

Retorna a competência ANS, hashes SHA-256, licenças, contagem de registros em memória vs. Postgres e o sinalizador `alinhado_fonte_unica` (além de `versao_tiss_suportada_max: "04.03.00"`), utilizado pelo portão de release do PEP antes da promoção para produção.

- `GET /v1/bases/versoes` (`Authorization: Bearer tsk_live_...`, `Cache-Control: public, max-age=300`) → `{ "versao_bases", "versao_tiss_suportada_max", "alinhado_fonte_unica", "bases": { "<base>": { "competencia_ans", "versao_tiss", "publicado_em", "carregado_em", "sha256", "sha256_arquivo_local", "licenca", "total_registros_memoria", "total_registros_pg", "status" } } }`

### 5.9 Scribe: Transcrição Diarizada (`POST /v1/consultas/transcrever` — RFC-009 / SPK-04)

Transcreve e separa locutores (*diarização*) a partir de um trecho de áudio codificado em base64, com **zero retenção de áudio** (decodificado e descartado em memória).

- **Request (`extra="forbid"`):**

| Campo | Tipo | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| `clinica_ref` | `string` | Sim | Identificador da clínica (1 a 128 chars). |
| `atendimento_ref` | `string` | Sim | Identificador opaco do atendimento (1 a 128 chars). |
| `audio_base64` | `string` | Sim | Áudio codificado em Base64 (1 a `6.000.000` chars, ~4,5 MB por trecho). |
| `mime_type` | `string` | Sim | `"audio/webm"`, `"audio/ogg"`, `"audio/mp4"`, `"audio/mpeg"`, `"audio/wav"` ou `"audio/flac"`. |
| `inicio_s` | `number` | Não | Deslocamento inicial do trecho em segundos (`0` a `14400`, padrão `0.0`). |
| `contexto` | `object[]` | Não | Até 12 segmentos anteriores (`extra="forbid"`): `{ "locutor", "texto" (<= 2.000 chars) }`. |

- **Response (`200 OK`):** `{ "segmentos": [{ "inicio_s", "fim_s", "locutor", "texto", "clinico" }], "degradado": boolean, "aviso_legal": string }`. Base64 inválido retorna `422`. Sem IA disponível, retorna `200 OK` com `degradado: true` e `segmentos: []`.

### 5.10 Scribe: Estruturação Clínica (`POST /v1/consultas/estruturar` — RFC-009 / SPK-04)

Estrutura a transcrição diarizada em SOAP e nos campos do modelo, cada item com `origens` (índices/números dos segmentos). Campos no request (`extra="forbid"`): `clinica_ref`, `atendimento_ref`, `especialidade`, `paciente` (`{ ref, sexo, idade } | null`), `modelo` (`[{ id, rotulo }]`, até 40), `transcricao` (`[{ ordem, inicio_s, fim_s, locutor, texto, clinico }]`, 1–2.000), `resumo_paciente` (`boolean`), `documentos` (`("pedido_exame" | "receita" | "atestado")[]`, até 3), `analisar_guia` (`boolean`) e `operadora` (`{ registro_ans } | null`).

- `documentos: ["pedido_exame", "receita", "atestado"]` → `documentos[]` com `tipo`, `origens`, `gerada_por_ia` e, conforme o tipo, `procedimentos[]` (`codigo_tuss`, `termo`, `origens`) e `indicacao_clinica`, `itens[]` (`medicamento`, `posologia`, `origens`) ou `texto`. Documento sem origem não é gerado; receita só com medicamento dito; TUSS só da base vigente.
- `analisar_guia: true` + `operadora` + `paciente` → `analise` com a mesma resposta de `POST /v1/guias/analisar`, sobre o SOAP e os procedimentos do pedido. Sem esses dados, `analise` é `null`.

Sem IA: `200 OK` com `degradado: true`, `provedor: "local"` e apenas o `pedido_exame` heurístico quando aplicável.

### 5.11 Scribe: `estruturacao_id` e Feedback (`POST /v1/consultas/estruturacoes/{estruturacao_id}/feedback` — RFC-009 / SPK-04)

- `POST /v1/consultas/estruturar` devolve `estruturacao_id`. O servidor guarda só metadados (contagens, provedor, latência), nunca transcrição, SOAP, campos ou resumo. A clínica segue a mesma política de `analisar` (`exigir_clinica_cadastrada` → `404 clinica_nao_cadastrada`; desativada → `404 clinica_desativada`).
- Feedback: mesmo corpo de `/v1/analises/{id}/feedback` (`papel`: `"faturista" | "medico"`, `itens[]` com `item_id`, `acao`: `"aceito" | "editado" | "rejeitado"`, `texto_final`), resposta `204 No Content`. `item_id`: `"campo:<id>"`, `"soap:<S|O|A|P>"` ou `"soap:<S|O|A|P>:<índice do trecho>"`, `"doc:<n>"`, `"resumo_paciente"`; outro formato → `422`. ID de outro integrador → `404`. `texto_final` só é gravado (após o scrubber) se a clínica tiver `guardar_entrada_para_avaliacao`; senão é descartado.

### 5.11.1 Resumo para o Paciente via SOAP e Metadados de Governança IA (`POST /v1/consultas/resumo-paciente` — SPK-15)

- **Metadados de Governança IA (`metadados_ia`):** as respostas de `POST /v1/guias/analisar`, `POST /v1/consultas/estruturar`, `POST /v1/glosas/recurso` (`/v1/glosas/recurso/sugerir`) e `POST /v1/consultas/resumo-paciente` incluem de forma aditiva o objeto `metadados_ia`: `{ "modelo": string, "versao_modelo": string, "versao_prompt": string, "provedor": "vertex_ai" | "local" }`.
- **`POST /v1/consultas/resumo-paciente`:** gera sugestão de resumo em linguagem simples para o paciente a partir da evolução SOAP já estruturada (sem exigir Escuta Ambiental nem `operadora`, suportando atendimentos particulares e convênios).
  - Request (`extra="forbid"`): `clinica_ref` (`string`, 1–128 chars), `atendimento_ref` (`string`, 1–128 chars), `paciente` (`{ "ref", "sexo": "M"|"F"|"I", "idade": 0..130 } | null`), `especialidade` (`string | null`), `soap` (`{ "subjetivo"?, "objetivo"?, "avaliacao"?, "plano"? }`, ao menos 1 seção com ≥ 10 chars, até 10.000 chars por seção) e `opcoes` (`{ "nivel_leitura"?: "simples" | "padrao", "incluir_sinais_alerta"?: boolean, "limite_caracteres"?: 200..3000 }`).
  - Response (`200 OK`): `{ "resumo_id": uuid, "versao_bases": string, "degradado": boolean, "origem_ia": "llm" | "local", "metadados_ia": MetadadosIA, "texto": string, "orientacoes": string[], "sinais_alerta": string[], "aviso_legal": string }`. Todo texto do `soap` passa pelo scrubber `app/pii.py` antes da chamada ao LLM; falha ou indisponibilidade do provedor responde `200 OK` com `degradado: true` e `provedor: "local"`.

### 5.12 Clínicas, Flags, Política do Integrador e Indicadores (`SPK-01`)

Gerencia o cadastro lógico de clínicas/unidades, feature flags, política do integrador e leitura de indicadores agregados do piloto. Rotas de escrita e política exigem chave com escopo **`admin`** (chaves `analise` recebem `403`).

- **`PUT /v1/clinicas/{clinica_ref}`** *(escopo `admin`)* — Cria ou atualiza a clínica de forma idempotente (`200 OK`).
  - Request (`extra="forbid"`): `nome` (`string | null`, 1–200 chars), `flags` (`{ modo?: "sombra" | "visivel", gerar_justificativa?: boolean, sugerir_codigos?: boolean, decisor?: "gemini" | "jev", guardar_entrada_para_avaliacao?: boolean }` com merge parcial), `aceite_contratual_ref` (`string | null`, até 200 chars — **obrigatório** quando `guardar_entrada_para_avaliacao: true`, senão `422`), `baseline_taxa_glosa` (`number | null`, `0.0..1.0`).
  - Response (`200 OK`): `{ "clinica_ref", "nome", "ativa", "flags": { "modo", "gerar_justificativa", "sugerir_codigos", "decisor", "guardar_entrada_para_avaliacao" }, "aceite_contratual_ref", "atualizado_em" }`.
- **`GET /v1/clinicas/{clinica_ref}`** — Retorna a clínica e suas flags efetivas (`404 clinica_nao_encontrada` se ausente).
- **`GET /v1/clinicas?cursor=<str>&limite=100`** — Lista paginada (`limite` de `1` a `100`): `{ "itens": [ClinicaResponse...], "proximo_cursor": string | null }`.
- **`DELETE /v1/clinicas/{clinica_ref}`** *(escopo `admin`)* — Desativa logicamente (`ativa = false`); análises subsequentes passam a retornar `404`.
- **`GET /v1/clinicas/{clinica_ref}/indicadores?de=YYYY-MM-DD&ate=YYYY-MM-DD`** — Indicadores agregados no período inclusivo (`ate < de` → `422`): `{ "periodo": { "de", "ate" }, "analises", "degradadas", "por_risco": { "alto", "medio", "baixo" }, "feedback": { "aceito", "editado", "rejeitado" }, "desfechos": { "pago", "glosado_parcial", "glosado_total", "valor_glosado" }, "glosas_sinalizadas_previamente": { "quantidade", "valor" }, "versao_bases_atual" }`.
- **`PUT /v1/integrador/configuracao`** *(escopo `admin`)* — Atualiza a política padrão do integrador (`extra="forbid"`): `{ "exigir_clinica_cadastrada"?: boolean | null, "modo_padrao"?: "sombra" | "visivel" | null }` → retorna a política efetiva (`200 OK`).

### 5.13 Recurso de Glosa Assistido (`POST /v1/glosas/recurso*` — SPK-02)

Gera argumentos de recurso de glosa ancorados em evidências literais da evolução e anexos, e registra feedback e desfecho do recurso para calibrar `chance_reversao`.

- **`POST /v1/glosas/recurso`** *(limite de corpo: 64 KB)*:
  - Request (`extra="forbid"`): `clinica_ref` (`string`, 1–128 chars), `analise_id` (`uuid | null`), `guia_ref` (`string | null`, até 128 chars), `operadora` (`{ "registro_ans": string (1–20) }`), `itens_glosados` (1 a 50 itens; `extra="forbid"`: `{ "codigo_tuss" (1–32), "valor_glosado" (>= 0), "codigo_glosa_tuss" (até 32 \| null), "motivo_glosa_texto" (1–2.000) }`), `evolucao_clinica` (`string`, 1–20.000 chars), `documentos_anexos` (`string[]`, até 30 rótulos) e `limite_caracteres` (`integer`, `100..5000`, padrão `2000`).
  - Response (`200 OK`): `{ "recurso_id", "versao_bases", "degradado", "aviso_legal", "itens": [{ "codigo_tuss", "argumento": { "texto", "gerada_por_ia" } \| null, "evidencias": [{ "trecho", "origem": "evolucao" \| "anexo" }], "criterios_nao_documentados": [{ "texto", "orientacao" }], "chance_reversao": number \| null, "base_historica": { "recursos_similares", "aceitos" } }] }`. (`chance_reversao` é `null` quando `recursos_similares < 10`; `404 analise_nao_encontrada` se `analise_id` não pertencer ao integrador.)
- **`POST /v1/glosas/recurso/{recurso_id}/desfecho`** (`204 No Content`):
  - Request (`extra="forbid"`): `status` (`"revertido_total" | "revertido_parcial" | "mantido"`), `valor_recuperado` (`number >= 0`, padrão `0.00`), `codigo_tuss` (`string | null`), `codigo_glosa_tuss` (`string | null`).
- **`POST /v1/glosas/recurso/{recurso_id}/feedback`** (`204 No Content`):
  - Request (`extra="forbid"`): `item_id` (`string`, 1–128 chars, ex.: `"<codigo_tuss>:recurso"`), `acao` (`"aceito" | "editado" | "rejeitado"`), `papel` (`"faturista" | "medico"`), `texto_final` (`string | null`, até 5.000 chars; sanitizado via scrubber).

### 5.14 Processamento Assíncrono, Jobs e Webhooks (`SPK-03`)

Permite enfileirar chamadas pesadas sem segurar a conexão HTTP e receber notificações assinadas via webhook HTTPS.

- **Disparo assíncrono (`Prefer: respond-async` + `Idempotency-Key`):**
  - Suportado em `POST /v1/tiss/lotes/ingerir` (estendendo o limite de `xml_conteudo` de `5.000.000` para **`20.000.000` caracteres**), `POST /v1/contratos/auditar`, `POST /v1/guias/esquecidas` e `POST /v1/desfechos`.
  - Com `Prefer: respond-async`, responde `202 Accepted` com cabeçalho `Location: /v1/jobs/{job_id}` e corpo `{ "job_id": "<uuid>", "status": "na_fila" }`.
  - `Idempotency-Key` (1–200 chars): reenvios idênticos retornam o mesmo `job_id`; mesma chave com payload diferente retorna `409 Conflict` (`idempotency_key_conflito`). O payload enfileirado é descartado ao final do processamento (LGPD).
- **`GET /v1/jobs/{job_id}`** — Consulta estado e resultado (`404 job_nao_encontrado` se ausente): `{ "job_id", "tipo", "status": "na_fila" | "processando" | "concluido" | "falhou", "criado_em", "concluido_em", "resultado": object | null, "erro": { "codigo", "mensagem" } | null }`.
- **`PUT /v1/integrador/webhook`** *(escopo `admin`)* — Cadastra ou rotaciona o endpoint HTTPS do integrador (com validação anti-SSRF contra `http://`, `localhost` e IPs privados).
  - Request (`extra="forbid"`): `{ "url": string (8–512 chars, HTTPS), "eventos": ("job.concluido" | "job.falhou")[] (min 1) }`.
  - Response (`200 OK`): `{ "url", "eventos", "segredo": "whsec_...", "ativo": true }` (`segredo` retornado para validação HMAC-SHA256 dos headers `X-Tissia-Assinatura: t=<ts>,v1=<hex>`, `X-Tissia-Evento` e `X-Tissia-Entrega`; o payload entregue contém `{ "evento", "job_id", "tipo" }`).
- **`POST /v1/integrador/webhook/teste`** *(escopo `admin`)* — Dispara um evento `"ping"` assinado para o webhook ativo (`404 webhook_nao_configurado` se inexistente) → `{ "sucesso": boolean, "status_http": integer | null, "erro": string | null }`.

### 5.15 Uso Mensal e Gestão de Chaves Standalone (`SPK-05`)

Rotas de autoatendimento e medição de cota para integradores (exigem chave com escopo **`admin`**).

- **`GET /v1/uso?mes=YYYY-MM`** (`mes` opcional, padrão mês UTC atual) — Retorna `{ "mes", "plano", "status", "cota_mensal", "total_requisicoes", "total_com_ia", "cota_restante", "por_clinica": { "<clinica_ref>": { "quantidade", "com_ia" } }, "por_rota": { "<rota>": { "quantidade", "com_ia" } } }`.
- **`GET /v1/integrador/chaves`** — Lista as chaves do integrador sem expor o segredo (`[{ "id", "prefixo", "escopos", "ambiente", "ativa", "expira_em", "ultimo_uso_em", "criado_em" }]`).
- **`POST /v1/integrador/chaves`** (`201 Created`) — Emite nova chave para rotação sem downtime:
  - Request: `{ "escopos": ("analise" | "admin")[] (padrão ["analise"]), "ambiente": "sandbox" | "prod" (padrão "prod"), "expira_em_dias"?: integer (1..3650) | null }`.
  - Response (`201 Created`): mesmos campos da listagem mais `"chave": "tsk_live_..." | "tsk_test_..."` exibida uma única vez.
- **`DELETE /v1/integrador/chaves/{chave_id}`** — Revoga logicamente a chave (`ativa = false`) → `{ "id", "ativa": false }` (`404` se não encontrada).

### 5.16 Autenticação e Saúde do Serviço

- **`GET /v1/auth/verificar`** *(autenticado)* — Valida a chave `Authorization: Bearer tsk_*` e retorna `{ "integrador_id", "nome", "ativo" }`.
- **`GET /healthz`** *(público, sem autenticação)* — Probe de disponibilidade e versão das bases → `{ "status": "ok", "versao_app", "versao_bases" }`.

---

## 6. Recomendações de Experiência do Usuário (UX/UI)

Para maximizar a adesão de médicos e faturistas sem adicionar atrito ao fluxo de trabalho (tempo médio de revisão recomendado de até 15 segundos por guia):

1. **Indicador de Risco Preditivo no Topo da Guia:**
   - Exibir um selo visual correspondente a `risco` (`baixo` em verde, `medio` em amarelo, `alto` em vermelho) acompanhado da probabilidade estimada em `score_risco` (`0.0` a `1.0`).
2. **Lista de Alertas Ordenada por Severidade:**
   - Priorizar a exibição dos itens de `alertas[]` na ordem `bloqueante` → `aviso` → `info`, destacando a fonte regulatória ou contratual (`fonte.tipo` e `fonte.referencia`).
3. **Checklist Visual de Critérios da DUT ANS:**
   - Nos procedimentos em que `dut.conformidade != "nao_aplicavel"`, apresentar cada item de `dut.criterios[]` com indicador visual de conformidade (`presente`, `ausente` ou `incerto`).
   - Quando `status == "presente"`, exibir o trecho literal extraído da evolução (`evidencia`) para dar segurança imediata à pessoa revisora.
   - Quando `status == "ausente"`, orientar o profissional a complementar a evolução clínica caso o paciente preencha clinicamente o requisito.
4. **Caixa Editável para Justificativa Clínica SP/SADT (Campo 23 TISS):**
   - Apresentar `justificativa_sugerida.texto` em um campo de texto editável (limitado a 500 caracteres) e botões de ação direta: **Aceitar e Preencher Guia**, **Editar** e **Ignorar**.
5. **Atalho de 1 Clique para Sugestões de Códigos TUSS e CID-10:**
   - Quando `sugestoes_codigos` apontar um código com `situacao == "ausente_na_guia"` ou `"diverge_do_informado"`, exibir a frase da evolução que originou a sugestão (`evidencia`) e permitir adicionar ou substituir o código na guia com 1 clique.
6. **Aviso Legal Visível:**
   - Exibir de forma fixa no rodapé do painel de revisão o texto retornado em `aviso_legal`.

---

## 7. Histórico de Revisões do Documento

| Versão | Data | Descrição das Alterações |
| :--- | :--- | :--- |
| `v1.0` | 04/11/2026 | Versão inicial com roteiro preliminar de integração e validação técnica. |
| `v1.1` | 04/11/2026 | Refinamento para especificação técnica oficial de integração da API `v1` e separação do roteiro de validação em documento independente. |
| `v1.2` | 04/11/2026 | Alinhamento estrito de todos os exemplos de Request e Response com os schemas Pydantic v2 (`extra="forbid"`) da API `v1`. |
| `v1.3` | 11/11/2026 | Adição das rotas de catálogo de terminologia TUSS e CID-10 (`/v1/catalogos/*` — RFC-011 / SPK-09). |
| `v1.4` | 14/11/2026 | Suporte a TISS `04.02.00`/`04.03.00` em `/v1/tiss/lotes/ingerir` e rota `GET /v1/bases/versoes` (`RFC-013` / `ADR-PEP-0024`). |
| `v1.5` | 15/11/2026 | Evolução do Scribe (`RFC-009` / `SPK-04`): geração de `documentos`, análise anti-glosa encadeada (`analise`), retorno de `estruturacao_id` e endpoint `POST /v1/consultas/estruturacoes/{id}/feedback`; inclusão de headers `X-Tissia-Tempos-Ms` e `X-Tissia-Cota-Restante` e códigos `201`, `202`, `409`, `429`. |
| `v1.6` | 16/11/2026 | Correção da numeração sequencial da seção 5 (`5.7`–`5.16`) e documentação completa de `SPK-01` (clínicas/indicadores/configuração), `SPK-02` (recurso de glosa), `SPK-03` (jobs, webhooks e limite de `20.000.000` chars em `/v1/tiss/lotes/ingerir` assíncrono), `SPK-04` (`POST /v1/consultas/transcrever`), `SPK-05` (`/v1/uso` e `/v1/integrador/chaves*`), `GET /v1/auth/verificar` e `GET /healthz`. |
| `v1.7` | 21/11/2026 | Implementação aditiva da `SPK-15`: objeto `metadados_ia` (`modelo`, `versao_modelo`, `versao_prompt`, `provedor`) nas respostas de IA, novo endpoint `POST /v1/consultas/resumo-paciente` (a partir do SOAP, sem exigir `operadora`) e extensão de `GET /v1/catalogos/tuss` / `/tuss/{codigo}` com `data_referencia`, `incluir_nao_vigentes` e campos `tabela`, `vigencia_inicio`, `vigencia_fim`, `vigente_na_data`, `unidade_medida_padrao`. |
