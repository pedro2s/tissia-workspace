---
projeto: tissia-ia
tipo: contrato-api
status: ativo
tags:
  - tissia
  - tissia-ia
  - contrato-api
  - postman
  - testes-api
aliases:
  - ensaios_postman
origem: tissia-ai/docs/ensaios_postman.md
---

# Tissia IA — Caderno de Ensaios e Cenários Reais para Testes no Postman (`v1`)

> **Objetivo:** Fornecer uma coleção completa de cenários clínicos, regulatórios e financeiros próximos da rotina real de clínicas e operadoras para ensaiar e demonstrar todos os endpoints da API `v1` no **Postman** (ou Insomnia).

---

## 0. Configuração Inicial do Ambiente no Postman

1. Crie um **Environment** no Postman com as seguintes variáveis:
   - `base_url`: `http://localhost:8000` (ou a URL do ambiente em que a API estiver rodando)
   - `api_key`: `tsk_test_<sua_chave_de_api>` (ou `tsk_live_...` em produção)
   - `analise_id`: *(será preenchido com o UUID retornado pelo Cenário 01)*
2. Em todas as requisições, configure os **Headers**:
   ```http
   Authorization: Bearer {{api_key}}
   Content-Type: application/json
   X-Request-Id: ensaio-postman-001
   ```
3. Se precisar gerar uma nova chave de API local rapidamente no terminal do projeto:
   ```bash
   .venv/bin/python -m scripts.criar_integrador --nome "Ensaio Postman"
   ```

---

## Bloco A — Cenários Clínicos Reais de Análise de Guia (`POST /v1/guias/analisar`)

### Cenário 01 — Cardiologia: DUT Conforme + Falta de Indicação Clínica + Regra de Operadora + Exame Esquecido na Guia

- **Situação real simulada:** Paciente masculino, 58 anos, atendido pelo convênio `005711` com dor torácica atípica e teste ergométrico prévio inconclusivo. O cardiologista solicita **Angiotomografia Coronariana (`41001230`)** e escreve na evolução que também solicitou **Ecodopplercardiograma Transtorácico (`40901106`)**, mas a guia SP/SADT foi aberta **sem o Campo 23 (`indicacao_clinica: null`)** e **sem o código do Ecocardiograma**.
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "guia_ref": "guia-cardio-2026-001",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "005711"
  },
  "especialidade": "cardiologia",
  "paciente": {
    "ref": "pac-cardio-001",
    "sexo": "M",
    "idade": 58
  },
  "evolucao_clinica": "Paciente sintomático com dor torácica atípica há 4 semanas e probabilidade pré-teste intermediária de doença arterial coronariana (DAC). Traz ECG basal com alterações inespecíficas de repolarização e teste ergométrico realizado há 10 dias com resultado inconclusivo por fadiga precoce. Solicito angiotomografia coronariana para estratificação anatômica e ecodopplercardiograma transtorácico para avaliação da função ventricular.",
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

#### O que observar na resposta (`200 OK`):
1. **Alerta Bloqueante (`TISS-SADT-INDICACAO`):** aponta que a guia SP/SADT está sem Indicação Clínica preenchida.
2. **Alerta de Operadora (`OP-005711-AUTH-PREVIA`):** avisa que o código `41001230` exige senha de autorização prévia na operadora `005711`.
3. **Checklist da DUT nº 3 (`conformidade: "conforme"`):** comprova que o critério `c1` está `"presente"` citando literalmente o trecho da evolução sobre probabilidade pré-teste intermediária e teste ergométrico inconclusivo.
4. **Justificativa Sugerida (`justificativa_sugerida.texto`):** texto estruturado (<= 500 caracteres) pronto para preencher o Campo 23 TISS.
5. **Recuperação de Receita (`sugestoes_codigos.procedimentos`):** identifica o código `40901106` (*Ecodopplercardiograma transtorácico*) ou `procedimento_nao_lancado` descrito na evolução, mas esquecido na guia.
6. Copie o `analise_id` retornado para usar no **Cenário 08**.

---

### Cenário 02 — Ortopedia: DUT Não Conforme (Infiltração / Viscossuplementação de Joelho sem Fisioterapia Prévia)

- **Situação real simulada:** Paciente feminina, 64 anos, com gonartrose (`M170`) confirmada em radiografia de joelho, solicitando **Infiltração articular de joelho (`30713030`)**. A **DUT nº 65 da ANS** exige obrigatoriamente **dois grupos de critérios (`logica: "todos"`)**: (1) comprovação por imagem E (2) falha de tratamento conservador (analgésicos/AINEs e fisioterapia) por **pelo menos 3 meses**. Na evolução abaixo, a paciente iniciou medicação há apenas 10 dias e não realizou fisioterapia.
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "guia_ref": "guia-orto-2026-002",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "326305"
  },
  "especialidade": "ortopedia",
  "paciente": {
    "ref": "pac-orto-002",
    "sexo": "F",
    "idade": 64
  },
  "evolucao_clinica": "Paciente com dor mecânica em joelho direito há 2 semanas. Traz radiografia de joelho demonstrando osteoartrite (gonartrose) femorotibial medial Kellgren-Lawrence III. Iniciou dipirona há 10 dias, ainda sem realização de fisioterapia motora ou reabilitação. Solicito infiltração articular de joelho direito.",
  "cids": ["M170"],
  "procedimentos": [
    {
      "codigo_tuss": "30713030",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Gonartrose em joelho direito.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

#### O que observar na resposta (`200 OK`):
1. **DUT nº 65 (`conformidade: "nao_conforme"` ou `"incerto"`):** o critério `c1` (radiografia demonstrando gonartrose) aparece como `"presente"`, mas o critério `c2` (falha de tratamento conservador por >= 3 meses) não é atendido.
2. **Prevenção de Glosa Técnica:** emite alerta de DUT informando exatamente qual requisito documental/clínico falta antes de submeter o pedido à operadora.

---

### Cenário 03 — Ortopedia Cirúrgica: DUT 110 Conforme (Reconstrução Ligamentar de Joelho)

- **Situação real simulada:** Paciente masculino, 29 anos, sofreu trauma torcional de joelho com instabilidade objetiva ao exame físico (testes de Lachman, Gaveta Anterior e Pivot Shift positivos) e ressonância magnética confirmando ruptura completa do ligamento cruzado anterior (`S835`). Solicita-se **Reconstrução ligamentar intra-articular do joelho (`30724058`, DUT 110)**.
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-ortopedia-sp-01",
  "guia_ref": "guia-orto-2026-003",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "359017"
  },
  "especialidade": "ortopedia",
  "paciente": {
    "ref": "pac-orto-003",
    "sexo": "M",
    "idade": 29
  },
  "evolucao_clinica": "Paciente vítima de entorse de joelho direito durante partida de futebol há 4 semanas, cursando com falseios articulares recorrentes. Ao exame físico ortopédico: derrame articular +/4+, teste de Lachman positivo (3+), teste da Gaveta Anterior positivo e Pivot Shift positivo. Ressonância magnética do joelho direito confirma ruptura completa do ligamento cruzado anterior (LCA) associada a lesão meniscal medial com bloqueio articular mecânico. Indico reconstrução ligamentar intra-articular do joelho.",
  "cids": ["S835"],
  "procedimentos": [
    {
      "codigo_tuss": "30724058",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Ruptura completa de LCA com instabilidade articular objetiva ao exame físico.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

#### O que observar na resposta (`200 OK`):
1. **DUT nº 110 (`conformidade: "conforme"`):** ambos os critérios (`c1` — instabilidade ligamentar objetiva com Lachman/Gaveta/Pivot Shift e RM; `c2` — lesão meniscal com bloqueio mecânico) validados com evidência literal.
2. **Risco Baixo (`risco: "baixo"`):** guia pronta para envio sem alertas bloqueantes.

---

### Cenário 04 — Múltiplos Procedimentos com DUT na Mesma Guia (Holter 24h + MAPA 24h)

- **Situação real simulada:** Paciente feminina, 52 anos, com palpitações recorrentes, episódios de pré-síncope e suspeita de hipertensão do avental branco. Foram lançados dois exames com DUT na mesma guia SP/SADT: **Holter 24h (`20102011`, DUT 38)** e **MAPA 24h (`20102020`, DUT 54)**.
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "guia_ref": "guia-cardio-2026-004",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "326305"
  },
  "especialidade": "cardiologia",
  "paciente": {
    "ref": "pac-cardio-004",
    "sexo": "F",
    "idade": 52
  },
  "evolucao_clinica": "Paciente refere palpitações taquicárdicas recorrentes há 2 meses associadas a dois episódios de pré-síncope e tontura inexplicada. Adicionalmente, apresenta níveis pressóricos elevados exclusivamente em consultório (PA 158x96 mmHg) com medidas residenciais normais, configurando forte suspeita de hipertensão do avental branco. Solicito Holter de 24 horas para investigação de arritmia cardíaca e MAPA de 24 horas para confirmação diagnóstica pressórica.",
  "cids": ["R002", "I10"],
  "procedimentos": [
    {
      "codigo_tuss": "20102011",
      "quantidade": 1
    },
    {
      "codigo_tuss": "20102020",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Investigação de palpitações e hipertensão.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

#### O que observar na resposta (`200 OK`):
1. **Avaliação Individual por Procedimento:** `procedimentos[0]` traz a avaliação da **DUT nº 38** (Holter) e `procedimentos[1]` traz a avaliação da **DUT nº 54** (MAPA).
2. **Justificativa Consolidada (`justificativa_sugerida`):** um único texto coeso cobrindo os dois procedimentos (`"procedimentos_cobertos": ["20102011", "20102020"]`) respeitando o teto de 500 caracteres do Campo 23 TISS.

---

### Cenário 05 — Diagnóstico por Imagem: Recuperação de Receita por Código Acessório de Doppler Omitido

- **Situação real simulada:** O médico realizou uma **Ultrassonografia de Abdome Total (`40901203`)** e descreveu no laudo/evolução que complementou o exame com **Doppler colorido** (`40901760`). Porém, a recepção lançou apenas o código principal `40901203` na guia, deixando de faturar o procedimento complementar.
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-imagem-sp",
  "guia_ref": "guia-usg-2026-005",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "359017"
  },
  "especialidade": "radiologia",
  "paciente": {
    "ref": "pac-img-005",
    "sexo": "F",
    "idade": 41
  },
  "evolucao_clinica": "Paciente com dor em hipocôndrio direito e elevação de transaminases. Realizada ultrassonografia de abdome total complementada com estudo de Doppler colorido do sistema porta-hepático, demonstrando fluxo hepatopetal preservado e ausência de trombose portal.",
  "cids": ["R104"],
  "procedimentos": [
    {
      "codigo_tuss": "40901203",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Dor abdominal em hipocôndrio direito a esclarecer.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```

#### O que observar na resposta (`200 OK`):
1. **Alerta `ACESS-US-DOPPLER` (`tipo: "acessorio_ausente"`):** avisa que o exame com Doppler colorido foi documentado na evolução, mas o código acessório `40901760` não foi incluído na guia (evitando perda direta de receita).

---

### Cenário 06 — Bateria de Glosas Administrativas Bloqueantes (Incompatibilidade de Sexo, Excludência, Quantidade e Vigência TUSS)

- **Situação real simulada:** Guia preenchida com múltiplos erros formais que causariam glosa imediata no lote TISS:
  1. Paciente do sexo masculino (`"sexo": "M"`) com CID `C539` (*Neoplasia maligna do colo do útero*, restrito ao sexo feminino);
  2. Cobrança de Ultrassonografia Transvaginal (`40901300`, restrita ao sexo feminino) para paciente masculino;
  3. Cobrança de Consulta em Consultório (`10101012`) com `quantidade: 2` na mesma guia;
  4. Cobrança concomitante de Consulta em Consultório (`10101012`) e Consulta em Pronto-Socorro (`10101039`) na mesma guia;
  5. Código TUSS fora de vigência (`99990001`).
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-auditoria-sp",
  "guia_ref": "guia-erros-2026-006",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "005711"
  },
  "especialidade": "clinica_medica",
  "paciente": {
    "ref": "pac-erro-006",
    "sexo": "M",
    "idade": 45
  },
  "evolucao_clinica": "Atendimento ambulatorial para reavaliação de exames.",
  "cids": ["C539"],
  "procedimentos": [
    {
      "codigo_tuss": "10101012",
      "quantidade": 2
    },
    {
      "codigo_tuss": "10101039",
      "quantidade": 1
    },
    {
      "codigo_tuss": "40901300",
      "quantidade": 1
    },
    {
      "codigo_tuss": "99990001",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Reavaliação clínica.",
  "opcoes": {
    "gerar_justificativa": false,
    "sugerir_codigos": false
  }
}
```

#### O que observar na resposta (`200 OK`):
1. **Risco Alto (`risco: "alto"`):** retorno instantâneo (< 30 ms, sem custo de IA pois `opcoes` desabilitou geração) listando os **5 alertas bloqueantes**:
   - `TISS-CID-SEXO:C539` (`cid_incompativel_sexo`)
   - `TISS-USG-TV-SEXO` (`incompativel_idade_sexo`)
   - `TISS-CONSULTA-QTD` (`quantidade_invalida`)
   - `EXCL-CONSULTAS-CONCOMITANTES` (`procedimento_excludente`)
   - `TISS-TUSS-VIGENCIA:99990001` (`tuss_invalido`)

---

### Cenário 07 — Conformidade LGPD: Bloqueio de PII Estruturado (`422`) vs. Sanitização de Texto Livre (`200`)

#### 7A — Tentativa de envio de CPF/Nome no JSON (`Deve retornar 422 Unprocessable Entity`)
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "guia_ref": "guia-lgpd-422",
  "tipo_guia": "consulta",
  "operadora": {
    "registro_ans": "005711"
  },
  "paciente": {
    "ref": "pac-007",
    "nome": "João Carlos da Silva",
    "cpf": "123.456.789-00",
    "sexo": "M",
    "idade": 50
  },
  "evolucao_clinica": "Consulta de rotina.",
  "cids": ["I10"],
  "procedimentos": [
    {
      "codigo_tuss": "10101012",
      "quantidade": 1
    }
  ]
}
```

#### 7B — Dados sensíveis digitados acidentalmente no meio da evolução clínica (`Deve retornar 200 OK com sanitização em memória`)
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "guia_ref": "guia-lgpd-200",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "005711"
  },
  "especialidade": "cardiologia",
  "paciente": {
    "ref": "pac-007",
    "sexo": "M",
    "idade": 50
  },
  "evolucao_clinica": "Paciente: João Carlos da Silva, CPF 123.456.789-00, CNS 700001234567890, telefone (11) 98765-4321, e-mail joao.silva@email.com.br, encaminhado pelo Dr. Roberto Almeida (CRM-SP 123456). Apresenta palpitações recorrentes e episódios de pré-síncope há 3 semanas. Solicito Holter de 24 horas.",
  "cids": ["R002"],
  "procedimentos": [
    {
      "codigo_tuss": "20102011",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Investigação de arritmia cardíaca por palpitações e pré-síncope.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```
- **O que observar na resposta (`200 OK`):** a análise ocorre normalmente e nem a justificativa sugerida nem as evidências expõem o CPF, CNS, telefone ou e-mail do paciente.

---

## Bloco B — Ciclo de Vida Completo: Autocorreção, Feedback, Retorno de Glosa, XML TISS, Contratos e Guias Esquecidas

### Cenário 08 — Registro de Feedback da Pessoa Faturista (`POST /v1/analises/{analise_id}/feedback`) e Reenvio da Guia Corrigida

#### Passo 8.1 — Registrar o Feedback da Análise do Cenário 01
- **Endpoint:** `POST {{base_url}}/v1/analises/{{analise_id}}/feedback` *(substitua `{{analise_id}}` pelo UUID retornado no Cenário 01)*

```json
{
  "papel": "faturista",
  "itens": [
    {
      "item_id": "TISS-SADT-INDICACAO",
      "acao": "aceito"
    },
    {
      "item_id": "OP-005711-AUTH-PREVIA",
      "acao": "aceito"
    },
    {
      "item_id": "41001230:justificativa",
      "acao": "editado",
      "texto_final": "Quadro clínico: dor torácica atípica há 4 semanas com probabilidade pré-teste intermediária para DAC. Exames prévios: ECG basal alterado e teste ergométrico inconclusivo há 10 dias. Indicação: solicita-se Angiotomografia coronariana (41001230) e Ecodopplercardiograma transtorácico (40901106) sob CID I209."
    },
    {
      "item_id": "sug_tuss:40901106",
      "acao": "aceito"
    }
  ]
}
```
- **Resposta esperada:** `204 No Content`.

#### Passo 8.2 — Reenviar a Guia Corrigida Mantendo o Mesmo `guia_ref` (`"guia-cardio-2026-001"`)
- **Endpoint:** `POST {{base_url}}/v1/guias/analisar`

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "guia_ref": "guia-cardio-2026-001",
  "tipo_guia": "sp_sadt",
  "operadora": {
    "registro_ans": "005711"
  },
  "especialidade": "cardiologia",
  "paciente": {
    "ref": "pac-cardio-001",
    "sexo": "M",
    "idade": 58
  },
  "evolucao_clinica": "Paciente sintomático com dor torácica atípica há 4 semanas e probabilidade pré-teste intermediária de doença arterial coronariana (DAC). Traz ECG basal com alterações inespecíficas de repolarização e teste ergométrico realizado há 10 dias com resultado inconclusivo por fadiga precoce. Solicito angiotomografia coronariana para estratificação anatômica e ecodopplercardiograma transtorácico para avaliação da função ventricular.",
  "cids": ["I209"],
  "procedimentos": [
    {
      "codigo_tuss": "41001230",
      "quantidade": 1
    },
    {
      "codigo_tuss": "40901106",
      "quantidade": 1
    }
  ],
  "indicacao_clinica": "Quadro clínico: dor torácica atípica há 4 semanas com probabilidade pré-teste intermediária para DAC. Exames prévios: ECG basal alterado e teste ergométrico inconclusivo há 10 dias. Indicação: solicita-se Angiotomografia coronariana (41001230) e Ecodopplercardiograma transtorácico (40901106) sob CID I209.",
  "opcoes": {
    "gerar_justificativa": true,
    "sugerir_codigos": true
  }
}
```
- **O que observar:** o alerta bloqueante `TISS-SADT-INDICACAO` desaparece, o Ecocardiograma `40901106` agora consta como `"ja_informado"`, e o banco registra automaticamente a revisão `rev_num = 2` para medir a **taxa de autocorreção na origem**.

---

### Cenário 09 — Conciliação de Retorno de Faturamento / Glosas (`POST /v1/desfechos`)

- **Situação real simulada:** O demonstrativo de retorno da operadora chegou. A guia do Cenário 01 (`guia-cardio-2026-001`, que foi corrigida antes do envio) foi **paga integralmente**, enquanto a guia do Cenário 02 (`guia-orto-2026-002`, que tinha alerta de DUT não conforme) sofreu **glosa total** pelo código `1706`.
- **Endpoint:** `POST {{base_url}}/v1/desfechos`

#### Request Body (`raw / JSON`)

```json
{
  "lote_ref": "retorno-mensal-2026-11",
  "itens": [
    {
      "clinica_ref": "clinica-cardio-paulista",
      "guia_ref": "guia-cardio-2026-001",
      "status": "pago",
      "valor_cobrado": 1450.00,
      "valor_glosado": 0.00
    },
    {
      "clinica_ref": "clinica-ortopedia-sp-01",
      "guia_ref": "guia-orto-2026-002",
      "status": "glosado_total",
      "valor_cobrado": 920.00,
      "valor_glosado": 920.00,
      "codigo_glosa_tuss": "1706",
      "motivo_glosa_texto": "Ausência de comprovação de falha de tratamento conservador e fisioterápico por no mínimo 3 meses conforme DUT 65 ANS."
    }
  ]
}
```

#### Resposta Esperada (`200 OK`):
```json
{
  "processados": 2,
  "nao_encontrados": []
}
```

---

### Cenário 10 — Ingestão Direta de Lote XML TISS (`POST /v1/tiss/lotes/ingerir`) e Calibração do Score Preditivo

- **Situação real simulada:** A clínica importa um arquivo XML TISS (`4.01.00`) de demonstrativo de retorno da operadora `005711` contendo histórico de glosas para o procedimento `41001230`.
- **Endpoint:** `POST {{base_url}}/v1/tiss/lotes/ingerir`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "xml_conteudo": "<?xml version=\"1.0\" encoding=\"UTF-8\"?><ans:mensagemTISS xmlns:ans=\"http://www.ans.gov.br/padroes/tiss/schemas\"><ans:cabecalho><ans:identificacaoTransacao><ans:tipoTransacao>DEMONSTRATIVO_RETORNO</ans:tipoTransacao><ans:sequencialTransacao>1001</ans:sequencialTransacao><ans:dataRegistroTransacao>2026-11-05</ans:dataRegistroTransacao><ans:horaRegistroTransacao>10:00:00</ans:horaRegistroTransacao></ans:identificacaoTransacao><ans:origem><ans:registroANS>005711</ans:registroANS></ans:origem><ans:destino><ans:codigoPrestadorNaOperadora>998877</ans:codigoPrestadorNaOperadora></ans:destino><ans:Padrao>4.01.00</ans:Padrao></ans:cabecalho><ans:operadoraParaPrestador><ans:demonstrativosRetorno><ans:demonstrativoAnaliseConta><ans:numeroProtocolo>PROT-2026-1105</ans:numeroProtocolo><ans:dadosConta><ans:relacaoGuias><ans:numeroGuiaPrestador>guia-cardio-2026-001</ans:numeroGuiaPrestador><ans:dataRealizacao>2026-11-01</ans:dataRealizacao><ans:procedimentoRealizado><ans:codigoProcedimento>41001230</ans:codigoProcedimento><ans:valorInformado>980.00</ans:valorInformado><ans:valorGlosa>980.00</ans:valorGlosa><ans:motivoGlosa><ans:codigoGlosa>1706</ans:codigoGlosa></ans:motivoGlosa></ans:procedimentoRealizado></ans:relacaoGuias></ans:dadosConta></ans:demonstrativoAnaliseConta></ans:demonstrativosRetorno></ans:operadoraParaPrestador></ans:mensagemTISS>"
}
```

#### Resposta Esperada (`200 OK`):
```json
{
  "tipo_mensagem": "demonstrativo_retorno",
  "versao_tiss": "4.01.00",
  "lote_ref": "PROT-2026-1105",
  "registro_ans": "005711",
  "itens_ingeridos": 1,
  "desfechos_vinculados": 1
}
```

---

### Cenário 11 — Auditoria de Tabela Contratual vs. Sistema de Gestão (`POST /v1/contratos/auditar`)

- **Situação real simulada:** A clínica renegociou sua tabela com a operadora `326305` (onde a Cintilografia `40701018` passou a pagar R$ 1.150,00 e exigir autorização prévia, a Infiltração `30713030` passou a pagar R$ 650,00 e o Tilt Teste `20102038` foi incluído no contrato por R$ 480,00). Porém, o sistema da clínica ainda está com valores antigos, sem flag de autorização prévia na Cintilografia e sem o Tilt Teste cadastrado.
- **Endpoint:** `POST {{base_url}}/v1/contratos/auditar`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "registro_ans": "326305",
  "provisionar_regras_autorizacao": true,
  "itens_contrato": [
    {
      "codigo_tuss": "40701018",
      "valor_contratado": 1150.00,
      "exige_autorizacao_previa": true,
      "observacoes": "Exige pedido médico e teste ergométrico prévio"
    },
    {
      "codigo_tuss": "30713030",
      "valor_contratado": 650.00,
      "exige_autorizacao_previa": false,
      "observacoes": null
    },
    {
      "codigo_tuss": "20102038",
      "valor_contratado": 480.00,
      "exige_autorizacao_previa": false,
      "observacoes": "Novo procedimento aditivado em 2026"
    }
  ],
  "itens_pep": [
    {
      "codigo_tuss": "40701018",
      "valor_configurado": 980.00,
      "exige_autorizacao_previa": false
    },
    {
      "codigo_tuss": "30713030",
      "valor_configurado": 720.00,
      "exige_autorizacao_previa": false
    }
  ]
}
```

#### O que observar na resposta (`200 OK`):
1. **4 Divergências Detectadas:**
   - `40701018`: `valor_abaixo_do_contrato` (sistema cobrando R$ 170,00 a menos por exame!) + `autorizacao_previa_nao_configurada`;
   - `30713030`: `valor_acima_do_contrato` (sistema cobrando R$ 70,00 acima do teto contratual, gerando glosa parcial recorrente);
   - `20102038`: `procedimento_ausente_no_pep` (procedimento contratado não parametrizado no sistema).
2. **Provisionamento Automático de Regra (`regras_autorizacao_criadas: 1`):** se você chamar `POST /v1/guias/analisar` agora para `clinica-cardio-paulista` na operadora `326305` com o código `40701018`, a API passará a emitir automaticamente o alerta de autorização prévia contratual!

---

### Cenário 12 — Conciliação de Agenda vs. Guias Emitidas (`POST /v1/guias/esquecidas`)

- **Situação real simulada:** No fechamento da semana, o sistema da clínica envia os 3 atendimentos marcados como realizados na agenda de `clinica-cardio-paulista`:
  1. Atendimento `ag-1001` (`guia_ref: "guia-cardio-2026-001"` — já analisada no Cenário 01);
  2. Atendimento `ag-1002` (`guia_ref: null` — médico atendeu o paciente, mas nenhuma guia TISS foi gerada);
  3. Atendimento `ag-1003` (`guia_ref: "guia-fantasma-999"` — número de guia preenchido na agenda, mas que nunca passou pelo faturamento nem por lote XML TISS).
- **Endpoint:** `POST {{base_url}}/v1/guias/esquecidas`

#### Request Body (`raw / JSON`)

```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "atendimentos": [
    {
      "atendimento_ref": "ag-1001",
      "paciente_ref": "pac-cardio-001",
      "data_atendimento": "2026-11-05",
      "registro_ans": "005711",
      "codigo_tuss": "41001230",
      "guia_ref": "guia-cardio-2026-001",
      "valor_estimado": 980.00
    },
    {
      "atendimento_ref": "ag-1002",
      "paciente_ref": "pac-cardio-099",
      "data_atendimento": "2026-11-05",
      "registro_ans": "005711",
      "codigo_tuss": "40901106",
      "guia_ref": null,
      "valor_estimado": 420.00
    },
    {
      "atendimento_ref": "ag-1003",
      "paciente_ref": "pac-cardio-105",
      "data_atendimento": "2026-11-05",
      "registro_ans": "326305",
      "codigo_tuss": "20102011",
      "guia_ref": "guia-fantasma-999",
      "valor_estimado": 310.00
    }
  ]
}
```

#### Resposta Esperada (`200 OK`):
```json
{
  "clinica_ref": "clinica-cardio-paulista",
  "total_atendimentos_verificados": 3,
  "total_guias_esquecidas": 2,
  "valor_total_em_risco": "730.00",
  "guias_esquecidas": [
    {
      "atendimento_ref": "ag-1002",
      "paciente_ref": "pac-cardio-099",
      "data_atendimento": "2026-11-05",
      "registro_ans": "005711",
      "codigo_tuss": "40901106",
      "valor_estimado": "420.00",
      "motivo": "Atendimento realizado na agenda sem guia TISS vinculada no PEP."
    },
    {
      "atendimento_ref": "ag-1003",
      "paciente_ref": "pac-cardio-105",
      "data_atendimento": "2026-11-05",
      "registro_ans": "326305",
      "codigo_tuss": "20102011",
      "valor_estimado": "310.00",
      "motivo": "Guia 'guia-fantasma-999' não localizada nas análises pré-envio nem nos lotes XML TISS faturados."
    }
  ]
}
```
