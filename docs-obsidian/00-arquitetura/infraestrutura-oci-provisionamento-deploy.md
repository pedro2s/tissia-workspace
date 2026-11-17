---
projeto: ecossistema
tipo: operacao
status: ativo
data: 2026-11-17
tags:
  - tissia
  - ecossistema
  - oci
  - infraestrutura
  - deploy
  - always-free
aliases:
  - Infraestrutura OCI
  - Provisionamento OCI
  - Deploy OCI Always Free
---

# Infraestrutura OCI (`sa-saopaulo-1`) — Provisionamento e Deploy Always Free

Guia operacional para provisionar e operar o ecossistema **Tissia (`PEP` + `tissia-ia` + Keycloak + PostgreSQL 16 + Redis)** na **Oracle Cloud Infrastructure (OCI)** usando o **Always Free Tier** na região **`sa-saopaulo-1` (São Paulo — conformidade LGPD)**.

> **Referências:** [[auditoria-mvp-plano-orquestracao|Auditoria MVP]], [[refatoracao-oci-always-free-plano|Plano Técnico de Refatoração OCI]], [[ADR-PEP-0009-hospedagem-gcp-sao-paulo|ADR-PEP-0009]].

---

## 1. Mapa de Recursos Provisionados na OCI

| Recurso OCI | Cota Always Free Utilizada | Função no Ecossistema |
| :--- | :--- | :--- |
| **Região** | `sa-saopaulo-1` (São Paulo) | Residência de dados de saúde no Brasil (LGPD Art. 11). |
| **Compute (VM Principal)** | `VM.Standard.A1.Flex` (**ARM64**, 2 OCPUs, 12 GB RAM) | Executa Docker Compose único: `caddy`, `pep-ui`, `pep-backoffice-ui`, `pep-api`, `pep-worker`, `pep-backoffice`, `tissia-ia`, `keycloak`, `postgres` (`pgvector:pg16`) e `redis`. |
| **Compute (Auxiliar)** | Até 2× `VM.Standard.E2.1.Micro` (1/8 OCPU, 1 GB RAM) | Opcional: Bastion/Jump Host, agente de monitoramento ou ambiente de testes leve. |
| **Armazenamento (Disco)** | Block Volume até 200 GB (ex.: 50 GB Boot + 120 GB Dados) | Volume persistente montado em `/srv/tissia-data` (dados do Postgres 16, âncoras locais se usadas, certificados Caddy). |
| **Snapshots de Disco** | Até 5 backups de Block Volume | Política semanal/pré-upgrade no volume de dados. |
| **Object Storage** | 10 GB Standard + 10 GB Archive (50.000 req/mês) | Bucket `tissia-auditoria-ancoras` (com *Retention Rule* WORM) + Bucket `tissia-backups` (`pg_dump` diário criptografado). |
| **Rede (VCN + LB)** | 1 VCN + Internet Gateway + Security List / NSG | Libera portas `80/443` (Caddy) e `22` (restrita por IP ou OCI Bastion). Opcional: OCI Load Balancer Always Free (10 Mbps) na frente da VM. |
| **Email Delivery** | Até 3.000 e-mails/mês via SMTP | Envio de convites de onboarding (`TenantProvisionado`) e notificações transacionais (`SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`). |
| **Vault / Secrets** | Até 150 secrets (software) | Armazenamento de segredos fora do código-fonte (ou `.env.prod` com permissão `0600` na VM para operação simples). |
| **Monitoring & APM** | 500M pontos + 10 monitores sintéticos/hora | Checagem externa contínua em `/healthz` e `/readyz` + alertas de queda/disco. |
| **LLM Externo (GCP)** | Google Vertex AI (`global` / Service Account ADC) | Única dependência externa paga por uso (tokens Gemini 3.x), via `GOOGLE_APPLICATION_CREDENTIALS` montado no container `tissia-ia`. |

---

## 2. Passo a Passo de Provisionamento na Console / CLI da OCI

### 2.1 Rede (VCN e Firewall)
1. Criar uma VCN em `sa-saopaulo-1` (ex.: `vcn-tissia-prod`, CIDR `10.0.0.0/16`) com **Subnet Pública** (`10.0.1.0/24`) e **Internet Gateway**.
2. Configurar **Security List / Network Security Group**:
   - **Ingress permitido:**
     - TCP `80` e `443` de `0.0.0.0/0` (Caddy/TLS).
     - TCP `22` apenas do seu IP fixo ou desabilitado usando **OCI Bastion** (Always Free).
   - **Ingress bloqueado:** portas `5432` (Postgres), `6379` (Redis), `8080` (`tissia-ia`) e `3000/3001` nunca devem ser expostas na rede pública.
3. No SO da VM (Ubuntu 24.04 Minimal ARM64), ajustar `iptables`/firewall local para liberar `80` e `443` (a imagem Ubuntu padrão da OCI bloqueia portas HTTP no `iptables` mesmo após abrir na VCN!).

### 2.2 Compute (`VM.Standard.A1.Flex`) e Block Volume
1. Criar instância **Ubuntu 24.04 Minimal (`aarch64` / ARM64)**:
   - Shape: **`VM.Standard.A1.Flex`** — **2 OCPUs**, **12 GB RAM** (dentro dos 1.500 OCPU-h e 9.000 GB-h/mês contínuos).
   - Boot Volume: **50 GB**.
   - Atribuir **IP Público Reservado** (para apontamento DNS estável).
2. Criar um **Block Volume** adicional de **100–150 GB** (`tissia-pg-data`), anexar via paravirtualizado à VM, formatar em `ext4` e montar em `/srv/tissia-data` no `/etc/fstab` (com `nofail`).
3. Instalar **Docker Engine + Docker Compose v2** na VM (`apt-get install docker.io docker-compose-v2`).

### 2.3 Object Storage (Âncoras WORM + Backups)
1. Criar bucket **`tissia-auditoria-ancoras`** (Tier *Standard*):
   - Habilitar **Retention Rule** (ex.: retenção bloqueada de 5 anos — *Time-Bound Retention Rule* com *Lock*) para garantir imutabilidade WORM das âncoras diárias de auditoria.
   - Gerar **Customer Secret Keys** (Access Key + Secret Key) para uso via **S3 Compatibility API** da OCI (`https://<namespace>.compat.objectstorage.sa-saopaulo-1.oraclecloud.com`).
2. Criar bucket **`tissia-backups-db`** (Tier *Standard* ou *Archive*) com regra de ciclo de vida (ex.: mover para *Archive* após 7 dias, expirar após 90 dias).

### 2.4 Email Delivery (SMTP Transacional)
1. Em **Developer Services → Email Delivery** (`sa-saopaulo-1`):
   - Configurar e verificar o domínio remetente (SPF + DKIM no DNS, ex.: `tissia.com.br`).
   - Criar **Approved Sender** (ex.: `no-reply@tissia.com.br`).
   - Gerar **SMTP Credentials** para o usuário IAM e anotar o endpoint SMTP (`smtp.email.sa-saopaulo-1.oci.oraclecloud.com`, porta `587`).

### 2.5 DNS e Subdomínios
Apontar registros `A` para o IP público reservado da VM (ou IP do OCI Load Balancer):
- `app.<dominio>` → `@pep/ui`
- `api.<dominio>` → `@pep/api` (ou roteado em `app.<dominio>/api`)
- `bo.<dominio>` → `@pep/backoffice` + `pep-backoffice`
- `auth.<dominio>` → `keycloak`

---

## 3. Topologia de Containers (`infra/oci/docker-compose.prod.yml`)

Todos os serviços rodam na mesma rede Docker privada, expostos exclusivamente através do **Caddy** (HTTPS automático via Let's Encrypt):

- `caddy` (portas `80`, `443`) → Proxy reverso único + terminação TLS.
- `postgres` (`pgvector/pgvector:pg16`) → Bancos `pep` e `tissia_ai`, dados em `/srv/tissia-data/postgres`.
- `redis` (`redis:7-alpine`) → Filas BullMQ (`pep-api` + `pep-worker`).
- `keycloak` (`quay.io/keycloak/keycloak:26.0`) → Realms `pep-prod` e `tissia-equipe-prod`, persistido no Postgres.
- `tissia-ai` (imagem `linux/arm64`) → Rede interna apenas (`http://tissia-ai:8080`), conectado à Vertex AI via `/secrets/gcp-adc.json:ro`.
- `pep-api` (`node dist/main.js`, `WORKER_EMBUTIDO=false`) → API clínica.
- `pep-worker` (`node dist/worker.js`) → Consumidores outbox + selagem de auditoria (gravando âncora no Object Storage S3 ou volume `/srv/tissia-data/ancoras`).
- `pep-backoffice` (`node dist/backoffice.js`) → API interna `/bo/v1`.
- `pep-ui` e `pep-backoffice-ui` → Estáticos servidos pelo Nginx/Caddy.

---

## 4. Fluxo de Deploy e Atualização

### 4.1 Primeiro Deploy na VM
1. Clonar/sincronizar o diretório `infra/oci/` na VM (`/opt/tissia/`).
2. Copiar `.env.oci.example` para `.env.prod` (permissão `0600`) preenchendo todas as chaves fortes (`SESSAO_JWT_SEGREDO`, `BO_SESSAO_SEGREDO`, `PII_CHAVE_MESTRA`, `TISSIA_AI_SAL_PACIENTE`, `SMTP_*`, `AUDITORIA_ANCORA_S3_*`, credenciais do banco).
3. Colocar a chave JSON da Service Account GCP (apenas permissão `roles/aiplatform.user`) em `/opt/tissia/secrets/gcp-adc.json` (`0400`).
4. Subir infraestrutura base e executar migrações:
   ```bash
   docker compose -f docker-compose.prod.yml --env-file .env.prod up -d postgres redis
   docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm pep-migrator
   docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm tissia-ai-migrator
   docker compose -f docker-compose.prod.yml --env-file .env.prod up -d
   ```

### 4.2 Atualizações Contínuas (CI/CD → VM)
1. O GitHub Actions constrói imagens **`linux/arm64`** (`docker buildx`) para `PEP` e `tissia-ia` e publica no **GHCR** (`ghcr.io/pedro2s/...`).
2. Na VM (manualmente ou via step SSH no deploy):
   ```bash
   docker compose -f docker-compose.prod.yml --env-file .env.prod pull
   docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm pep-migrator
   docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm tissia-ai-migrator
   docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --remove-orphans
   ```

---

## 5. Rotina de Backup, Monitoramento e Prevenção de Reclaim

- **Backup diário (`cron` às 03:30 BRT):** script `backup-db.sh` executa `pg_dumpall`, criptografa com `openssl` / `age` e envia para o bucket `tissia-backups-db` via S3/OCI CLI.
- **Monitoramento sintético (OCI APM):** configurar checks a cada 10 minutos para `https://api.<dominio>/readyz` e `https://app.<dominio>` com alerta por e-mail/Notifications (além de evitar inatividade da VM Always Free).
- **Observação sobre capacidade Always Free:** manter o consumo contínuo abaixo de 1.500 OCPU-h/mês e 9.000 GB-h/mês (2 OCPUs e 12 GB RAM = ~1.460 OCPU-h e ~8.760 GB-h em meses de 30 dias — **atenção em meses de 31 dias = 744h**: `2 * 744 = 1.488 OCPU-h` e `12 * 744 = 8.928 GB-h`, ambos dentro do teto!).

