---
projeto: ecossistema
tipo: plano-refatoracao
status: concluido
data: 2026-11-17
tags:
  - tissia
  - ecossistema
  - oci
  - refatoracao
  - multi-agente
aliases:
  - Plano Refatoração OCI
  - Refatoração OCI Always Free
---

# Plano Técnico de Refatoração — Adaptação para OCI Always Free

Especificação técnica das alterações de código e empacotamento necessárias para rodar o ecossistema (`PEP` + `tissia-ia`) na **OCI (`sa-saopaulo-1`)** com mínimo custo e sem *overengineering*, mantendo compatibilidade com o fluxo atual de desenvolvimento/CI e com GCP caso necessário no futuro.

> **Guia operacional complementar:** [[infraestrutura-oci-provisionamento-deploy|Infraestrutura OCI — Provisionamento e Deploy]].

---

## 1. Visão das Frentes Paralelas (Multi-Agente)

O trabalho foi fatiado em **3 frentes independentes** com fronteiras de arquivos estritamente disjuntas:

| Frente | Repositório | Escopo Principal | Arquivos-Alvo |
| :--- | :--- | :--- | :--- |
| **Frente 1 — Âncora S3 / OCI Object Storage** | `projetos/PEP` | Adicionar `AncoraEmS3` (SigV4 nativo com `node:crypto` + `fetch`, zero dependência externa) compatível com OCI Object Storage S3 API / MinIO / R2. | `apps/api/src/modulos/auditoria/selagem.ts`, `apps/api/src/comum/config.ts`, `apps/api/.env.example`, `apps/api/test/auditoria-selagem.spec.ts` |
| **Frente 2 — Stack Unificada `infra/oci/`** | `projetos/PEP` | Criar `docker-compose.prod.yml` (ARM64-compatível), `Caddyfile`, `postgres-init.sql`, `.env.oci.example` e script `backup-db.sh`. | `infra/oci/*` (arquivos novos) |
| **Frente 3 — CI/CD Multi-Arch (`linux/arm64`)** | `projetos/PEP` e `projetos/tissia-ia` | Suporte a build `linux/arm64` (QEMU + Buildx) e publicação no GHCR (opcional/habilitado por variável), mantendo os testes atuais intactos. | `projetos/PEP/.github/workflows/oci-images.yml` (novo workflow) e `projetos/tissia-ia/.github/workflows/oci-images.yml` (novo workflow) |

---

## 2. Especificação Detalhada por Frente

### Frente 1 — `PEP`: Suporte a Âncora S3-Compatible (`AncoraEmS3`)
- **Motivação:** Atualmente `AncoraEmGcs` em `apps/api/src/modulos/auditoria/selagem.ts` é acoplada a `metadata.google.internal` e `storage.googleapis.com`. Para persistir âncoras imutáveis fora do banco na OCI (Object Storage com *Retention Rule*), precisamos de um adaptador S3-compatible padrão.
- **Requisitos Técnicos:**
  1. Em `apps/api/src/comum/config.ts`:
     - Adicionar variáveis opcionais:
       - `AUDITORIA_ANCORA_S3_ENDPOINT` (URL, ex.: `https://<ns>.compat.objectstorage.sa-saopaulo-1.oraclecloud.com`)
       - `AUDITORIA_ANCORA_S3_REGION` (default `'sa-saopaulo-1'`)
       - `AUDITORIA_ANCORA_S3_BUCKET` (string opcional)
       - `AUDITORIA_ANCORA_S3_ACCESS_KEY` (string opcional)
       - `AUDITORIA_ANCORA_S3_SECRET_KEY` (string opcional)
     - Atualizar a regra de validação de produção (`NODE_ENV=production`): aceitar se **`AUDITORIA_ANCORA_BUCKET`** OU **`AUDITORIA_ANCORA_S3_BUCKET`** OU **`AUDITORIA_ANCORA_DIR !== '.ancoras-auditoria'`** estiver definido.
     - Se `AUDITORIA_ANCORA_S3_BUCKET` estiver definido, exigir `AUDITORIA_ANCORA_S3_ENDPOINT`, `AUDITORIA_ANCORA_S3_ACCESS_KEY` e `AUDITORIA_ANCORA_S3_SECRET_KEY`.
  2. Em `apps/api/src/modulos/auditoria/selagem.ts`:
     - Implementar `AncoraEmS3 implements AncoraStore` usando assinatura **AWS Signature Version 4 (SigV4)** pura com `node:crypto` + `fetch` (sem adicionar `@aws-sdk/client-s3` para manter a imagem enxuta).
     - `gravar(a)`: `PUT` no objeto `<prefixo>/<tenantId>/<sequencia>.json` com header `If-None-Match: *` (idempotência: se retornar `412` ou `409`, faz `GET` e compara `hash`; se igual, retorna sem erro; se divergente, lança erro).
     - `todas(tenantId)`: `GET` `ListObjectsV2` (`?list-type=2&prefix=...`) paginado via `NextContinuationToken` e leitura dos objetos `.json` ordenados por `sequencia`.
     - Atualizar `criarAncoraStore(cfg)` para priorizar `AncoraEmS3` quando `AUDITORIA_ANCORA_S3_BUCKET` estiver definido, depois `AncoraEmGcs`, e por fim `AncoraEmArquivo`.
  3. Em `apps/api/.env.example`: documentar as novas variáveis `AUDITORIA_ANCORA_S3_*`.
  4. Em `apps/api/test/auditoria-selagem.spec.ts`: adicionar testes unitários com `http` mockado validando assinatura SigV4, `If-None-Match: *`, tratamento de `412` (idempotente vs. divergente) e paginação de `todas()`.

---

### Frente 2 — `PEP`: Empacotamento Unificado de Produção (`infra/oci/`)
- **Motivação:** Substituir 6 serviços Cloud Run + Cloud SQL + Memorystore por uma stack única reprodutível via Docker Compose na VM Arm `VM.Standard.A1.Flex` (`sa-saopaulo-1`), mantendo separação de bancos, papéis RLS e isolamento da `tissia-ia`.
- **Requisitos Técnicos:**
  1. Criar `infra/oci/postgres-init.sql`:
     - Garantir criação dos papéis (`app_rw`, `app_admin`, `app_plataforma`) e dos dois bancos (`pep` e `tissia_ai`) com extensões `vector` (`pgvector`) e `pg_trgm` no `tissia_ai`.
  2. Criar `infra/oci/Caddyfile`:
     - Reverse proxy único com HTTPS automático:
       - `{$PEP_DOMAIN}`: `/api/*` → `pep-api:3000`, demais rotas → `pep-ui:8080`.
       - `{$BO_DOMAIN}`: `/bo/*` → `pep-backoffice:3001`, demais rotas → `pep-backoffice-ui:8080`.
       - `{$AUTH_DOMAIN}`: `/*` → `keycloak:8080`.
  3. Criar `infra/oci/docker-compose.prod.yml`:
     - Serviços: `caddy` (`caddy:2-alpine`), `postgres` (`pgvector/pgvector:pg16` com tuning leve para 12 GB RAM total), `redis` (`redis:7-alpine`), `keycloak` (`quay.io/keycloak/keycloak:26.0` em modo `start` com Postgres), `pep-migrator` (profile/one-shot `db:setup`), `tissia-ai-migrator` (one-shot `alembic upgrade head`), `tissia-ai` (`IA_PROVEDOR=vertex`, `VERTEX_LOCATION=global`, credencial ADC montada `:ro`, sem porta pública), `pep-api` (`WORKER_EMBUTIDO=false`), `pep-worker`, `pep-backoffice`, `pep-ui` e `pep-backoffice-ui`.
     - Todos os serviços com `restart: unless-stopped`, `healthcheck`, limites de memória (`mem_limit`) adequados ao orçamento de 12 GB e volumes apontando para `${DATA_DIR:-/srv/tissia-data}`.
  4. Criar `infra/oci/.env.oci.example` com todas as variáveis mapeadas e comentadas (sem segredos reais).
  5. Criar `infra/oci/backup-db.sh` (executável): script para `pg_dumpall` compactado com `gzip` (e criptografia opcional via `BACKUP_PASSPHRASE` com `openssl enc -aes-256-cbc -pbkdf2`), pronto para ser agendado no `cron`.
  6. Criar `infra/oci/README.md` resumindo os comandos de subida, migração e backup.

---

### Frente 3 — `PEP` e `tissia-ia`: Workflows de Build Multi-Arch (`linux/arm64`)
- **Motivação:** `VM.Standard.A1.Flex` exige imagens `linux/arm64`. Os workflows `ci.yml` existentes já testam e validam o deploy GCP; criar workflows dedicados (`oci-images.yml`) garante build ARM64 limpo sem quebrar os workflows atuais.
- **Requisitos Técnicos:**
  1. Em `projetos/PEP/.github/workflows/oci-images.yml`:
     - Disparado em `workflow_dispatch` e `push` na `main` quando `vars.OCI_BUILD_HABILITADO == 'true'` (e em PR apenas build de validação ou `workflow_dispatch` para não gastar minutos sem necessidade; incluir `pull_request` com `push: false` apenas se rápido, ou `push` na `main` + `workflow_dispatch`).
     - Usar `docker/setup-qemu-action@v3` + `docker/setup-buildx-action@v3` + `docker/login-action@v3` (GHCR `ghcr.io`) + `docker/build-push-action@v6` com `platforms: linux/arm64` para `api` (`runner`), `migrator`, `ui` e `backoffice-ui`.
  2. Em `projetos/tissia-ia/.github/workflows/oci-images.yml`:
     - Mesmo padrão (`qemu` + `buildx` + `platforms: linux/arm64`) publicando `ghcr.io/${{ github.repository_owner }}/tissia-ai:${{ github.sha }}` (e `latest`) quando `vars.OCI_BUILD_HABILITADO == 'true'` ou via `workflow_dispatch`.
  3. Validar que ambos os `Dockerfile` (`PEP` e `tissia-ia`) não possuem instruções incompatíveis com `linux/arm64`.

---

## 3. Critérios de Aceite Globais
- [ ] `projetos/PEP`: `pnpm typecheck && pnpm deps && pnpm --filter @pep/api test -- test/auditoria-selagem.spec.ts` passando 100%.
- [ ] `docker compose -f infra/oci/docker-compose.prod.yml --env-file infra/oci/.env.oci.example config` validado sem erros de sintaxe.
- [ ] Workflows YAML válidos e sem alterar o comportamento dos `ci.yml` existentes.

