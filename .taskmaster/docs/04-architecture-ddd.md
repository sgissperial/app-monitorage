# 04 - Architecture DDD

## Vue d'ensemble

L'application suit les principes du **Domain-Driven Design (DDD)** avec une architecture en couches distinctes, tant côté backend que frontend.

---

## Architecture globale

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                │
│                   (React + Vite + MUI)                         │
├─────────────────────────────────────────────────────────────────┤
│                         API REST                                │
│                   (Swagger / OpenAPI)                          │
├─────────────────────────────────────────────────────────────────┤
│                         BACKEND                                 │
│                      (Node.js 22)                              │
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
│   ├── backend/                 # API Node.js
│   │   ├── src/
│   │   │   ├── domain/          # Couche domaine
│   │   │   ├── application/     # Couche application (use cases)
│   │   │   ├── infrastructure/  # Couche infrastructure
│   │   │   └── presentation/    # Couche présentation (API)
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── frontend/                # Application React
│   │   ├── src/
│   │   │   ├── domain/          # Logique métier frontend
│   │   │   ├── application/     # Services et état
│   │   │   ├── infrastructure/  # API client, storage
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
│   │   ├── entities/
│   │   │   ├── Endpoint.ts
│   │   │   ├── HealthStatus.ts
│   │   │   └── MonitoringResult.ts
│   │   ├── value-objects/
│   │   │   ├── EndpointUrl.ts
│   │   │   ├── EndpointType.ts
│   │   │   └── StatusCode.ts
│   │   ├── repositories/
│   │   │   └── IEndpointRepository.ts
│   │   └── services/
│   │       └── HealthCheckService.ts
│   │
│   ├── connector/
│   │   ├── entities/
│   │   │   └── Connector.ts
│   │   ├── value-objects/
│   │   │   ├── ConnectorType.ts
│   │   │   └── ConnectorCredentials.ts
│   │   ├── interfaces/
│   │   │   └── IConnector.ts
│   │   └── repositories/
│   │       └── IConnectorRepository.ts
│   │
│   ├── ticket/
│   │   ├── entities/
│   │   │   └── Ticket.ts
│   │   ├── value-objects/
│   │   │   ├── TicketStatus.ts
│   │   │   └── TicketPriority.ts
│   │   └── repositories/
│   │       └── ITicketRepository.ts
│   │
│   └── action/
│       ├── entities/
│       │   ├── Action.ts
│       │   └── ActionLog.ts
│       ├── value-objects/
│       │   ├── ActionType.ts
│       │   └── ActionResult.ts
│       └── repositories/
│           └── IActionLogRepository.ts
│
├── application/
│   ├── monitoring/
│   │   ├── commands/
│   │   │   ├── CreateEndpointCommand.ts
│   │   │   ├── UpdateEndpointCommand.ts
│   │   │   └── DeleteEndpointCommand.ts
│   │   ├── queries/
│   │   │   ├── GetAllEndpointsQuery.ts
│   │   │   ├── GetEndpointHealthQuery.ts
│   │   │   └── GetMonitoringDashboardQuery.ts
│   │   └── handlers/
│   │       ├── CreateEndpointHandler.ts
│   │       ├── UpdateEndpointHandler.ts
│   │       └── ...
│   │
│   ├── connector/
│   │   ├── commands/
│   │   │   └── ExecuteConnectorActionCommand.ts
│   │   ├── queries/
│   │   │   ├── GetConnectorStatusQuery.ts
│   │   │   └── ListConnectorResourcesQuery.ts
│   │   └── handlers/
│   │       └── ...
│   │
│   ├── ticket/
│   │   └── queries/
│   │       ├── GetJiraTicketsQuery.ts
│   │       └── GetGlpiTicketsQuery.ts
│   │
│   └── action/
│       ├── commands/
│       │   ├── PurgeCacheCommand.ts
│       │   └── TriggerWorkflowCommand.ts
│       └── handlers/
│           └── ...
│
├── infrastructure/
│   ├── persistence/
│   │   ├── repositories/
│   │   │   ├── PostgresEndpointRepository.ts
│   │   │   ├── PostgresConnectorRepository.ts
│   │   │   └── PostgresActionLogRepository.ts
│   │   ├── entities/
│   │   │   ├── EndpointEntity.ts
│   │   │   ├── ConnectorEntity.ts
│   │   │   └── ActionLogEntity.ts
│   │   └── migrations/
│   │       └── ...
│   │
│   ├── connectors/
│   │   ├── base/
│   │   │   ├── BaseConnector.ts
│   │   │   └── ConnectorFactory.ts
│   │   ├── TalendConnector.ts
│   │   ├── ApimConnector.ts
│   │   ├── GitHubConnector.ts
│   │   ├── KeycloakConnector.ts
│   │   ├── DomainHealthConnector.ts
│   │   ├── CloudflareConnector.ts
│   │   ├── GlpiConnector.ts
│   │   └── JiraConnector.ts
│   │
│   ├── config/
│   │   ├── database.ts
│   │   └── environment.ts
│   │
│   └── services/
│       ├── EncryptionService.ts
│       └── SchedulerService.ts
│
└── presentation/
    ├── controllers/
    │   ├── MonitoringController.ts
    │   ├── EndpointController.ts
    │   ├── ConnectorController.ts
    │   ├── TicketController.ts
    │   └── ActionController.ts
    │
    ├── routes/
    │   ├── index.ts
    │   ├── monitoring.routes.ts
    │   ├── endpoint.routes.ts
    │   ├── connector.routes.ts
    │   ├── ticket.routes.ts
    │   └── action.routes.ts
    │
    ├── middleware/
    │   ├── authMiddleware.ts
    │   ├── errorMiddleware.ts
    │   └── validationMiddleware.ts
    │
    ├── dtos/
    │   ├── EndpointDto.ts
    │   ├── MonitoringResultDto.ts
    │   ├── TicketDto.ts
    │   └── ActionDto.ts
    │
    └── swagger/
        └── openapi.yaml
```

---

## Interface Connector (Pattern Strategy)

### Définition de l'interface

```typescript
// domain/connector/interfaces/IConnector.ts

export interface IConnector {
  /**
   * Vérifie l'état de santé du système externe
   */
  checkHealth(): Promise<HealthCheckResult>;

  /**
   * Liste les ressources disponibles (jobs, tickets, workflows...)
   */
  listResources(options?: ListResourcesOptions): Promise<Resource[]>;

  /**
   * Exécute une action spécifique
   */
  executeAction(action: ConnectorAction): Promise<ActionResult>;

  /**
   * Retourne le type du connector
   */
  getType(): ConnectorType;

  /**
   * Vérifie si le connector est correctement configuré
   */
  isConfigured(): boolean;
}

export interface HealthCheckResult {
  status: 'UP' | 'DOWN' | 'DEGRADED';
  latencyMs?: number;
  statusCode?: number;
  message?: string;
  timestamp: Date;
}

export interface Resource {
  id: string;
  name: string;
  type: string;
  metadata: Record<string, unknown>;
}

export interface ConnectorAction {
  type: string;
  parameters: Record<string, unknown>;
}

export interface ActionResult {
  success: boolean;
  message: string;
  data?: Record<string, unknown>;
  timestamp: Date;
}

export enum ConnectorType {
  TALEND = 'TALEND',
  APIM = 'APIM',
  GITHUB = 'GITHUB',
  KEYCLOAK = 'KEYCLOAK',
  DOMAIN_HEALTH = 'DOMAIN_HEALTH',
  CLOUDFLARE = 'CLOUDFLARE',
  GLPI = 'GLPI',
  JIRA = 'JIRA',
}
```

### Classe de base abstraite

```typescript
// infrastructure/connectors/base/BaseConnector.ts

export abstract class BaseConnector implements IConnector {
  protected readonly config: ConnectorConfig;
  protected readonly httpClient: HttpClient;
  protected readonly logger: Logger;

  constructor(config: ConnectorConfig) {
    this.config = config;
    this.httpClient = new HttpClient({
      baseUrl: config.baseUrl,
      timeout: config.timeout ?? 5000,
      retries: config.retries ?? 3,
    });
    this.logger = new Logger(this.getType());
  }

  abstract checkHealth(): Promise<HealthCheckResult>;
  abstract listResources(options?: ListResourcesOptions): Promise<Resource[]>;
  abstract executeAction(action: ConnectorAction): Promise<ActionResult>;
  abstract getType(): ConnectorType;

  isConfigured(): boolean {
    return !!(this.config.baseUrl && this.config.credentials);
  }

  protected async handleError(error: Error): Promise<never> {
    this.logger.error('Connector error', { error });
    throw new ConnectorError(this.getType(), error.message);
  }
}
```

---

## Architecture Frontend (DDD-like)

### Structure détaillée du frontend

```
frontend/src/
├── domain/
│   ├── monitoring/
│   │   ├── models/
│   │   │   ├── Endpoint.ts
│   │   │   └── MonitoringStatus.ts
│   │   └── services/
│   │       └── MonitoringService.ts
│   │
│   ├── ticket/
│   │   ├── models/
│   │   │   └── Ticket.ts
│   │   └── services/
│   │       └── TicketService.ts
│   │
│   └── action/
│       ├── models/
│       │   └── Action.ts
│       └── services/
│           └── ActionService.ts
│
├── application/
│   ├── hooks/
│   │   ├── useMonitoring.ts
│   │   ├── useTickets.ts
│   │   ├── useActions.ts
│   │   └── useAuth.ts
│   │
│   ├── stores/
│   │   ├── monitoringStore.ts
│   │   ├── ticketStore.ts
│   │   └── authStore.ts
│   │
│   └── providers/
│       ├── AuthProvider.tsx
│       └── NotificationProvider.tsx
│
├── infrastructure/
│   ├── api/
│   │   ├── apiClient.ts          # Client généré depuis Swagger
│   │   ├── monitoringApi.ts
│   │   ├── ticketApi.ts
│   │   └── actionApi.ts
│   │
│   └── storage/
│       └── localStorage.ts
│
└── presentation/
    ├── components/
    │   ├── common/
    │   │   ├── Toolbar.tsx
    │   │   ├── StatusCard.tsx
    │   │   ├── Toast.tsx
    │   │   ├── ConfirmDialog.tsx
    │   │   └── LoadingSpinner.tsx
    │   │
    │   ├── monitoring/
    │   │   ├── MonitoringDashboard.tsx
    │   │   ├── EndpointCard.tsx
    │   │   └── EndpointDetails.tsx
    │   │
    │   ├── tickets/
    │   │   ├── TicketList.tsx
    │   │   ├── TicketCard.tsx
    │   │   └── TicketFilters.tsx
    │   │
    │   ├── actions/
    │   │   ├── CacheActions.tsx
    │   │   ├── WorkflowList.tsx
    │   │   └── WorkflowTrigger.tsx
    │   │
    │   └── admin/
    │       ├── EndpointAdmin.tsx
    │       ├── EndpointForm.tsx
    │       └── ConnectorConfig.tsx
    │
    ├── pages/
    │   ├── DashboardPage.tsx
    │   ├── MonitoringPage.tsx
    │   ├── TicketsPage.tsx
    │   ├── CachesPage.tsx
    │   ├── GitHubActionsPage.tsx
    │   └── AdminPage.tsx
    │
    ├── layouts/
    │   └── MainLayout.tsx
    │
    └── routes/
        └── AppRoutes.tsx
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

## Documents connexes

- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
- [05 - Conception API](./05-api-design.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
