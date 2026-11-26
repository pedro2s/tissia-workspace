---
projeto: PEP
tipo: operacao
status: ativo
tags:
  - tissia
  - pep
  - operacao
  - demo
  - docker
origem: tissia/docs/demo.md
---

# Ambiente de demonstração

Ambiente **descartável** do Marco D ([[release-plan-pep#126-ambiente-de-demonstração|release plan §12.6]]): UI, API, worker, Keycloak, Postgres, Redis e a Spike (mesma imagem do `../tissia-ai`, sem fork) em Docker Compose, atrás de um Caddy. Ponto de entrada único: `/` UI, `/api` API, `/auth` Keycloak. Não é a arquitetura de produção ([[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009]] volta na P1).

> **Segurança.** A demo **nunca** usa dado real: tudo é sintético ("Exemplo"). O realm `pep-demo` (login só com senha, sem segundo fator) e `MFA_AMR_ACEITOS=pwd` **nunca vão para produção**; por isso a imagem roda com `NODE_ENV=development` (a API recusa `pwd` em `production`). Segredos ficam em `infra/demo/.env` (ignorado no git); a credencial do Vertex nunca entra no repositório.

> **Assinatura simulada.** O `apps/api/.env.demo` define `ASSINATURA_PROVEDOR=local` e um PIN sintético para a demo poder assinar receita de controle especial e antimicrobiano. É um provedor determinístico sem valor jurídico, **proibido em produção** (a API recusa o boot); ver [[ADR-PEP-0036-defaults-fail-closed-autenticacao-e-assinatura|ADR-PEP-0036]] e [[RFC-PEP-011-assinatura-icp-brasil|RFC-PEP-011]].

## Rodar a demo na sua máquina (um comando)

```bash
pnpm demo:dev        # sobe Keycloak (realm pep-demo), migra, semeia e inicia API e UI em modo demonstração
```

Pare antes qualquer `pnpm dev` que esteja usando as portas 3000 e 5173. O comando usa `apps/api/.env.demo` e `apps/ui/.env.demo` (versionados, sem segredo): login real no Keycloak em `http://localhost:8180/realms/pep-demo`, sem `X-Dev-*` e sem IDs de tenant/unidade em variável de ambiente. Entre em `http://localhost:5173` com `dra.exemplo` e a senha `demonstracao-pep`. Esse realm é o único que aceita essa senha: o `pep-dev` (padrão do `pnpm dev`) tem outro usuário/senha e exige segundo fator, e é por isso que `VITE_AUTH_MODE=oidc` sozinho dá "Nome de usuário ou senha inválida".

## Arquivos (`infra/demo/`)

`docker-compose.yml` · `Dockerfile.pep` (api/worker/init) · `Dockerfile.ui` · `Caddyfile` · `gerar-env.sh` (segredos) · `gerar-realm.mjs` (acrescenta `DEMO_URL` ao realm) · `demo.sh up|reset|down|logs` · `.env.example` (todas as variáveis).

## Subir localmente

Pré-requisitos: Docker e o repo `tissia-ai` ao lado deste (`../tissia-ai`).

```bash
cd infra/demo
./gerar-env.sh            # cria .env com segredos aleatórios; local = http://localhost:8080
# Vertex (Gemini): edite GCP_PROJECT no .env; GCP_ADC_FILE já aponta para o ADC do gcloud, se existir.
# Sem credencial: IA_PROVEDOR=local (heurísticas da Spike).
./demo.sh up              # build + sobe tudo (~2 min na 1ª vez) e imprime a URL
```

Abra http://localhost:8080. `./demo.sh reset` recria os dados fictícios (migrações + seed; ~10 s; a agenda volta para "hoje"). `./demo.sh down` derruba e apaga volumes; `./demo.sh logs [serviço]`.

### Tema de login Keycloak (`tissia`)

Os realms `pep-demo` e `pep-dev` utilizam o tema customizado `tissia` (`loginTheme: "tissia"`). O artefato JAR gerado pelo Keycloakify é montado em `/opt/keycloak/providers/`:
- Para compilar o tema antes de subir a stack: `pnpm --filter @pep/keycloak-tema build` (gera `apps/keycloak-tema/dist_keycloak/keycloak-theme-tissia.jar`).
- Se o JAR ainda não tiver sido construído no ambiente local, o Keycloak subirá normalmente com o tema padrão como fallback automático.

## Subir em uma VM (GCP e2-standard-4, `southamerica-east1`)

Instruções; nada é executado por este repositório.

1. Criar a VM (Ubuntu 24.04, e2-standard-4, disco 50 GB) e liberar tráfego 80 e 443. Reservar IP estático e apontar `demo.<seu-domínio>` (registro A) para ele.
2. Instalar Docker (`curl -fsSL https://get.docker.com | sh`) e clonar `tissia` e `tissia-ai` na mesma pasta.
3. Vertex (Gemini) usa o ADC (Application Default Credentials), sem chave nem variável de segredo no repositório. **Local:** nada a fazer além de `gcloud auth application-default login`; o `gerar-env.sh` já aponta `GCP_ADC_FILE` para `~/.config/gcloud/application_default_credentials.json`. **Na VM:** copie esse mesmo arquivo de ADC (gerado com `gcloud auth application-default login` na VM) ou, se preferir uma service account com `roles/aiplatform.user`, use a chave JSON dela; em ambos os casos fora do repo (`chmod 644`) e informado em `GCP_ADC_FILE`. *Não testado:* anexar a service account à VM e usar o ADC por metadata, que exigiria tirar `GOOGLE_APPLICATION_CREDENTIALS` do compose.
4. `cd tissia/infra/demo && ./gerar-env.sh demo.<seu-domínio>`; editar `.env`: `GCP_PROJECT`, `GCP_ADC_FILE=/caminho/do/adc.json` (já preenchido se houver ADC do gcloud) e `IA_PROVEDOR=vertex`.
5. `./demo.sh up`. O Caddy emite o certificado sozinho (o DNS precisa já estar apontando). Teste `https://demo.<seu-domínio>`.
6. Antes de cada apresentação: `./demo.sh reset`.

## Usuários (realm `pep-demo`, senha `demonstracao-pep`)

| Usuário | Papel |
| :--- | :--- |
| `dra.exemplo` | Médica (Ortopedia) |
| `dr.exemplo.b` | Médico (Cardiologia) |
| `faturista.exemplo` | Faturista |
| `gestora.exemplo` | Gestora |
| `recepcao.exemplo` | Recepção (agenda e cadastro de pacientes) |

## Roteiro de 15 minutos (release plan §12.2)

1. Login como `dra.exemplo` e escolha da unidade (real).
2. Agenda do dia (Dia); abrir o próximo atendimento (caso das 09:30).
3. Prontuário: evolução SOAP de Ortopedia (LCA), pedido de RM e de artroscopia com indicação clínica.
4. Fechar o atendimento: guia SP/SADT rascunho e análise da Spike (risco, alertas, DUT 110, Campo 23 sugerido e editável).
5. Aceitar a justificativa, editar uma frase, rejeitar uma sugestão; a reanálise baixa o risco.
6. Derrubar a IA (flag) e mostrar a análise parcial: regras TISS seguem e o atendimento não trava.
7. Entrar como `faturista.exemplo`: mesa do faturista ordenada por risco.
8. "Glosas que teriam sido evitadas" (lote sintético, R$ por operadora e motivo; aviso de simulação).
9a. Voz ambiental: consentimento, escuta, SOAP pela Spike, revisão trecho a trecho e assinatura.
9. Tour de telas de prévia (selo "Prévia" no menu). 10. Opcional: backoffice.

## Plano B

Vídeo do roteiro completo gravado no ambiente e, no notebook, esta mesma demo local (`./demo.sh up`, com `IA_PROVEDOR=local` se a rede falhar; a análise perde a justificativa do Gemini, mas o fluxo se mantém).
