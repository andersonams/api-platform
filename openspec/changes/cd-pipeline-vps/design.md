# Design — CD Pipeline para VPS

## Arquivos a criar/modificar

### 1. `.github/workflows/deploy.yml` (novo)

Workflow principal com 3 jobs encadeados:

```yaml
on:
  push:
    branches: [main]

jobs:
  test:        # reutiliza ci.yml
  build:       # builda e pusha imagens
  deploy:      # SSH na VPS (requer aprovação)
```

**Job: test**
- Usa `workflow_call` para reutilizar o `ci.yml` existente
- Se falhar, pipeline para aqui

**Job: build-and-push**
- Depende de `test`
- Matrix strategy para buildar PHP e PWA em paralelo
- Docker Buildx com cache do GitHub Actions (gha)
- Login no Docker Hub via `docker/login-action`
- Push com tags: `latest` + `sha-<7 chars>`
- Usa `docker/metadata-action` para gerar tags

**Job: deploy**
- Depende de `build-and-push`
- Environment: `production` (gate de aprovação)
- SSH via `appleboy/ssh-action`
- Comandos na VPS:
  ```bash
  cd /opt/api-platform
  docker compose -f compose.prod.yaml pull
  docker compose -f compose.prod.yaml up -d --remove-orphans
  docker image prune -f
  ```

### 2. `compose.prod.yaml` (modificar)

Atualmente referencia build local. Precisa referenciar imagens do Docker Hub:

```yaml
services:
  php:
    image: andersonmouradev/api-php:latest
    # remove build context

  pwa:
    image: andersonmouradev/api-pwa:latest
    # remove build context

  database:
    image: postgres:16-alpine
    # mantém como está
```

Ajustes adicionais:
- `SERVER_NAME=:80` (HTTP, sem domínio)
- `restart: unless-stopped` em todos os serviços
- Variáveis sensíveis via `.env` na VPS (não commitado)

### 3. `.github/workflows/ci.yml` (modificar)

Tornar reutilizável adicionando trigger `workflow_call` para que o `deploy.yml` possa chamá-lo.

## Fluxo de dados na VPS

```
  /opt/api-platform/          ← diretório na VPS
  ├── compose.prod.yaml       ← vem do repo ou é copiado
  └── .env                    ← criado manualmente na VPS
       │
       ├── POSTGRES_PASSWORD=xxx
       ├── APP_SECRET=xxx
       ├── MERCURE_JWT_SECRET=xxx
       ├── CORS_ALLOW_ORIGIN=xxx
       └── SERVER_NAME=:80
```

## Segurança

- Secrets nunca são commitados — `.env` de produção fica só na VPS
- SSH key é específica para deploy (sem acesso root se possível)
- Docker Hub token com escopo mínimo (read/write apenas nos repos necessários)
- O `compose.prod.yaml` não expõe porta do PostgreSQL externamente
