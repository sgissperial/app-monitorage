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

### Tables et colonnes

#### Table : endpoints

Stocke les endpoints à monitorer.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | UUID | PK, auto-generated | Identifiant unique |
| name | VARCHAR(100) | NOT NULL | Nom d'affichage |
| type | ENUM | NOT NULL | Type : APIM, DOMAIN, TALEND |
| url | VARCHAR(500) | NOT NULL | URL du healthcheck |
| check_interval | INTEGER | NOT NULL, default 60 | Intervalle en secondes |
| timeout | INTEGER | NOT NULL, default 5000 | Timeout en ms |
| expected_status | INTEGER | NOT NULL, default 200 | Code HTTP attendu |
| headers | JSONB | default '{}' | Headers personnalisés |
| enabled | BOOLEAN | NOT NULL, default true | Actif/inactif |
| created_at | TIMESTAMP WITH TZ | NOT NULL, auto | Date de création |
| updated_at | TIMESTAMP WITH TZ | NOT NULL, auto | Date de modification |

**Index :** type, enabled

#### Table : connectors

Configuration des connectors externes.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | UUID | PK, auto-generated | Identifiant unique |
| type | ENUM | NOT NULL, UNIQUE | Type de connector |
| base_url | VARCHAR(500) | nullable | URL de base de l'API |
| credentials | TEXT | nullable, chiffré | Credentials (AES-256) |
| options | JSONB | default '{}' | Options spécifiques |
| enabled | BOOLEAN | NOT NULL, default true | Actif/inactif |
| created_at | TIMESTAMP WITH TZ | NOT NULL, auto | Date de création |
| updated_at | TIMESTAMP WITH TZ | NOT NULL, auto | Date de modification |

**ConnectorType enum :** TALEND, APIM, GITHUB, KEYCLOAK, DOMAIN_HEALTH, CLOUDFLARE, GLPI, JIRA

**Index :** type, enabled

#### Table : health_history

Historique des checks de santé.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | UUID | PK, auto-generated | Identifiant unique |
| endpoint_id | UUID | FK → endpoints(id), CASCADE | Endpoint associé |
| status | ENUM | NOT NULL | UP, DOWN, DEGRADED, UNKNOWN |
| status_code | INTEGER | nullable | Code HTTP de la réponse |
| latency_ms | INTEGER | nullable | Temps de réponse |
| error_message | TEXT | nullable | Message d'erreur |
| checked_at | TIMESTAMP WITH TZ | NOT NULL, auto | Horodatage du check |

**Index :** endpoint_id, checked_at DESC

**Note :** Partitionnement par mois recommandé pour gros volumes.

#### Table : action_logs

Logs d'audit des actions utilisateur.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| id | UUID | PK, auto-generated | Identifiant unique |
| action | VARCHAR(50) | NOT NULL | Type d'action |
| target | VARCHAR(100) | NOT NULL | Cible de l'action |
| user_id | VARCHAR(100) | NOT NULL | ID utilisateur Azure AD |
| user_email | VARCHAR(255) | NOT NULL | Email utilisateur |
| user_roles | JSONB | default '[]' | Rôles de l'utilisateur |
| ip_address | VARCHAR(45) | nullable | Adresse IP |
| user_agent | TEXT | nullable | User-Agent |
| parameters | JSONB | default '{}' | Paramètres de l'action |
| result | ENUM | NOT NULL | SUCCESS, FAILURE |
| error_message | TEXT | nullable | Message d'erreur |
| created_at | TIMESTAMP WITH TZ | NOT NULL, auto | Horodatage |

**Index :** user_id, action, created_at DESC

### Mécanismes automatiques

**Triggers :**
- `updated_at` : Mise à jour automatique sur UPDATE pour endpoints et connectors

**Vue :** `current_endpoint_status`
- Agrège endpoints + dernier health_history
- Retourne : id, name, type, url, enabled, status, status_code, latency_ms, last_check

### Données initiales

Les connectors doivent être pré-insérés avec leur type et une configuration par défaut :
- Tous les types de connectors présents
- `enabled = false` par défaut (sauf DOMAIN_HEALTH)
- URLs de base prédéfinies pour GitHub et Cloudflare

---

## Environnement de développement local

### Architecture Docker Compose

**Services requis :**

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| postgres | postgres:16-alpine | 5432 | Base de données |
| backend | Custom (Dockerfile.dev) | 3000 | API Node.js + Express |
| frontend | Custom (Dockerfile.dev) | 5173 | React + Vite |

### Configuration PostgreSQL (dev)

| Variable | Valeur par défaut |
|----------|-------------------|
| POSTGRES_USER | monitorage |
| POSTGRES_PASSWORD | monitorage_dev |
| POSTGRES_DB | monitorage |

**Volumes :**
- Persistance des données PostgreSQL
- Montage des migrations pour initialisation

**Healthcheck :**
- Commande : `pg_isready`
- Intervalle : 10s, timeout : 5s, retries : 5

### Configuration Backend (dev)

**Variables d'environnement :**
| Variable | Description |
|----------|-------------|
| NODE_ENV | development |
| PORT | 3000 |
| DATABASE_URL | Connection string PostgreSQL |
| AZURE_TENANT_ID | ID tenant Azure AD |
| AZURE_CLIENT_ID | ID application backend |
| ENCRYPTION_KEY | Clé AES-256 |

**Volumes :**
- Hot reload du code source
- Accès au package shared

**Dépendances :**
- Attend que PostgreSQL soit healthy

### Configuration Frontend (dev)

**Variables d'environnement :**
| Variable | Description |
|----------|-------------|
| VITE_API_BASE_URL | http://localhost:3000/api/v1 |
| VITE_AZURE_TENANT_ID | ID tenant Azure AD |
| VITE_AZURE_CLIENT_ID | ID application frontend |
| VITE_REDIRECT_URI | http://localhost:5173/callback |
| VITE_POST_LOGOUT_URI | http://localhost:5173 |

### Dockerfile Backend (dev)

**Base :** node:22-alpine

**Étapes :**
1. Copie package*.json et tsconfig.json
2. Installation des dépendances (npm ci)
3. Copie du code source
4. Exposition port 3000
5. Commande : npm run dev (hot reload)

### Dockerfile Frontend (dev)

**Base :** node:22-alpine

**Étapes :**
1. Copie des fichiers de configuration (package.json, vite.config.ts, tsconfig*.json, index.html)
2. Installation des dépendances
3. Copie src et public
4. Exposition port 5173
5. Commande : npm run dev -- --host

### Variables d'environnement (.env.example)

| Catégorie | Variables |
|-----------|-----------|
| Base de données | DB_USER, DB_PASSWORD, DB_NAME |
| Azure AD | AZURE_TENANT_ID, AZURE_CLIENT_ID, AZURE_CLIENT_ID_FRONTEND |
| Sécurité | ENCRYPTION_KEY |
| Connectors | TALEND_API_KEY, TALEND_BASE_URL, GITHUB_PAT, GITHUB_OWNER, GITHUB_REPO, KEYCLOAK_URL, KEYCLOAK_REALM, KEYCLOAK_CLIENT_ID, KEYCLOAK_CLIENT_SECRET, CLOUDFLARE_API_KEY, CLOUDFLARE_ZONE_ID, GLPI_API_KEY, GLPI_BASE_URL, JIRA_API_KEY, JIRA_BASE_URL, JIRA_PROJECT_KEY, JIRA_BOARD_ID |

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

| Ressource | SKU recommandé | Description |
|-----------|----------------|-------------|
| Container Apps | Consumption | Backend Node.js + Express |
| Storage Account | Standard_LRS | Frontend static files |
| PostgreSQL Flexible Server | Burstable B1ms | Base de données |
| Key Vault | Standard | Stockage des secrets |
| Application Insights | - | Monitoring et logs |
| Azure Front Door | Standard | CDN, WAF, routing |

### Dockerfile Backend (production)

**Stage 1 - Builder :**
- Base : node:22-alpine
- Installation dépendances
- Build TypeScript

**Stage 2 - Production :**
- Base : node:22-alpine
- Dépendances production uniquement
- Copie du build depuis stage 1
- User non-root (nodejs:1001)
- Exposition port 3000
- Healthcheck : GET /health
- Commande : node dist/index.js

### Configuration Container Apps

**Ressources :**
- CPU : 0.5
- Memory : 1Gi

**Scaling :**
- Min replicas : 1
- Max replicas : 3
- Règle : 100 requêtes concurrentes

**Ingress :**
- External : true
- Target port : 3000
- Transport : HTTP
- Allow insecure : false

**Secrets :**
- Référencés depuis Key Vault
- Variables : DATABASE_URL, AZURE_TENANT_ID, AZURE_CLIENT_ID, ENCRYPTION_KEY

### Déploiement Frontend (Static Website)

**Configuration Storage Account :**
- Static website enabled
- Index document : index.html
- 404 document : index.html (pour SPA routing)

**Déploiement :**
- Upload batch vers container `$web`
- Overwrite enabled

---

## CI/CD (GitHub Actions)

### Pipeline de déploiement

**Déclencheurs :**
- Push sur `main`
- Workflow dispatch avec choix d'environnement (staging/production)

### Job : build-backend

**Étapes :**
1. Checkout du code
2. Login Azure Container Registry
3. Build image Docker avec tag SHA
4. Push image (SHA + latest)

### Job : build-frontend

**Étapes :**
1. Checkout du code
2. Setup Node.js 22
3. Install dependencies (npm ci)
4. Build avec variables d'environnement
5. Upload artifact

### Job : deploy-backend

**Prérequis :** build-backend

**Étapes :**
1. Login Azure
2. Deploy sur Container Apps avec nouvelle image

### Job : deploy-frontend

**Prérequis :** build-frontend

**Étapes :**
1. Download artifact
2. Login Azure
3. Upload vers Storage Static Website
4. Purge cache CDN

### Secrets GitHub requis

| Secret | Description |
|--------|-------------|
| ACR_USERNAME | Username Azure Container Registry |
| ACR_PASSWORD | Password Azure Container Registry |
| AZURE_CREDENTIALS | Service principal JSON |
| API_BASE_URL | URL API production |
| AZURE_TENANT_ID | ID tenant Azure AD |
| AZURE_CLIENT_ID_FRONTEND | ID application frontend |

---

## Backup et reprise d'activité

### Backup PostgreSQL (Azure)

| Fonctionnalité | Configuration |
|----------------|---------------|
| Backup automatique | Activé |
| Rétention | 7-35 jours |
| Point-in-time restore | Disponible |
| Geo-redundant backup | Optionnel (recommandé) |

### Stratégie de reprise

| Métrique | Cible | Stratégie |
|----------|-------|-----------|
| RTO | < 1h | Redéploiement Container Apps + restore DB |
| RPO | < 5min | Backup automatique + PITR |

### Procédure de restore

1. Créer un nouveau serveur PostgreSQL depuis backup
2. Mettre à jour la connection string dans Key Vault
3. Redémarrer Container Apps
4. Vérifier la connectivité

---

## Documents connexes

- [04 - Architecture DDD](./04-architecture-ddd.md)
- [06 - Sécurité et authentification](./06-security-auth.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
