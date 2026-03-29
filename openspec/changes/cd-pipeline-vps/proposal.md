# CD Pipeline — Deploy via Docker Hub para VPS

## Problema

O projeto possui CI funcional (testes, lint, health checks) mas não tem Continuous Delivery. O deploy para o VPS é manual e não padronizado.

## Proposta

Criar um workflow GitHub Actions que, após os testes passarem no push para `main`:

1. **Builda** as imagens de produção (PHP/FrankenPHP e PWA/Next.js)
2. **Pusha** para o Docker Hub (`andersonmouradev/api-php`, `andersonmouradev/api-pwa`)
3. **Deploya** na VPS via SSH, com **aprovação manual** antes do deploy

## Escopo

### O que inclui

- Workflow `deploy.yml` com 3 jobs: test → build & push → deploy
- Arquivo `compose.prod.yaml` ajustado para referenciar as imagens do Docker Hub
- Configuração do Caddy para HTTP (sem domínio por enquanto)
- Documentação dos secrets necessários no GitHub

### O que NÃO inclui

- HTTPS/TLS (sem domínio por enquanto — quando tiver, basta mudar `SERVER_NAME`)
- Kubernetes/Helm (deploy via Docker Compose direto)
- Monitoramento, alertas, logging centralizado
- Pipeline de staging/homologação
- Rollback automatizado

## Arquitetura

```
  Push main
      │
      ▼
  ┌─────────────────────────────────────────┐
  │  Job: test                              │
  │  (reutiliza ci.yml existente)           │
  └────────────┬────────────────────────────┘
               │ ✓
               ▼
  ┌─────────────────────────────────────────┐
  │  Job: build-and-push                    │
  │                                         │
  │  docker build (target: prod)            │
  │  docker push → Docker Hub               │
  │    andersonmouradev/api-php:latest      │
  │    andersonmouradev/api-php:<sha>       │
  │    andersonmouradev/api-pwa:latest      │
  │    andersonmouradev/api-pwa:<sha>       │
  └────────────┬────────────────────────────┘
               │
               ▼
  ┌─────────────────────────────────────────┐
  │  Job: deploy                            │
  │  environment: production (approval)     │
  │                                         │
  │  SSH → VPS:                             │
  │    docker compose pull                  │
  │    docker compose up -d                 │
  └─────────────────────────────────────────┘
```

## Secrets GitHub necessários

| Secret | Descrição |
|--------|-----------|
| `DOCKERHUB_USERNAME` | `andersonmouradev` |
| `DOCKERHUB_TOKEN` | Access token do Docker Hub |
| `VPS_HOST` | IP da VPS |
| `VPS_USER` | Usuário SSH na VPS |
| `VPS_SSH_KEY` | Chave privada SSH |

## Setup necessário no GitHub

- Criar **Environment** `production` com "Required reviewers" no repositório

## Setup necessário na VPS

- Docker e Docker Compose instalados
- Diretório do projeto com `compose.prod.yaml` e `.env` de produção
- Chave pública SSH do deploy adicionada em `~/.ssh/authorized_keys`
- Porta 80 liberada no firewall

## Decisões técnicas

- **Tags das imagens**: `latest` + short SHA do commit (permite rollback por tag)
- **Caddy em HTTP**: `SERVER_NAME=:80` (sem TLS até ter domínio)
- **Banco na mesma stack**: PostgreSQL roda no Compose junto com PHP e PWA
- **Reutiliza CI existente**: O job de test chama o `ci.yml` como reusable workflow
