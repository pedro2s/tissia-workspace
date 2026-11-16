---
projeto: PEP
tipo: adr
status: aceita
tags:
  - tissia
  - pep
  - adr
  - decisao-arquitetural
origem: tissia/docs/adr/ADR-PEP-0012-motor-tiss.md
---

# ADR-PEP-0012 — Geração de XML TISS própria; webservices por operadora via parceiro avaliado

**Status:** Aceita — 13/11/2026 (proposta de 07/11/2026; atualizada em 13/11/2026 com a versão vigente, o CID e o projeto `tiss`)

## Contexto

Faturar convênio exige: gerar guias e lotes XML no padrão TISS, enviar à operadora (portal, webservice ou upload), consultar elegibilidade, pedir autorização prévia e importar o demonstrativo de retorno. Cada operadora tem particularidades de webservice e homologação. É o item mais caro do produto e a decisão em aberto nº 1 do design de produto. A Spike **não** gera XML nem fala com operadoras; ela só lê XML (`/v1/tiss/lotes/ingerir`) e analisa guias.

Versão vigente do Padrão TISS publicada pela ANS (competência setembro/2026): Componente Organizacional 202609, Componente de Conteúdo e Estrutura 202511, Componente de Comunicação 04.03.00 (prestador ↔ operadora: guias, elegibilidade, autorização, lotes, demonstrativos e recurso de glosa), Representação de Conceitos em Saúde (TUSS) 202609. A versão 01.06.00 do Componente de Comunicação é a do Monitoramento TISS (operadora → ANS) e não se aplica ao PEP, que é prestador. O PEP planejava 4.01, já superada.

## Decisão

- **MVP (P1): geração própria de guias (consulta, SP/SADT) e lote XML TISS 4.03.00**, validados contra os XSD oficiais da ANS antes de liberar o download, com hash do lote calculado conforme o padrão. Envio na P1 = download do XML para upload no portal da operadora pelo faturista.
- **Leitura do demonstrativo de retorno** própria (P2), reaproveitando a lógica e os testes do parser da Spike como referência; o XML também vai para a Spike via `ingerir`.
- **Webservices por operadora** (elegibilidade, autorização prévia, envio de lote, status de protocolo): **avaliar parceiro homologado na P2** antes de construir. Interface única no módulo `integracoes` (`OperadoraGateway`) para trocar implementação sem mexer em `faturamento`.
- **Leitura do demonstrativo:** 4.03.00; versões anteriores só se alguma operadora do piloto ainda devolver demonstrativo nelas **[validar]**.
- Versão TISS é configuração por operadora, não constante.
- **CID fora do XML de SP/SADT e de consulta:** no 4.03.00 essas guias não têm tag estruturada de CID (a consulta perdeu o CID por sigilo médico) e uma tag inserida à mão reprova no XSD. O CID fica no prontuário e na análise da Spike, nunca no XML dessas guias. Diagnóstico só chega à operadora como texto que o médico escreve ou aceita na Indicação Clínica (`indicacaoClinica`); o PEP **não** injeta código CID nesse texto. CID estruturado (anexos de quimioterapia, radioterapia e OPME; solicitação e resumo de internação) fica fora do MVP.
- **Projeto `tiss` (Voa Health, Apache-2.0) só como referência:** leitura do código e aproveitamento de dados (XSDs oficiais com proveniência, Tabela 38, CADOP, campos da SP/SADT), nunca como parte do produto. Nada de importar a biblioteca, subir o serviço ou chamar a API hospedada (receberia PII). Lógica inspirada nele é reescrita no PEP com atribuição; dados seguem a licença da fonte (ANS) e ficam registrados com proveniência. **Em aberto:** usar o `tiss` como validador do XML gerado, só na CI e como segunda opinião ao XSD (`libxml2-wasm`), sem entrar no runtime.

## Alternativas consideradas

| Alternativa | Por que não |
| :--- | :--- |
| Construir tudo (XML + webservices de todas as operadoras) | Homologação operadora por operadora não cabe em 90 dias |
| Comprar tudo (motor TISS de parceiro, inclusive geração) | Gera dependência no núcleo do produto; o diferencial é a guia nascer da evolução, que precisa do modelo de guia nosso |
| Usar a Spike para gerar XML | Fora do escopo dela e quebraria o produto standalone |
| Usar a biblioteca ou a API `tiss` como motor | Python fora da stack ([[ADR-PEP-0002-backend-nestjs-monolito-modular\|ADR-PEP-0002]]); a API hospedada receberia dados do beneficiário; dependência externa no núcleo, o mesmo problema de "comprar tudo" |

## Consequências

- P1 sem envio automático: o faturista faz upload no portal; métrica de tempo de faturamento considera isso.
- Atualização do padrão TISS vira tarefa recorrente (monitorar ANS); XSD versionados no repo com proveniência (URL e SHA-256 do pacote da ANS).
- A Spike hoje só lê XML até 4.01.00 (`app/tiss_xml.py`): um demonstrativo 4.03.00 seria recusado no `ingerir`. Proposta em [[SPK-10-tiss-4-03|SPK-10]].
- Origem, versão e alerta de defasagem de XSD, Tabela 38, domínios e operadoras: [[ADR-PEP-0024-dados-de-referencia-ans|ADR-PEP-0024]].
- Detalhes na [[RFC-PEP-006-guias-e-lotes-tiss|RFC-PEP-006]] e [[integracoes-externas|integracoes]].

**Aceite (13/11/2026):** aceita pelo responsável do produto.
- **Webservices:** ainda não há parceiro homologado. A P1 segue só com geração de XML e envio manual pelo portal; a avaliação de parceiro continua na P2, atrás do `OperadoraGateway`.
- **Operadoras e canais de envio do piloto:** os dados de `operadoras/_data` do projeto `tiss` (CADOP e indicadores de glosa da ANS) não trazem volume nem canal de envio, então não definem quais são as 2–3 operadoras de maior volume. O dado vem da carteira da própria clínica-piloto e vai para o cadastro de contratos (`canal_envio`, `versao_tiss`, RFC-PEP-008 fatia 0), não bloqueia a onda 5.
- **Versões anteriores à 4.03.00:** segue **[validar]** por operadora do piloto (RFC-PEP-006, Q1).
