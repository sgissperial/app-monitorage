# 06 - Sécurité et authentification

## Vue d'ensemble

L'application utilise **Azure App Registration** (OAuth2 / OIDC) pour l'authentification des utilisateurs. Ce document détaille les mécanismes de sécurité, la gestion des rôles et la protection des données sensibles.

---

## Architecture d'authentification

### Flux OAuth2 / OIDC

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │────▶│   Azure AD   │────▶│   Backend    │
│    (SPA)     │◀────│    (IdP)     │◀────│   (API)      │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       │  1. Redirect       │                    │
       │────────────────────▶                    │
       │                    │                    │
       │  2. Login + MFA    │                    │
       │◀───────────────────│                    │
       │                    │                    │
       │  3. Auth Code      │                    │
       │◀────────────────────                    │
       │                    │                    │
       │  4. Exchange Code  │                    │
       │────────────────────▶                    │
       │                    │                    │
       │  5. Tokens         │                    │
       │◀────────────────────                    │
       │                    │                    │
       │  6. API Call + Token                    │
       │─────────────────────────────────────────▶
       │                    │                    │
       │                    │  7. Validate Token │
       │                    │◀───────────────────│
       │                    │                    │
       │  8. Response       │                    │
       │◀─────────────────────────────────────────
```

### Type de flux

- **Authorization Code Flow with PKCE** (recommandé pour les SPA)
- Pas de client secret côté frontend
- Tokens stockés en mémoire (pas de localStorage pour l'access token)

---

## Configuration Azure App Registration

### Application Frontend (SPA)

| Paramètre | Valeur |
|-----------|--------|
| Type | Single-page application |
| Redirect URIs | `http://localhost:5173/callback` (dev), `https://monitorage.example.com/callback` (prod) |
| Logout URI | `http://localhost:5173` (dev), `https://monitorage.example.com` (prod) |
| Implicit grant | Désactivé |
| PKCE | Activé |

### Application Backend (API)

| Paramètre | Valeur |
|-----------|--------|
| Type | Web API |
| Application ID URI | `api://app-monitorage-api` |
| Expose API | Scopes définis |

### Scopes exposés par l'API

| Scope | Description |
|-------|-------------|
| `api://app-monitorage-api/Monitoring.Read` | Lecture des données de monitoring |
| `api://app-monitorage-api/Actions.Execute` | Exécution d'actions (purge, workflow) |
| `api://app-monitorage-api/Admin.Full` | Administration complète |

---

## Gestion des rôles et permissions

### Rôles applicatifs

| Rôle | Description | Permissions |
|------|-------------|-------------|
| `user` | Utilisateur standard | Lecture monitoring, tickets |
| `operator` | Opérateur | Lecture + exécution d'actions |
| `admin` | Administrateur | Toutes les permissions |

### Matrice des permissions

| Permission | user | operator | admin |
|------------|------|----------|-------|
| `monitoring:read` | ✅ | ✅ | ✅ |
| `tickets:read` | ✅ | ✅ | ✅ |
| `actions:execute` | ❌ | ✅ | ✅ |
| `admin:endpoints` | ❌ | ❌ | ✅ |
| `admin:connectors` | ❌ | ❌ | ✅ |
| `admin:logs` | ❌ | ❌ | ✅ |

### Attribution des rôles

Les rôles sont attribués via :
1. **Azure AD App Roles** (recommandé)
2. **Groupes Azure AD** mappés vers des rôles applicatifs

```json
// Extrait du manifeste Azure AD
{
  "appRoles": [
    {
      "id": "uuid-1",
      "allowedMemberTypes": ["User"],
      "displayName": "Operator",
      "value": "operator",
      "description": "Opérateur avec droits d'exécution d'actions"
    },
    {
      "id": "uuid-2",
      "allowedMemberTypes": ["User"],
      "displayName": "Administrator",
      "value": "admin",
      "description": "Administrateur avec tous les droits"
    }
  ]
}
```

---

## Implémentation Backend

### Middleware d'authentification

```typescript
// infrastructure/middleware/authMiddleware.ts

import { Request, Response, NextFunction } from 'express';
import { verify, decode } from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}/discovery/v2.0/keys`,
});

export const authMiddleware = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        error: { code: 'UNAUTHORIZED', message: 'Token manquant' },
      });
    }

    const token = authHeader.substring(7);
    const decoded = decode(token, { complete: true });

    if (!decoded) {
      return res.status(401).json({
        success: false,
        error: { code: 'INVALID_TOKEN', message: 'Token invalide' },
      });
    }

    const key = await client.getSigningKey(decoded.header.kid);
    const signingKey = key.getPublicKey();

    const payload = verify(token, signingKey, {
      algorithms: ['RS256'],
      audience: process.env.AZURE_CLIENT_ID,
      issuer: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}/v2.0`,
    });

    req.user = {
      id: payload.oid,
      email: payload.preferred_username,
      name: payload.name,
      roles: payload.roles || ['user'],
    };

    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      error: { code: 'TOKEN_VALIDATION_FAILED', message: 'Validation du token échouée' },
    });
  }
};
```

### Middleware d'autorisation

```typescript
// infrastructure/middleware/authorizationMiddleware.ts

import { Request, Response, NextFunction } from 'express';

type Permission =
  | 'monitoring:read'
  | 'tickets:read'
  | 'actions:execute'
  | 'admin:endpoints'
  | 'admin:connectors'
  | 'admin:logs';

const rolePermissions: Record<string, Permission[]> = {
  user: ['monitoring:read', 'tickets:read'],
  operator: ['monitoring:read', 'tickets:read', 'actions:execute'],
  admin: [
    'monitoring:read',
    'tickets:read',
    'actions:execute',
    'admin:endpoints',
    'admin:connectors',
    'admin:logs',
  ],
};

export const requirePermission = (...permissions: Permission[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRoles = req.user?.roles || [];

    const userPermissions = new Set<Permission>();
    for (const role of userRoles) {
      const perms = rolePermissions[role] || [];
      perms.forEach((p) => userPermissions.add(p));
    }

    const hasPermission = permissions.every((p) => userPermissions.has(p));

    if (!hasPermission) {
      return res.status(403).json({
        success: false,
        error: {
          code: 'FORBIDDEN',
          message: 'Permissions insuffisantes',
          required: permissions,
        },
      });
    }

    next();
  };
};
```

### Utilisation dans les routes

```typescript
// presentation/routes/admin.routes.ts

import { Router } from 'express';
import { authMiddleware } from '../middleware/authMiddleware';
import { requirePermission } from '../middleware/authorizationMiddleware';
import { EndpointController } from '../controllers/EndpointController';

const router = Router();

// Toutes les routes admin nécessitent authentification
router.use(authMiddleware);

// Routes endpoints (admin only)
router.get(
  '/endpoints',
  requirePermission('admin:endpoints'),
  EndpointController.list
);

router.post(
  '/endpoints',
  requirePermission('admin:endpoints'),
  EndpointController.create
);

router.put(
  '/endpoints/:id',
  requirePermission('admin:endpoints'),
  EndpointController.update
);

router.delete(
  '/endpoints/:id',
  requirePermission('admin:endpoints'),
  EndpointController.delete
);

export default router;
```

---

## Implémentation Frontend

### Configuration MSAL

```typescript
// infrastructure/auth/msalConfig.ts

import { Configuration, LogLevel } from '@azure/msal-browser';

export const msalConfig: Configuration = {
  auth: {
    clientId: import.meta.env.VITE_AZURE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${import.meta.env.VITE_AZURE_TENANT_ID}`,
    redirectUri: import.meta.env.VITE_REDIRECT_URI,
    postLogoutRedirectUri: import.meta.env.VITE_POST_LOGOUT_URI,
  },
  cache: {
    cacheLocation: 'sessionStorage', // ou 'localStorage' si refresh token souhaité
    storeAuthStateInCookie: false,
  },
  system: {
    loggerOptions: {
      logLevel: LogLevel.Warning,
    },
  },
};

export const loginRequest = {
  scopes: [
    'openid',
    'profile',
    'email',
    'api://app-monitorage-api/Monitoring.Read',
    'api://app-monitorage-api/Actions.Execute',
  ],
};

export const apiRequest = {
  scopes: ['api://app-monitorage-api/.default'],
};
```

### Provider d'authentification

```typescript
// application/providers/AuthProvider.tsx

import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import {
  PublicClientApplication,
  AccountInfo,
  InteractionRequiredAuthError,
} from '@azure/msal-browser';
import { msalConfig, loginRequest, apiRequest } from '../../infrastructure/auth/msalConfig';

interface AuthContextType {
  user: AccountInfo | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  getAccessToken: () => Promise<string>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

const msalInstance = new PublicClientApplication(msalConfig);

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<AccountInfo | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const initAuth = async () => {
      await msalInstance.initialize();
      const response = await msalInstance.handleRedirectPromise();

      if (response) {
        setUser(response.account);
      } else {
        const accounts = msalInstance.getAllAccounts();
        if (accounts.length > 0) {
          setUser(accounts[0]);
        }
      }
      setIsLoading(false);
    };

    initAuth();
  }, []);

  const login = async () => {
    await msalInstance.loginRedirect(loginRequest);
  };

  const logout = async () => {
    await msalInstance.logoutRedirect();
  };

  const getAccessToken = async (): Promise<string> => {
    if (!user) throw new Error('User not authenticated');

    try {
      const response = await msalInstance.acquireTokenSilent({
        ...apiRequest,
        account: user,
      });
      return response.accessToken;
    } catch (error) {
      if (error instanceof InteractionRequiredAuthError) {
        await msalInstance.acquireTokenRedirect(apiRequest);
        throw error;
      }
      throw error;
    }
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        logout,
        getAccessToken,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

### Intercepteur API

```typescript
// infrastructure/api/apiClient.ts

import axios from 'axios';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
});

// L'intercepteur sera configuré après initialisation de l'auth
export const configureApiAuth = (getAccessToken: () => Promise<string>) => {
  apiClient.interceptors.request.use(
    async (config) => {
      const token = await getAccessToken();
      config.headers.Authorization = `Bearer ${token}`;
      return config;
    },
    (error) => Promise.reject(error)
  );
};

export default apiClient;
```

---

## Protection des données sensibles

### Stockage des credentials en base

Les credentials des connectors (API keys, tokens, etc.) sont chiffrés avant stockage :

```typescript
// infrastructure/services/EncryptionService.ts

import crypto from 'crypto';

export class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;

  constructor() {
    const encryptionKey = process.env.ENCRYPTION_KEY;
    if (!encryptionKey || encryptionKey.length !== 64) {
      throw new Error('ENCRYPTION_KEY must be 64 hex characters (256 bits)');
    }
    this.key = Buffer.from(encryptionKey, 'hex');
  }

  encrypt(plaintext: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    // Format: iv:authTag:encrypted
    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
  }

  decrypt(ciphertext: string): string {
    const [ivHex, authTagHex, encrypted] = ciphertext.split(':');

    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');

    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }
}
```

### Variables d'environnement

| Variable | Description | Exemple |
|----------|-------------|---------|
| `AZURE_TENANT_ID` | ID du tenant Azure AD | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `AZURE_CLIENT_ID` | ID de l'application backend | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `ENCRYPTION_KEY` | Clé AES-256 (hex) | `64 caractères hex` |
| `DATABASE_URL` | Connection string PostgreSQL | `postgresql://...` |

### Bonnes pratiques

1. **Ne jamais commiter de secrets** dans le code source
2. **Rotation régulière** des clés de chiffrement
3. **Audit des accès** aux credentials
4. **Principe du moindre privilège** pour les API keys des connectors

---

## Audit et traçabilité

### Logs d'actions

Toutes les actions sensibles sont loguées :

```typescript
// domain/action/entities/ActionLog.ts

export interface ActionLog {
  id: string;
  action: ActionType;
  target: string;
  userId: string;
  userEmail: string;
  userRoles: string[];
  ipAddress: string;
  userAgent: string;
  parameters: Record<string, unknown>;
  result: 'SUCCESS' | 'FAILURE';
  errorMessage?: string;
  timestamp: Date;
}

export enum ActionType {
  CACHE_PURGE_KEYCLOAK = 'CACHE_PURGE_KEYCLOAK',
  CACHE_PURGE_CLOUDFLARE = 'CACHE_PURGE_CLOUDFLARE',
  WORKFLOW_TRIGGER = 'WORKFLOW_TRIGGER',
  ENDPOINT_CREATE = 'ENDPOINT_CREATE',
  ENDPOINT_UPDATE = 'ENDPOINT_UPDATE',
  ENDPOINT_DELETE = 'ENDPOINT_DELETE',
  CONNECTOR_CONFIG_UPDATE = 'CONNECTOR_CONFIG_UPDATE',
}
```

### Rétention des logs

- **Logs d'actions** : 90 jours minimum
- **Logs d'authentification** : 30 jours
- **Logs techniques** : 7 jours

---

## Recommandations de sécurité

### Headers HTTP

```typescript
// presentation/middleware/securityMiddleware.ts

import helmet from 'helmet';

app.use(helmet());
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'", 'https://login.microsoftonline.com'],
  },
}));
```

### Rate limiting

```typescript
import rateLimit from 'express-rate-limit';

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requêtes par fenêtre
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', apiLimiter);
```

### Validation des entrées

Utilisation de **Zod** ou **Joi** pour la validation :

```typescript
import { z } from 'zod';

const createEndpointSchema = z.object({
  name: z.string().min(1).max(100),
  type: z.enum(['APIM', 'DOMAIN', 'TALEND']),
  url: z.string().url(),
  checkInterval: z.number().int().min(10).optional(),
  timeout: z.number().int().min(1000).optional(),
  expectedStatus: z.number().int().min(100).max(599).optional(),
  enabled: z.boolean().optional(),
});
```

---

## Documents connexes

- [05 - Conception API](./05-api-design.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
