# 05 - Conception API

## Vue d'ensemble

L'API REST est documentée via **Swagger / OpenAPI 3.0**, **généré automatiquement depuis le code** backend (TypeScript + Express). Cette documentation permet la génération automatique d'un client TypeScript pour le frontend React.

---

## Stratégie de génération OpenAPI

### Approche code-first

La documentation OpenAPI est **générée à partir du code backend**, et non écrite manuellement. Cette approche garantit que la documentation est toujours synchronisée avec l'implémentation.

**Outils recommandés pour Express + TypeScript :**
- **tsoa** : Génération à partir de décorateurs TypeScript
- **swagger-jsdoc** : Génération à partir de commentaires JSDoc
- **routing-controllers + class-validator** : Décorateurs avec validation intégrée

### Workflow de génération

1. Le développeur ajoute des décorateurs/annotations aux controllers et DTOs
2. Un script de build génère le fichier `openapi.json` ou `openapi.yaml`
3. Le fichier OpenAPI est exposé via un endpoint `/api-docs`
4. Un script frontend génère le client TypeScript à partir de l'OpenAPI

---

## Conventions API

### Base URL

| Environnement | URL |
|---------------|-----|
| Production | `https://api.monitorage.example.com/api/v1` |
| Développement | `http://localhost:3000/api/v1` |

### Format des réponses

**Réponse succès :**
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "uuid"
  }
}
```

**Réponse erreur :**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Description de l'erreur",
    "details": [ ... ]
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "uuid"
  }
}
```

### Codes HTTP

| Code | Utilisation |
|------|-------------|
| 200 | Succès (GET, PUT, PATCH) |
| 201 | Création réussie (POST) |
| 204 | Suppression réussie (DELETE) |
| 400 | Erreur de validation |
| 401 | Non authentifié |
| 403 | Non autorisé |
| 404 | Ressource non trouvée |
| 500 | Erreur serveur |

---

## Endpoints API

### Monitoring

#### GET /api/v1/monitoring/status

Récupère le statut global de tous les endpoints monitorés.

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "summary": {
      "total": 15,
      "up": 12,
      "down": 2,
      "degraded": 1
    },
    "endpoints": [
      {
        "id": "uuid",
        "name": "API Users",
        "type": "APIM",
        "url": "https://api.example.com/health",
        "status": "UP",
        "statusCode": 200,
        "latencyMs": 45,
        "lastCheck": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

#### GET /api/v1/monitoring/endpoints/{id}

Récupère les détails d'un endpoint spécifique avec son historique.

**Paramètres :**
- `id` (path) : UUID de l'endpoint

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "API Users",
    "type": "APIM",
    "url": "https://api.example.com/health",
    "checkInterval": 60,
    "timeout": 5000,
    "expectedStatus": 200,
    "enabled": true,
    "history": [
      {
        "status": "UP",
        "statusCode": 200,
        "latencyMs": 45,
        "checkedAt": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

#### POST /api/v1/monitoring/endpoints/{id}/check

Force un check immédiat d'un endpoint.

---

### Administration des endpoints

#### GET /api/v1/admin/endpoints

Liste tous les endpoints configurés (admin only).

**Query params :**
- `type` (optional) : Filtrer par type (APIM, DOMAIN, TALEND)
- `enabled` (optional) : Filtrer par statut actif

#### POST /api/v1/admin/endpoints

Crée un nouvel endpoint (admin only).

**Exemple de body :**
```json
{
  "name": "API Orders",
  "type": "APIM",
  "url": "https://api.example.com/orders/health",
  "checkInterval": 60,
  "timeout": 5000,
  "expectedStatus": 200,
  "headers": {
    "Ocp-Apim-Subscription-Key": "***"
  },
  "enabled": true
}
```

#### PUT /api/v1/admin/endpoints/{id}

Met à jour un endpoint (admin only).

#### DELETE /api/v1/admin/endpoints/{id}

Supprime un endpoint (admin only).

#### POST /api/v1/admin/endpoints/{id}/test

Teste la connectivité d'un endpoint sans le sauvegarder.

---

### Connectors - Talend

#### GET /api/v1/talend/jobs/errors

Liste les jobs Talend en erreur.

**Exemple de réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": "job-123",
      "name": "ETL_Users_Sync",
      "environment": "PROD",
      "errorDate": "2024-01-15T08:45:00Z",
      "errorMessage": "Connection timeout",
      "consoleUrl": "https://tmc.example.com/jobs/job-123"
    }
  ]
}
```

---

### Connectors - GitHub Actions

#### GET /api/v1/github/workflows

Liste les workflows disponibles (uniquement ceux avec `workflow_dispatch`).

**Exemple de réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": 12345,
      "name": "Deploy Production",
      "repository": "org/app-monitorage",
      "path": ".github/workflows/deploy.yml",
      "lastRun": {
        "id": 67890,
        "status": "success",
        "conclusion": "success",
        "startedAt": "2024-01-15T09:00:00Z",
        "completedAt": "2024-01-15T09:05:00Z",
        "url": "https://github.com/org/app-monitorage/actions/runs/67890"
      },
      "inputs": [
        {
          "name": "environment",
          "type": "choice",
          "required": true,
          "options": ["staging", "production"]
        }
      ]
    }
  ]
}
```

#### POST /api/v1/github/workflows/{id}/dispatch

Déclenche un workflow.

**Exemple de body :**
```json
{
  "ref": "main",
  "inputs": {
    "environment": "production"
  }
}
```

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "triggered": true,
    "message": "Workflow déclenché avec succès",
    "runUrl": "https://github.com/org/app-monitorage/actions/runs/67891"
  }
}
```

---

### Connectors - Keycloak

#### POST /api/v1/keycloak/cache/purge

Purge le cache Keycloak.

**Exemple de body (optionnel) :**
```json
{
  "realm": "master"
}
```

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "purged": true,
    "realm": "master",
    "timestamp": "2024-01-15T10:35:00Z"
  }
}
```

---

### Connectors - Cloudflare

#### GET /api/v1/cloudflare/zones

Liste les zones Cloudflare configurées.

#### POST /api/v1/cloudflare/zones/{zoneId}/purge

Purge le cache d'une zone Cloudflare.

**Exemple de body (purge complète) :**
```json
{
  "purgeAll": true
}
```

**Exemple de body (purge sélective) :**
```json
{
  "files": [
    "https://example.com/styles.css",
    "https://example.com/script.js"
  ]
}
```

---

### Tickets - GLPI

#### GET /api/v1/glpi/tickets

Liste les tickets GLPI de l'utilisateur.

**Query params :**
- `status` (optional) : Filtrer par statut (new, assigned, planned, pending, solved, closed)
- `limit` (optional) : Nombre max de résultats (défaut: 50)

**Exemple de réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": 1234,
      "title": "Problème de connexion VPN",
      "status": "assigned",
      "priority": 3,
      "createdAt": "2024-01-14T14:00:00Z",
      "updatedAt": "2024-01-15T08:00:00Z",
      "url": "https://glpi.example.com/front/ticket.form.php?id=1234"
    }
  ]
}
```

---

### Tickets - Jira

#### GET /api/v1/jira/tickets/sprint

Liste les tickets du sprint actif.

**Query params :**
- `projectKey` (optional) : Clé du projet (défaut: configuré)
- `boardId` (optional) : ID du board (défaut: configuré)

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "sprint": {
      "id": 42,
      "name": "Sprint 15",
      "state": "active",
      "startDate": "2024-01-08T00:00:00Z",
      "endDate": "2024-01-22T00:00:00Z"
    },
    "tickets": [
      {
        "key": "PROJ-123",
        "summary": "Implémenter la page de monitoring",
        "status": "In Progress",
        "assignee": {
          "displayName": "John Doe",
          "avatarUrl": "https://..."
        },
        "type": "Story",
        "priority": "High",
        "url": "https://jira.example.com/browse/PROJ-123"
      }
    ]
  }
}
```

#### GET /api/v1/jira/tickets/backlog

Liste les tickets du backlog (hors sprint). Structure identique sans l'objet sprint.

---

### Actions - Logs

#### GET /api/v1/actions/logs

Liste les logs d'actions (admin only).

**Query params :**
- `type` (optional) : Type d'action
- `userId` (optional) : Filtrer par utilisateur
- `from` (optional) : Date de début
- `to` (optional) : Date de fin
- `limit` (optional) : Nombre max (défaut: 100)

**Exemple de réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "action": "CACHE_PURGE",
      "target": "keycloak",
      "userId": "user-uuid",
      "userEmail": "user@example.com",
      "result": "SUCCESS",
      "details": {
        "realm": "master"
      },
      "timestamp": "2024-01-15T10:35:00Z"
    }
  ]
}
```

---

### Authentification

#### GET /api/v1/auth/me

Récupère les informations de l'utilisateur connecté.

**Exemple de réponse :**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "John Doe",
    "roles": ["user", "admin"],
    "permissions": [
      "monitoring:read",
      "actions:execute",
      "admin:endpoints"
    ]
  }
}
```

---

## Spécification OpenAPI

### Structure attendue

La documentation OpenAPI générée doit inclure :

**Informations générales :**
- Titre : App Monitorage API
- Version : 1.0.0
- Servers : localhost (dev), production URL

**Tags (groupes d'endpoints) :**
- monitoring : Endpoints de monitoring
- admin : Administration (requiert rôle admin)
- talend : Intégration Talend
- github : Intégration GitHub Actions
- keycloak : Intégration Keycloak
- cloudflare : Intégration Cloudflare
- glpi : Intégration GLPI
- jira : Intégration Jira
- auth : Authentification

**Schémas de sécurité :**
- bearerAuth : JWT Bearer token

**Schémas de données principaux :**

| Schéma | Description |
|--------|-------------|
| EndpointType | Enum : APIM, DOMAIN, TALEND |
| EndpointStatus | Enum : UP, DOWN, DEGRADED, UNKNOWN |
| Endpoint | Objet endpoint complet |
| CreateEndpointRequest | Payload création endpoint |
| ApiResponse | Enveloppe réponse succès |
| ApiError | Enveloppe réponse erreur |

**Réponses communes :**
- BadRequest (400)
- Unauthorized (401)
- Forbidden (403)
- NotFound (404)

---

## Génération du client React

### Stratégie

Le client TypeScript frontend est **généré automatiquement** à partir du fichier OpenAPI exposé par le backend.

**Outils recommandés :**
- **openapi-typescript-codegen** : Génération de client Axios/Fetch
- **orval** : Génération avec React Query intégré
- **openapi-generator** : Génération multi-langages

### Workflow de génération

1. Le backend expose `/api-docs` avec le fichier OpenAPI
2. Un script npm dans le frontend récupère et génère le client
3. Le client est utilisé dans la couche `infrastructure/api`
4. Les types sont automatiquement synchronisés avec le backend

### Intégration dans le frontend

Le client généré est encapsulé dans des services métier (domain services) qui ajoutent :
- Gestion des erreurs applicatives
- Transformation des données
- Cache et optimistic updates si nécessaire

---

## Documents connexes

- [04 - Architecture DDD](./04-architecture-ddd.md)
- [06 - Sécurité et authentification](./06-security-auth.md)
