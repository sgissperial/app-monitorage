# 04 - Architecture DDD

## Vue d'ensemble

L'application suit les principes du **Domain-Driven Design (DDD)** avec une architecture en couches distinctes, tant côté backend que frontend. Le backend utilise **Node.js 22** avec **TypeScript** et **Express**.

---

## Architecture globale

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                │
│              (React + Vite + TypeScript + MUI)                 │
├─────────────────────────────────────────────────────────────────┤
│                         API REST                                │
│              (Swagger / OpenAPI généré depuis le code)         │
├─────────────────────────────────────────────────────────────────┤
│                         BACKEND                                 │
│              (Node.js 22 + TypeScript + Express)               │
├─────────────────────────────────────────────────────────────────┤
│                       PostgreSQL                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │      SYSTÈMES EXTERNES        │
              │  Talend │ APIM │ GitHub │ ... │
              └───────────────────────────────┘
```

---

## Structure du mono-repo

```
app-monitorage/
├── packages/
│   ├── backend/                 # API Node.js + Express + TypeScript
│   │   ├── src/
│   │   │   ├── domain/          # Couche domaine
│   │   │   ├── application/     # Couche application (use cases)
│   │   │   ├── infrastructure/  # Couche infrastructure
│   │   │   └── presentation/    # Couche présentation (API Express)
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── frontend/                # Application React + TypeScript
│   │   ├── src/
│   │   │   ├── domain/          # Logique métier frontend
│   │   │   ├── application/     # Services et état
│   │   │   ├── infrastructure/  # API client (généré), storage
│   │   │   └── presentation/    # Composants UI
│   │   ├── package.json
│   │   └── vite.config.ts
│   │
│   └── shared/                  # Types et utilitaires partagés
│       ├── src/
│       │   ├── types/           # Types TypeScript partagés
│       │   └── utils/           # Utilitaires communs
│       └── package.json
│
├── docker-compose.yml
├── package.json                 # Workspace root
└── README.md
```

---

## Architecture Backend (DDD)

### Couches architecturales

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION                             │
│  Controllers │ Routes │ Middleware │ DTOs │ Swagger         │
├─────────────────────────────────────────────────────────────┤
│                    APPLICATION                              │
│  Use Cases │ Commands │ Queries │ Event Handlers            │
├─────────────────────────────────────────────────────────────┤
│                      DOMAIN                                 │
│  Entities │ Value Objects │ Aggregates │ Domain Services    │
│  Repository Interfaces │ Domain Events                      │
├─────────────────────────────────────────────────────────────┤
│                   INFRASTRUCTURE                            │
│  Repository Impl │ Connectors │ Database │ External APIs    │
└─────────────────────────────────────────────────────────────┘
```

### Structure détaillée du backend

```
backend/src/
├── domain/
│   ├── monitoring/
│   │   ├── entities/            # Endpoint, HealthStatus, MonitoringResult
│   │   ├── value-objects/       # EndpointUrl, EndpointType, StatusCode
│   │   ├── repositories/        # IEndpointRepository (interface)
│   │   └── services/            # HealthCheckService
│   │
│   ├── connector/
│   │   ├── entities/            # Connector
│   │   ├── value-objects/       # ConnectorType, ConnectorCredentials
│   │   ├── interfaces/          # IConnector (interface commune)
│   │   └── repositories/        # IConnectorRepository
│   │
│   ├── ticket/
│   │   ├── entities/            # Ticket
│   │   ├── value-objects/       # TicketStatus, TicketPriority
│   │   └── repositories/        # ITicketRepository
│   │
│   └── action/
│       ├── entities/            # Action, ActionLog
│       ├── value-objects/       # ActionType, ActionResult
│       └── repositories/        # IActionLogRepository
│
├── application/
│   ├── monitoring/
│   │   ├── commands/            # CreateEndpoint, UpdateEndpoint, DeleteEndpoint
│   │   ├── queries/             # GetAllEndpoints, GetEndpointHealth, GetMonitoringDashboard
│   │   └── handlers/            # Handlers pour chaque command/query
│   │
│   ├── connector/
│   │   ├── commands/            # ExecuteConnectorAction
│   │   ├── queries/             # GetConnectorStatus, ListConnectorResources
│   │   └── handlers/
│   │
│   ├── ticket/
│   │   └── queries/             # GetJiraTickets, GetGlpiTickets
│   │
│   └── action/
│       ├── commands/            # PurgeCache, TriggerWorkflow
│       └── handlers/
│
├── infrastructure/
│   ├── persistence/
│   │   ├── repositories/        # Implémentations PostgreSQL
│   │   ├── entities/            # Entités ORM/mapping
│   │   └── migrations/
│   │
│   ├── connectors/
│   │   ├── base/                # BaseConnector, ConnectorFactory
│   │   ├── TalendConnector
│   │   ├── ApimConnector
│   │   ├── GitHubConnector
│   │   ├── KeycloakConnector
│   │   ├── DomainHealthConnector
│   │   ├── CloudflareConnector
│   │   ├── GlpiConnector
│   │   └── JiraConnector
│   │
│   ├── config/                  # Configuration database, environment
│   │
│   └── services/                # EncryptionService, SchedulerService
│
└── presentation/
    ├── controllers/             # MonitoringController, EndpointController, etc.
    ├── routes/                  # Routes Express avec décorateurs Swagger
    ├── middleware/              # Auth, Error, Validation
    └── dtos/                    # Data Transfer Objects avec annotations Swagger
```

---

## Interface Connector (Pattern Strategy)

### Spécification de l'interface IConnector

Chaque connector doit implémenter l'interface commune suivante :

| Méthode | Description | Retour |
|---------|-------------|--------|
| `checkHealth()` | Vérifie l'état de santé du système externe | HealthCheckResult |
| `listResources(options?)` | Liste les ressources disponibles (jobs, tickets, workflows...) | Resource[] |
| `executeAction(action)` | Exécute une action spécifique | ActionResult |
| `getType()` | Retourne le type du connector | ConnectorType |
| `isConfigured()` | Vérifie si le connector est correctement configuré | boolean |

### Types de données

**HealthCheckResult :**
| Champ | Type | Description |
|-------|------|-------------|
| status | 'UP' \| 'DOWN' \| 'DEGRADED' | État du service |
| latencyMs | number (optionnel) | Temps de réponse en ms |
| statusCode | number (optionnel) | Code HTTP de la réponse |
| message | string (optionnel) | Message descriptif |
| timestamp | Date | Horodatage du check |

**Resource :**
| Champ | Type | Description |
|-------|------|-------------|
| id | string | Identifiant unique |
| name | string | Nom de la ressource |
| type | string | Type de ressource |
| metadata | objet | Métadonnées spécifiques |

**ActionResult :**
| Champ | Type | Description |
|-------|------|-------------|
| success | boolean | Succès de l'action |
| message | string | Message descriptif |
| data | objet (optionnel) | Données retournées |
| timestamp | Date | Horodatage |

**ConnectorType (enum) :**
- TALEND
- APIM
- GITHUB
- KEYCLOAK
- DOMAIN_HEALTH
- CLOUDFLARE
- GLPI
- JIRA

### Classe de base BaseConnector

Tous les connectors doivent hériter d'une classe abstraite `BaseConnector` qui fournit :

- Injection de la configuration (ConnectorConfig)
- Client HTTP configuré avec timeout et retries
- Logger dédié
- Gestion centralisée des erreurs
- Méthode `isConfigured()` par défaut

Chaque connector concret implémente les méthodes abstraites spécifiques à son système externe.

---

## Architecture Frontend (DDD-like)

### Structure détaillée du frontend

```
frontend/src/
├── domain/
│   ├── monitoring/
│   │   ├── models/              # Endpoint, MonitoringStatus
│   │   └── services/            # MonitoringService
│   │
│   ├── ticket/
│   │   ├── models/              # Ticket
│   │   └── services/            # TicketService
│   │
│   └── action/
│       ├── models/              # Action
│       └── services/            # ActionService
│
├── application/
│   ├── hooks/                   # useMonitoring, useTickets, useActions, useAuth
│   ├── stores/                  # monitoringStore, ticketStore, authStore
│   └── providers/               # AuthProvider, NotificationProvider
│
├── infrastructure/
│   ├── api/
│   │   ├── apiClient            # Client généré depuis Swagger/OpenAPI
│   │   ├── monitoringApi
│   │   ├── ticketApi
│   │   └── actionApi
│   │
│   └── storage/                 # localStorage utilities
│
└── presentation/
    ├── components/
    │   ├── common/              # Toolbar, StatusCard, Toast, ConfirmDialog, LoadingSpinner
    │   ├── monitoring/          # MonitoringDashboard, EndpointCard, EndpointDetails
    │   ├── tickets/             # TicketList, TicketCard, TicketFilters
    │   ├── actions/             # CacheActions, WorkflowList, WorkflowTrigger
    │   └── admin/               # EndpointAdmin, EndpointForm, ConnectorConfig
    │
    ├── pages/                   # DashboardPage, MonitoringPage, TicketsPage, etc.
    ├── layouts/                 # MainLayout
    └── routes/                  # AppRoutes
```

---

## Bounded Contexts

### Identification des contextes

| Bounded Context | Responsabilité | Entités principales |
|----------------|----------------|---------------------|
| **Monitoring** | Surveillance des endpoints | Endpoint, HealthStatus, MonitoringResult |
| **Connector** | Intégration systèmes externes | Connector, ConnectorConfig |
| **Ticket** | Gestion des tickets | Ticket, TicketStatus |
| **Action** | Actions opérationnelles | Action, ActionLog |
| **Identity** | Authentification et autorisations | User, Role |

### Context Map

```
┌─────────────────┐     ┌─────────────────┐
│   MONITORING    │────▶│   CONNECTOR     │
│                 │     │                 │
│  - Endpoints    │     │  - Intégrations │
│  - Health       │     │  - APIs ext.    │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │    ┌─────────────┐    │
         └───▶│   ACTION    │◀───┘
              │             │
              │  - Purge    │
              │  - Workflow │
              │  - Logs     │
              └──────┬──────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│     TICKET      │     │    IDENTITY     │
│                 │     │                 │
│  - Jira         │     │  - Auth Azure   │
│  - GLPI         │     │  - Roles        │
└─────────────────┘     └─────────────────┘
```

---

## Patterns utilisés

### Backend

| Pattern | Utilisation |
|---------|-------------|
| **Repository** | Abstraction de la persistence |
| **Factory** | Création des connectors |
| **Strategy** | Comportement des différents connectors |
| **Command/Query (CQRS light)** | Séparation lecture/écriture |
| **Value Object** | Types immuables (URL, Status, etc.) |
| **Domain Service** | Logique métier transverse |

### Frontend

| Pattern | Utilisation |
|---------|-------------|
| **Container/Presentational** | Séparation logique/UI |
| **Custom Hooks** | Réutilisation de logique |
| **Provider** | Injection de dépendances (contexte) |
| **Service Layer** | Appels API encapsulés |

---

## Flux de données

### Exemple : Affichage du dashboard monitoring

```
1. [Page] DashboardPage monte
         │
         ▼
2. [Hook] useMonitoring() s'exécute
         │
         ▼
3. [Service] MonitoringService.getStatus()
         │
         ▼
4. [API] GET /api/monitoring/status
         │
         ▼
5. [Controller] MonitoringController.getStatus()
         │
         ▼
6. [Handler] GetMonitoringDashboardHandler.handle()
         │
         ▼
7. [Repository] EndpointRepository.findAllEnabled()
         │
         ▼
8. [Service] HealthCheckService.checkAll(endpoints)
         │
         ▼
9. [Connector] Chaque connector.checkHealth()
         │
         ▼
10. Agrégation des résultats et retour
```

---

## Génération du client API

Le frontend utilise un **client TypeScript généré automatiquement** à partir de la documentation Swagger/OpenAPI exposée par le backend.

**Approche recommandée :**
- Le backend génère la documentation OpenAPI à partir des décorateurs/annotations dans le code (ex: tsoa, routing-controllers, swagger-jsdoc)
- Un script de build génère le client TypeScript pour le frontend (ex: openapi-typescript-codegen, orval)
- Le client généré est utilisé dans la couche `infrastructure/api` du frontend

---

## Documents connexes

- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
- [05 - Conception API](./05-api-design.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
