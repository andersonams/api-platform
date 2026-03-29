# Tasks — CD Pipeline para VPS

## Tasks

- [x] **Task 1: Tornar ci.yml reutilizável**
  Adicionar `workflow_call` ao trigger do `ci.yml` existente para que possa ser chamado pelo workflow de deploy.

- [x] **Task 2: Ajustar compose.prod.yaml para produção**
  Substituir build contexts por imagens do Docker Hub (`andersonmouradev/api-php`, `andersonmouradev/api-pwa`). Configurar `SERVER_NAME=:80`, `restart: unless-stopped`, e garantir que o PostgreSQL não exponha porta externamente.

- [x] **Task 3: Criar workflow deploy.yml**
  Criar o workflow com 3 jobs: test (chama ci.yml), build-and-push (Docker Hub), deploy (SSH na VPS com environment `production`).

- [x] **Task 4: Documentar setup do ambiente**
  Adicionar instruções no proposal ou README sobre: secrets do GitHub, criação do environment `production`, setup da VPS (diretório, `.env`, SSH key, firewall).
