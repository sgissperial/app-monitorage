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

**Configuration du manifeste Azure AD :**

Les App Roles doivent être définis dans le manifeste Azure AD avec les propriétés suivantes :
- `id` : UUID unique pour chaque rôle
- `allowedMemberTypes` : ["User"]
- `displayName` : Nom affiché (ex: "Operator", "Administrator")
- `value` : Valeur technique (ex: "operator", "admin")
- `description` : Description du rôle

---

## Implémentation Backend

### Middleware d'authentification

Le middleware d'authentification doit :

1. **Extraire le token** : Récupérer le Bearer token depuis l'header `Authorization`
2. **Décoder le token** : Parser le JWT pour extraire le header (kid)
3. **Récupérer la clé publique** : Via JWKS depuis Azure AD (`https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys`)
4. **Valider le token** :
   - Algorithme : RS256
   - Audience : Client ID de l'application backend
   - Issuer : `https://login.microsoftonline.com/{tenant}/v2.0`
5. **Extraire les informations utilisateur** :
   - `oid` : ID utilisateur
   - `preferred_username` : Email
   - `name` : Nom complet
   - `roles` : Rôles assignés (défaut: ['user'])
6. **Injecter dans la requête** : Ajouter l'objet user à `req.user`

**Librairies recommandées :**
- `jsonwebtoken` pour la validation JWT
- `jwks-rsa` pour la récupération des clés publiques

### Middleware d'autorisation

Le middleware d'autorisation doit :

1. **Définir le mapping rôle → permissions** :
   - user : monitoring:read, tickets:read
   - operator : monitoring:read, tickets:read, actions:execute
   - admin : toutes les permissions

2. **Créer une fonction `requirePermission(...permissions)`** qui :
   - Récupère les rôles de l'utilisateur depuis `req.user.roles`
   - Calcule les permissions effectives selon le mapping
   - Vérifie que toutes les permissions requises sont présentes
   - Retourne 403 si permissions insuffisantes

### Application aux routes

- Toutes les routes API doivent être protégées par le middleware d'authentification
- Les routes admin doivent en plus vérifier la permission `admin:*`
- Les routes d'actions doivent vérifier la permission `actions:execute`

---

## Implémentation Frontend

### Configuration MSAL

Le frontend utilise **@azure/msal-browser** pour l'authentification.

**Configuration requise :**
- `clientId` : ID de l'application frontend Azure AD
- `authority` : URL d'autorité Azure AD
- `redirectUri` : URI de callback après login
- `postLogoutRedirectUri` : URI après logout
- `cacheLocation` : 'sessionStorage' (recommandé) ou 'localStorage'

**Scopes de login :**
- openid, profile, email
- Scopes API personnalisés

### Provider d'authentification

Le AuthProvider doit exposer :

| Propriété/Méthode | Description |
|-------------------|-------------|
| `user` | Compte utilisateur connecté (ou null) |
| `isAuthenticated` | Boolean indiquant si connecté |
| `isLoading` | Boolean pendant l'initialisation |
| `login()` | Déclenche la redirection de login |
| `logout()` | Déclenche la déconnexion |
| `getAccessToken()` | Récupère un token valide (avec refresh silencieux) |

**Comportement attendu :**
1. À l'initialisation, vérifier s'il existe une session
2. Gérer le retour de redirection (`handleRedirectPromise`)
3. Refresh silencieux du token si expiré
4. Redirection interactive si refresh impossible

### Intercepteur API

L'intercepteur HTTP doit :
1. Intercepter chaque requête vers l'API
2. Appeler `getAccessToken()` pour obtenir un token valide
3. Ajouter l'header `Authorization: Bearer {token}`

---

## Protection des données sensibles

### Stockage des credentials en base

Les credentials des connectors (API keys, tokens, etc.) doivent être chiffrés avant stockage.

**Algorithme recommandé :** AES-256-GCM

**Format de stockage :** `{iv}:{authTag}:{ciphertext}` (encodé en hexadécimal)

**Principes de l'EncryptionService :**
- Clé de chiffrement de 256 bits (64 caractères hex)
- IV aléatoire de 16 bytes pour chaque chiffrement
- AuthTag pour garantir l'intégrité

### Variables d'environnement

| Variable | Description | Format |
|----------|-------------|--------|
| `AZURE_TENANT_ID` | ID du tenant Azure AD | UUID |
| `AZURE_CLIENT_ID` | ID de l'application backend | UUID |
| `ENCRYPTION_KEY` | Clé AES-256 | 64 caractères hex |
| `DATABASE_URL` | Connection string PostgreSQL | URI PostgreSQL |

### Bonnes pratiques

1. **Ne jamais commiter de secrets** dans le code source
2. **Rotation régulière** des clés de chiffrement
3. **Audit des accès** aux credentials
4. **Principe du moindre privilège** pour les API keys des connectors

---

## Audit et traçabilité

### Structure des logs d'actions

Chaque action sensible génère un log avec :

| Champ | Type | Description |
|-------|------|-------------|
| id | UUID | Identifiant unique du log |
| action | enum | Type d'action (voir ci-dessous) |
| target | string | Cible de l'action |
| userId | string | ID de l'utilisateur |
| userEmail | string | Email de l'utilisateur |
| userRoles | string[] | Rôles de l'utilisateur |
| ipAddress | string | Adresse IP |
| userAgent | string | User-Agent du navigateur |
| parameters | objet | Paramètres de l'action |
| result | enum | SUCCESS ou FAILURE |
| errorMessage | string (opt) | Message d'erreur si échec |
| timestamp | datetime | Horodatage |

### Types d'actions loguées

- CACHE_PURGE_KEYCLOAK
- CACHE_PURGE_CLOUDFLARE
- WORKFLOW_TRIGGER
- ENDPOINT_CREATE
- ENDPOINT_UPDATE
- ENDPOINT_DELETE
- CONNECTOR_CONFIG_UPDATE

### Rétention des logs

| Type de log | Durée de rétention |
|-------------|-------------------|
| Logs d'actions | 90 jours minimum |
| Logs d'authentification | 30 jours |
| Logs techniques | 7 jours |

---

## Recommandations de sécurité

### Headers HTTP

Utiliser **helmet** pour configurer les headers de sécurité :

| Header | Configuration |
|--------|---------------|
| Strict-Transport-Security | max-age=31536000; includeSubDomains |
| X-Content-Type-Options | nosniff |
| X-Frame-Options | DENY |
| Content-Security-Policy | default-src 'self'; connect-src 'self' https://login.microsoftonline.com |
| X-XSS-Protection | 1; mode=block |

### Rate limiting

| Endpoint | Limite |
|----------|--------|
| API générale | 100 requêtes / 15 minutes / utilisateur |
| Actions (purge, workflow) | 10 requêtes / minute / utilisateur |
| Authentification | 5 requêtes / minute / IP |

### Validation des entrées

Utiliser une librairie de validation (Zod, Joi, class-validator) pour valider :
- Format des données entrantes
- Contraintes métier (longueur, format URL, etc.)
- Types de données

**Règles de validation pour CreateEndpoint :**
| Champ | Règles |
|-------|--------|
| name | string, min 1, max 100 |
| type | enum (APIM, DOMAIN, TALEND) |
| url | string, format URL valide |
| checkInterval | integer, min 10 (optionnel) |
| timeout | integer, min 1000 (optionnel) |
| expectedStatus | integer, 100-599 (optionnel) |
| enabled | boolean (optionnel) |

---

## Documents connexes

- [05 - Conception API](./05-api-design.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
