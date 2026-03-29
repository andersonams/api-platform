# Guia de Setup — CD Pipeline para VPS

## 1. GitHub — Secrets do repositório

Em **Settings → Secrets and variables → Actions**, criar:

| Secret | Valor |
|--------|-------|
| `DOCKERHUB_USERNAME` | `andersonmouradev` |
| `DOCKERHUB_TOKEN` | Access token do Docker Hub (criar em https://hub.docker.com/settings/security) |
| `VPS_HOST` | IP da VPS |
| `VPS_USER` | Usuário SSH (ex: `deploy`) |
| `VPS_SSH_KEY` | Conteúdo da chave privada SSH (ex: `~/.ssh/id_ed25519`) |

## 2. GitHub — Environment de produção

Em **Settings → Environments**, criar environment `production`:

1. Clicar em "New environment"
2. Nome: `production`
3. Marcar "Required reviewers"
4. Adicionar seu usuário como reviewer
5. Salvar

Isso cria o gate de aprovação manual antes de cada deploy.

## 3. Docker Hub — Access Token

1. Acessar https://hub.docker.com/settings/security
2. Clicar "New Access Token"
3. Descrição: `github-actions-deploy`
4. Permissão: **Read & Write**
5. Copiar o token e salvar como `DOCKERHUB_TOKEN` nos secrets do GitHub

## 4. VPS — Chave SSH

Na sua máquina local:

```bash
# Gerar chave dedicada para deploy (se ainda não tiver)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key

# Copiar a chave pública para a VPS
ssh-copy-id -i ~/.ssh/deploy_key.pub usuario@IP_DA_VPS
```

O conteúdo de `~/.ssh/deploy_key` (chave privada) vai no secret `VPS_SSH_KEY`.

## 5. VPS — Preparar o ambiente

```bash
# Criar diretório do projeto
sudo mkdir -p /opt/api-platform
sudo chown $USER:$USER /opt/api-platform

# Copiar os compose files para a VPS
scp compose.yaml compose.prod.yaml usuario@IP_DA_VPS:/opt/api-platform/

# Criar arquivo .env de produção na VPS
ssh usuario@IP_DA_VPS
cat > /opt/api-platform/.env << 'EOF'
# App
APP_SECRET=GERAR_UM_SECRET_ALEATORIO
SERVER_NAME=:80

# Database
POSTGRES_USER=app
POSTGRES_PASSWORD=GERAR_UMA_SENHA_FORTE
POSTGRES_DB=app
POSTGRES_VERSION=16

# Mercure
CADDY_MERCURE_JWT_SECRET=GERAR_UM_SECRET_ALEATORIO
CADDY_MERCURE_URL=http://php/.well-known/mercure
CADDY_MERCURE_PUBLIC_URL=http://SEU_IP/.well-known/mercure

# CORS
CORS_ALLOW_ORIGIN='^https?://SEU_IP(:[0-9]+)?$'

# Trusted
TRUSTED_PROXIES=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
TRUSTED_HOSTS='^SEU_IP|php$$'
EOF
```

Substituir `SEU_IP`, `GERAR_UM_SECRET_ALEATORIO` e `GERAR_UMA_SENHA_FORTE` por valores reais.

Para gerar secrets aleatórios:
```bash
openssl rand -hex 32
```

## 6. VPS — Firewall

```bash
# Liberar porta 80 (HTTP)
sudo ufw allow 80/tcp

# Verificar
sudo ufw status
```

## 7. Primeiro deploy

O primeiro deploy será feito pelo GitHub Actions após push na `main`:

1. Push na `main` aciona o workflow
2. Tests rodam automaticamente
3. Imagens são buildadas e enviadas ao Docker Hub
4. Deploy aguarda aprovação no GitHub (aba Actions → workflow run → Review deployments)
5. Após aprovação, a VPS faz pull das imagens e sobe os containers

## Quando tiver domínio

Basta atualizar no `.env` da VPS:

```bash
SERVER_NAME=seudominio.com
CADDY_MERCURE_PUBLIC_URL=https://seudominio.com/.well-known/mercure
CORS_ALLOW_ORIGIN='^https?://seudominio\.com$'
TRUSTED_HOSTS='^seudominio\.com|php$$'
```

O Caddy gera o certificado TLS automaticamente.
