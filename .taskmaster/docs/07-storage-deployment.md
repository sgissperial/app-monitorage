# 07 - Stockage et déploiement

## Vue d'ensemble

Ce document décrit le schéma de base de données PostgreSQL, la configuration des environnements de développement et de production, ainsi que les stratégies de déploiement.

---

## Base de données PostgreSQL

### Schéma de base de données

```
┌─────────────────────────────────────────────────────────────────┐
│                         SCHEMA: public                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐      ┌──────────────┐     ┌──────────────┐   │
│  │   endpoints  │      │  connectors  │     │ action_logs  │   │
│  ├──────────────┤      ├──────────────┤     ├──────────────┤   │
│  │ id           │      │ id           │     │ id           │   │
│  │ name         │      │ type         │     │ action       │   │
│  │ type         │      │ base_url     │     │ target       │   │
│  │ url          │      │ credentials  │     │ user_id      │   │
│  │ check_interval│     │ options      │     │ result       │   │
│  │ timeout      │      │ enabled      │     │ timestamp    │   │
│  │ expected_status│    │ created_at   │     │ ...          │   │
│  │ headers      │      │ updated_at   │     └──────────────┘   │
│  │ enabled      │      └──────────────┘                        │
│  │ created_at   │                                              │
│  │ updated_at   │      ┌──────────────┐                        │
│  └──────────────┘      │ health_history│                       │
│                        ├──────────────┤                        │
│                        │ id           │                        │
│                        │ endpoint_id  │──────┐                 │
│                        │ status       │      │                 │
│                        │ status_code  │      │ FK              │
│                        │ latency_ms   │      │                 │
│                        │ checked_at   │◀─────┘                 │
│                        └──────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Script de migration initial

```sql
-- migrations/001_initial_schema.sql

-- Extension pour UUID
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Enum pour les types d'endpoints
CREATE TYPE endpoint_type AS ENUM ('APIM', 'DOMAIN', 'TALEND');

-- Enum pour les types de connectors
CREATE TYPE connector_type AS ENUM (
  'TALEND', 'APIM', 'GITHUB', 'KEYCLOAK',
  'DOMAIN_HEALTH', 'CLOUDFLARE', 'GLPI', 'JIRA'
);

-- Enum pour les statuts de santé
CREATE TYPE health_status AS ENUM ('UP', 'DOWN', 'DEGRADED', 'UNKNOWN');

-- Enum pour les résultats d'actions
CREATE TYPE action_result AS ENUM ('SUCCESS', 'FAILURE');

-- Table des endpoints monitorés
CREATE TABLE endpoints (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(100) NOT NULL,
  type endpoint_type NOT NULL,
  url VARCHAR(500) NOT NULL,
  check_interval INTEGER NOT NULL DEFAULT 60,
  timeout INTEGER NOT NULL DEFAULT 5000,
  expected_status INTEGER NOT NULL DEFAULT 200,
  headers JSONB DEFAULT '{}',
  enabled BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Index sur les endpoints
CREATE INDEX idx_endpoints_type ON endpoints(type);
CREATE INDEX idx_endpoints_enabled ON endpoints(enabled);

-- Table des connectors
CREATE TABLE connectors (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  type connector_type NOT NULL UNIQUE,
  base_url VARCHAR(500),
  credentials TEXT, -- Chiffré (AES-256)
  options JSONB DEFAULT '{}',
  enabled BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Index sur les connectors
CREATE INDEX idx_connectors_type ON connectors(type);
CREATE INDEX idx_connectors_enabled ON connectors(enabled);

-- Table de l'historique des checks
CREATE TABLE health_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  endpoint_id UUID NOT NULL REFERENCES endpoints(id) ON DELETE CASCADE,
  status health_status NOT NULL,
  status_code INTEGER,
  latency_ms INTEGER,
  error_message TEXT,
  checked_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Index sur l'historique
CREATE INDEX idx_health_history_endpoint_id ON health_history(endpoint_id);
CREATE INDEX idx_health_history_checked_at ON health_history(checked_at DESC);

-- Partition par mois pour l'historique (optionnel, pour gros volumes)
-- CREATE TABLE health_history_2024_01 PARTITION OF health_history
--   FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Table des logs d'actions
CREATE TABLE action_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  action VARCHAR(50) NOT NULL,
  target VARCHAR(100) NOT NULL,
  user_id VARCHAR(100) NOT NULL,
  user_email VARCHAR(255) NOT NULL,
  user_roles JSONB DEFAULT '[]',
  ip_address VARCHAR(45),
  user_agent TEXT,
  parameters JSONB DEFAULT '{}',
  result action_result NOT NULL,
  error_message TEXT,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Index sur les logs
CREATE INDEX idx_action_logs_user_id ON action_logs(user_id);
CREATE INDEX idx_action_logs_action ON action_logs(action);
CREATE INDEX idx_action_logs_created_at ON action_logs(created_at DESC);

-- Trigger pour mise à jour automatique de updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_endpoints_updated_at
  BEFORE UPDATE ON endpoints
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_connectors_updated_at
  BEFORE UPDATE ON connectors
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Vue pour le statut actuel des endpoints
CREATE VIEW current_endpoint_status AS
SELECT DISTINCT ON (e.id)
  e.id,
  e.name,
  e.type,
  e.url,
  e.enabled,
  h.status,
  h.status_code,
  h.latency_ms,
  h.checked_at AS last_check
FROM endpoints e
LEFT JOIN health_history h ON e.id = h.endpoint_id
ORDER BY e.id, h.checked_at DESC;
```

### Données initiales

```sql
-- migrations/002_seed_connectors.sql

-- Insertion des connectors avec configuration par défaut
INSERT INTO connectors (type, base_url, enabled) VALUES
  ('TALEND', NULL, false),
  ('APIM', NULL, false),
  ('GITHUB', 'https://api.github.com', false),
  ('KEYCLOAK', NULL, false),
  ('DOMAIN_HEALTH', NULL, true),
  ('CLOUDFLARE', 'https://api.cloudflare.com/client/v4', false),
  ('GLPI', NULL, false),
  ('JIRA', NULL, false);
```

---

## Environnement de développement local

### docker-compose.yml

```yaml
version: '3.8'

services:
  # Base de données PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: monitorage-db
    environment:
      POSTGRES_USER: ${DB_USER:-monitorage}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-monitorage_dev}
      POSTGRES_DB: ${DB_NAME:-monitorage}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./packages/backend/migrations:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-monitorage}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Backend Node.js
  backend:
    build:
      context: ./packages/backend
      dockerfile: Dockerfile.dev
    container_name: monitorage-backend
    environment:
      NODE_ENV: development
      PORT: 3000
      DATABASE_URL: postgresql://${DB_USER:-monitorage}:${DB_PASSWORD:-monitorage_dev}@postgres:5432/${DB_NAME:-monitorage}
      AZURE_TENANT_ID: ${AZURE_TENANT_ID}
      AZURE_CLIENT_ID: ${AZURE_CLIENT_ID}
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
      # Connectors (optionnel en dev)
      TALEND_API_KEY: ${TALEND_API_KEY:-}
      GITHUB_PAT: ${GITHUB_PAT:-}
      KEYCLOAK_URL: ${KEYCLOAK_URL:-}
      CLOUDFLARE_API_KEY: ${CLOUDFLARE_API_KEY:-}
      GLPI_API_KEY: ${GLPI_API_KEY:-}
      JIRA_API_KEY: ${JIRA_API_KEY:-}
    ports:
      - "3000:3000"
    volumes:
      - ./packages/backend/src:/app/src
      - ./packages/shared:/app/packages/shared
    depends_on:
      postgres:
        condition: service_healthy
    command: npm run dev

  # Frontend React
  frontend:
    build:
      context: ./packages/frontend
      dockerfile: Dockerfile.dev
    container_name: monitorage-frontend
    environment:
      VITE_API_BASE_URL: http://localhost:3000/api/v1
      VITE_AZURE_TENANT_ID: ${AZURE_TENANT_ID}
      VITE_AZURE_CLIENT_ID: ${AZURE_CLIENT_ID_FRONTEND}
      VITE_REDIRECT_URI: http://localhost:5173/callback
      VITE_POST_LOGOUT_URI: http://localhost:5173
    ports:
      - "5173:5173"
    volumes:
      - ./packages/frontend/src:/app/src
      - ./packages/shared:/app/packages/shared
    depends_on:
      - backend
    command: npm run dev -- --host

volumes:
  postgres_data:

networks:
  default:
    name: monitorage-network
```

### Dockerfile Backend (dev)

```dockerfile
# packages/backend/Dockerfile.dev

FROM node:22-alpine

WORKDIR /app

# Copie des fichiers de dépendances
COPY package*.json ./
COPY tsconfig.json ./

# Installation des dépendances
RUN npm ci

# Copie du code source
COPY src ./src

# Exposition du port
EXPOSE 3000

# Commande par défaut (sera overridée par docker-compose)
CMD ["npm", "run", "dev"]
```

### Dockerfile Frontend (dev)

```dockerfile
# packages/frontend/Dockerfile.dev

FROM node:22-alpine

WORKDIR /app

# Copie des fichiers de dépendances
COPY package*.json ./
COPY vite.config.ts ./
COPY tsconfig*.json ./
COPY index.html ./

# Installation des dépendances
RUN npm ci

# Copie du code source
COPY src ./src
COPY public ./public

# Exposition du port
EXPOSE 5173

# Commande par défaut
CMD ["npm", "run", "dev", "--", "--host"]
```

### Fichier .env.example

```bash
# .env.example

# Base de données
DB_USER=monitorage
DB_PASSWORD=change_me_in_production
DB_NAME=monitorage

# Azure AD
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID_FRONTEND=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Chiffrement
ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef

# Connectors (optionnel en dev)
TALEND_API_KEY=
TALEND_BASE_URL=
GITHUB_PAT=
GITHUB_OWNER=
GITHUB_REPO=
KEYCLOAK_URL=
KEYCLOAK_REALM=
KEYCLOAK_CLIENT_ID=
KEYCLOAK_CLIENT_SECRET=
CLOUDFLARE_API_KEY=
CLOUDFLARE_ZONE_ID=
GLPI_API_KEY=
GLPI_BASE_URL=
JIRA_API_KEY=
JIRA_BASE_URL=
JIRA_PROJECT_KEY=
JIRA_BOARD_ID=
```

---

## Environnement de production (Azure)

### Architecture Azure

```
┌─────────────────────────────────────────────────────────────────┐
│                         AZURE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Azure Front Door                         │   │
│  │            (CDN + WAF + Load Balancing)                 │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                       │
│           ┌─────────────┴─────────────┐                        │
│           │                           │                        │
│           ▼                           ▼                        │
│  ┌─────────────────┐         ┌─────────────────┐              │
│  │  Storage Account│         │  Container Apps │              │
│  │  Static Website │         │    (Backend)    │              │
│  │   (Frontend)    │         │                 │              │
│  └─────────────────┘         └────────┬────────┘              │
│                                       │                        │
│                                       ▼                        │
│                              ┌─────────────────┐              │
│                              │   PostgreSQL    │              │
│                              │ Flexible Server │              │
│                              └─────────────────┘              │
│                                                                 │
│  ┌─────────────────┐         ┌─────────────────┐              │
│  │   Key Vault     │         │  App Insights   │              │
│  │   (Secrets)     │         │  (Monitoring)   │              │
│  └─────────────────┘         └─────────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Ressources Azure

| Ressource | SKU | Description |
|-----------|-----|-------------|
| Container Apps | Consumption | Backend Node.js |
| Storage Account | Standard_LRS | Frontend static files |
| PostgreSQL Flexible Server | Burstable B1ms | Base de données |
| Key Vault | Standard | Stockage des secrets |
| Application Insights | - | Monitoring et logs |
| Azure Front Door | Standard | CDN, WAF, routing |

### Dockerfile Backend (production)

```dockerfile
# packages/backend/Dockerfile

FROM node:22-alpine AS builder

WORKDIR /app

# Copie des fichiers de dépendances
COPY package*.json ./
COPY tsconfig.json ./

# Installation des dépendances
RUN npm ci

# Copie du code source
COPY src ./src

# Build TypeScript
RUN npm run build

# Stage de production
FROM node:22-alpine AS production

WORKDIR /app

# Copie des dépendances de production uniquement
COPY package*.json ./
RUN npm ci --only=production

# Copie du build
COPY --from=builder /app/dist ./dist

# User non-root
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Exposition du port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Commande de démarrage
CMD ["node", "dist/index.js"]
```

### Configuration Container Apps

```yaml
# azure/container-apps.yaml

containerApp:
  name: monitorage-backend
  location: westeurope
  managedEnvironmentId: /subscriptions/.../managedEnvironments/monitorage-env

  template:
    containers:
      - name: backend
        image: monitorageacr.azurecr.io/monitorage-backend:latest
        resources:
          cpu: 0.5
          memory: 1Gi
        env:
          - name: NODE_ENV
            value: production
          - name: PORT
            value: "3000"
          - name: DATABASE_URL
            secretRef: database-url
          - name: AZURE_TENANT_ID
            secretRef: azure-tenant-id
          - name: AZURE_CLIENT_ID
            secretRef: azure-client-id
          - name: ENCRYPTION_KEY
            secretRef: encryption-key

    scale:
      minReplicas: 1
      maxReplicas: 3
      rules:
        - name: http-requests
          http:
            metadata:
              concurrentRequests: "100"

  configuration:
    ingress:
      external: true
      targetPort: 3000
      transport: http
      allowInsecure: false

    secrets:
      - name: database-url
        keyVaultUrl: https://monitorage-kv.vault.azure.net/secrets/database-url
      - name: azure-tenant-id
        keyVaultUrl: https://monitorage-kv.vault.azure.net/secrets/azure-tenant-id
      - name: azure-client-id
        keyVaultUrl: https://monitorage-kv.vault.azure.net/secrets/azure-client-id
      - name: encryption-key
        keyVaultUrl: https://monitorage-kv.vault.azure.net/secrets/encryption-key
```

### Déploiement Frontend (Static Website)

```yaml
# azure/storage-static-website.yaml

# Configuration via Azure CLI
# az storage blob service-properties update --account-name monitoragefrontend --static-website --index-document index.html --404-document index.html

# Déploiement via GitHub Actions
# az storage blob upload-batch --account-name monitoragefrontend --source ./dist --destination '$web' --overwrite
```

---

## CI/CD (GitHub Actions)

### Workflow de déploiement

```yaml
# .github/workflows/deploy.yml

name: Deploy to Azure

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environnement cible'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: monitorageacr.azurecr.io

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and push backend image
        run: |
          docker build -t ${{ env.REGISTRY }}/monitorage-backend:${{ github.sha }} ./packages/backend
          docker push ${{ env.REGISTRY }}/monitorage-backend:${{ github.sha }}
          docker tag ${{ env.REGISTRY }}/monitorage-backend:${{ github.sha }} ${{ env.REGISTRY }}/monitorage-backend:latest
          docker push ${{ env.REGISTRY }}/monitorage-backend:latest

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci
        working-directory: ./packages/frontend

      - name: Build frontend
        run: npm run build
        working-directory: ./packages/frontend
        env:
          VITE_API_BASE_URL: ${{ secrets.API_BASE_URL }}
          VITE_AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          VITE_AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID_FRONTEND }}

      - name: Upload frontend artifact
        uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: ./packages/frontend/dist

  deploy-backend:
    needs: build-backend
    runs-on: ubuntu-latest
    steps:
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          containerAppName: monitorage-backend
          resourceGroup: monitorage-rg
          imageToDeploy: ${{ env.REGISTRY }}/monitorage-backend:${{ github.sha }}

  deploy-frontend:
    needs: build-frontend
    runs-on: ubuntu-latest
    steps:
      - name: Download frontend artifact
        uses: actions/download-artifact@v4
        with:
          name: frontend-dist
          path: ./dist

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Storage Static Website
        run: |
          az storage blob upload-batch \
            --account-name monitoragefrontend \
            --source ./dist \
            --destination '$web' \
            --overwrite

      - name: Purge CDN cache
        run: |
          az afd endpoint purge \
            --resource-group monitorage-rg \
            --profile-name monitorage-cdn \
            --endpoint-name monitorage-frontend \
            --content-paths "/*"
```

---

## Backup et reprise d'activité

### Backup PostgreSQL

- **Backup automatique** : Activé via Azure (7-35 jours de rétention)
- **Point-in-time restore** : Disponible
- **Geo-redundant backup** : Optionnel (recommandé en production)

### Stratégie de reprise

| RTO cible | RPO cible | Stratégie |
|-----------|-----------|-----------|
| < 1h | < 5min | Backup automatique + PITR |

---

## Documents connexes

- [04 - Architecture DDD](./04-architecture-ddd.md)
- [06 - Sécurité et authentification](./06-security-auth.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
