# 05 - Conception API

## Vue d'ensemble

L'API REST est documentée via **Swagger / OpenAPI 3.0** et permet la génération automatique d'un client TypeScript pour le frontend React.

---

## Conventions API

### Base URL

```
Production : https://api.monitorage.example.com/api/v1
Développement : http://localhost:3000/api/v1
```

### Format des réponses

Toutes les réponses suivent un format standardisé :

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

En cas d'erreur :

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

**Réponse :**
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

Récupère les détails d'un endpoint spécifique.

**Paramètres :**
- `id` (path) : UUID de l'endpoint

**Réponse :**
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

**Réponse :**
```json
{
  "success": true,
  "data": {
    "status": "UP",
    "statusCode": 200,
    "latencyMs": 52,
    "checkedAt": "2024-01-15T10:31:00Z"
  }
}
```

---

### Administration des endpoints

#### GET /api/v1/admin/endpoints

Liste tous les endpoints configurés (admin only).

**Query params :**
- `type` (optional) : Filtrer par type (APIM, DOMAIN, TALEND)
- `enabled` (optional) : Filtrer par statut actif

**Réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "API Users",
      "type": "APIM",
      "url": "https://api.example.com/health",
      "checkInterval": 60,
      "timeout": 5000,
      "expectedStatus": 200,
      "headers": {},
      "enabled": true,
      "createdAt": "2024-01-10T08:00:00Z",
      "updatedAt": "2024-01-15T09:00:00Z"
    }
  ]
}
```

#### POST /api/v1/admin/endpoints

Crée un nouvel endpoint (admin only).

**Body :**
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

**Réponse (201) :**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "API Orders",
    ...
  }
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

**Réponse :**
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

Liste les workflows disponibles.

**Réponse :**
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

**Body :**
```json
{
  "ref": "main",
  "inputs": {
    "environment": "production"
  }
}
```

**Réponse :**
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

**Body (optionnel) :**
```json
{
  "realm": "master"
}
```

**Réponse :**
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

**Réponse :**
```json
{
  "success": true,
  "data": [
    {
      "id": "zone-abc123",
      "name": "example.com",
      "status": "active"
    }
  ]
}
```

#### POST /api/v1/cloudflare/zones/{zoneId}/purge

Purge le cache d'une zone Cloudflare.

**Body :**
```json
{
  "purgeAll": true
}
```

ou

```json
{
  "files": [
    "https://example.com/styles.css",
    "https://example.com/script.js"
  ]
}
```

**Réponse :**
```json
{
  "success": true,
  "data": {
    "purged": true,
    "zoneId": "zone-abc123",
    "timestamp": "2024-01-15T10:36:00Z"
  }
}
```

---

### Tickets - GLPI

#### GET /api/v1/glpi/tickets

Liste les tickets GLPI de l'utilisateur.

**Query params :**
- `status` (optional) : Filtrer par statut (new, assigned, planned, pending, solved, closed)
- `limit` (optional) : Nombre max de résultats (défaut: 50)

**Réponse :**
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

**Réponse :**
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

Liste les tickets du backlog (hors sprint).

**Réponse :** Structure identique sans l'objet sprint.

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

**Réponse :**
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

**Réponse :**
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

### Fichier openapi.yaml (extrait)

```yaml
openapi: 3.0.3
info:
  title: App Monitorage API
  description: API de monitoring et d'actions opérationnelles
  version: 1.0.0
  contact:
    name: Équipe DevOps
    email: devops@example.com

servers:
  - url: http://localhost:3000/api/v1
    description: Développement local
  - url: https://api.monitorage.example.com/api/v1
    description: Production

tags:
  - name: monitoring
    description: Endpoints de monitoring
  - name: admin
    description: Administration (requiert rôle admin)
  - name: talend
    description: Intégration Talend
  - name: github
    description: Intégration GitHub Actions
  - name: keycloak
    description: Intégration Keycloak
  - name: cloudflare
    description: Intégration Cloudflare
  - name: glpi
    description: Intégration GLPI
  - name: jira
    description: Intégration Jira
  - name: auth
    description: Authentification

paths:
  /monitoring/status:
    get:
      tags:
        - monitoring
      summary: Statut global du monitoring
      operationId: getMonitoringStatus
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Succès
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MonitoringStatusResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /admin/endpoints:
    get:
      tags:
        - admin
      summary: Liste des endpoints configurés
      operationId: getAdminEndpoints
      security:
        - bearerAuth: []
      parameters:
        - name: type
          in: query
          schema:
            $ref: '#/components/schemas/EndpointType'
        - name: enabled
          in: query
          schema:
            type: boolean
      responses:
        '200':
          description: Succès
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EndpointListResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'

    post:
      tags:
        - admin
      summary: Créer un endpoint
      operationId: createEndpoint
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateEndpointRequest'
      responses:
        '201':
          description: Endpoint créé
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EndpointResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    EndpointType:
      type: string
      enum:
        - APIM
        - DOMAIN
        - TALEND

    EndpointStatus:
      type: string
      enum:
        - UP
        - DOWN
        - DEGRADED
        - UNKNOWN

    Endpoint:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        type:
          $ref: '#/components/schemas/EndpointType'
        url:
          type: string
          format: uri
        status:
          $ref: '#/components/schemas/EndpointStatus'
        statusCode:
          type: integer
        latencyMs:
          type: integer
        lastCheck:
          type: string
          format: date-time
        enabled:
          type: boolean

    CreateEndpointRequest:
      type: object
      required:
        - name
        - type
        - url
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        type:
          $ref: '#/components/schemas/EndpointType'
        url:
          type: string
          format: uri
        checkInterval:
          type: integer
          minimum: 10
          default: 60
        timeout:
          type: integer
          minimum: 1000
          default: 5000
        expectedStatus:
          type: integer
          default: 200
        headers:
          type: object
          additionalProperties:
            type: string
        enabled:
          type: boolean
          default: true

    ApiResponse:
      type: object
      properties:
        success:
          type: boolean
        meta:
          type: object
          properties:
            timestamp:
              type: string
              format: date-time
            requestId:
              type: string
              format: uuid

    ApiError:
      type: object
      properties:
        success:
          type: boolean
          enum: [false]
        error:
          type: object
          properties:
            code:
              type: string
            message:
              type: string
            details:
              type: array
              items:
                type: object

  responses:
    BadRequest:
      description: Requête invalide
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ApiError'
    Unauthorized:
      description: Non authentifié
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ApiError'
    Forbidden:
      description: Non autorisé
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ApiError'
    NotFound:
      description: Ressource non trouvée
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ApiError'
```

---

## Génération du client React

### Configuration

Le client TypeScript est généré automatiquement à partir du fichier OpenAPI via **openapi-typescript-codegen** ou **orval**.

```bash
# Exemple avec openapi-typescript-codegen
npx openapi-typescript-codegen \
  --input ./swagger/openapi.yaml \
  --output ./src/infrastructure/api/generated \
  --client axios
```

### Utilisation dans le frontend

```typescript
// src/infrastructure/api/monitoringApi.ts
import { MonitoringService } from './generated';

export const monitoringApi = {
  getStatus: () => MonitoringService.getMonitoringStatus(),
  getEndpointDetails: (id: string) => MonitoringService.getEndpointById(id),
  forceCheck: (id: string) => MonitoringService.checkEndpoint(id),
};
```

---

## Documents connexes

- [04 - Architecture DDD](./04-architecture-ddd.md)
- [06 - Sécurité et authentification](./06-security-auth.md)
