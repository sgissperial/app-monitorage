# 08 - Exigences non fonctionnelles

## Vue d'ensemble

Ce document définit les exigences non fonctionnelles (NFR) de l'application App Monitorage, couvrant la performance, la disponibilité, la sécurité, la maintenabilité et l'utilisabilité.

---

## Performance

### NFR-PERF-001 : Temps de réponse API

| Métrique | Cible | Critique |
|----------|-------|----------|
| P50 (médiane) | < 200ms | < 500ms |
| P95 | < 500ms | < 1000ms |
| P99 | < 1000ms | < 2000ms |

**Contexte :** Temps de réponse mesuré côté serveur, hors latence réseau.

### NFR-PERF-002 : Temps de chargement frontend

| Métrique | Cible | Critique |
|----------|-------|----------|
| First Contentful Paint (FCP) | < 1.5s | < 3s |
| Largest Contentful Paint (LCP) | < 2.5s | < 4s |
| Time to Interactive (TTI) | < 3s | < 5s |
| Cumulative Layout Shift (CLS) | < 0.1 | < 0.25 |

### NFR-PERF-003 : Rafraîchissement du dashboard

| Métrique | Cible |
|----------|-------|
| Temps de rafraîchissement complet | < 2s |
| Intervalle de rafraîchissement configurable | 10s - 300s |
| Rafraîchissement partiel (delta) | < 500ms |

### NFR-PERF-004 : Charge supportée

| Métrique | Cible |
|----------|-------|
| Utilisateurs concurrents | 50 |
| Requêtes par seconde (RPS) | 100 |
| Endpoints monitorés simultanément | 200 |

### NFR-PERF-005 : Base de données

| Métrique | Cible |
|----------|-------|
| Temps de requête (lecture) | < 50ms |
| Temps de requête (écriture) | < 100ms |
| Connexions simultanées | 20 |

---

## Disponibilité et fiabilité

### NFR-AVAIL-001 : Disponibilité globale

| Métrique | Cible |
|----------|-------|
| Disponibilité annuelle | 99.5% |
| Temps d'indisponibilité max/mois | 3.6h |
| Temps d'indisponibilité max/incident | 1h |

### NFR-AVAIL-002 : Recovery Time Objective (RTO)

| Scénario | RTO |
|----------|-----|
| Incident applicatif | < 15min |
| Incident infrastructure | < 1h |
| Disaster recovery | < 4h |

### NFR-AVAIL-003 : Recovery Point Objective (RPO)

| Type de données | RPO |
|-----------------|-----|
| Configuration endpoints | < 5min |
| Historique monitoring | < 1h |
| Logs d'actions | < 1h |

### NFR-AVAIL-004 : Résilience des connectors

| Comportement | Spécification |
|--------------|---------------|
| Timeout par défaut | 5000ms |
| Nombre de retries | 3 |
| Backoff strategy | Exponential (1s, 2s, 4s) |
| Circuit breaker | Ouverture après 5 échecs consécutifs |
| Temps de récupération circuit | 30s |

### NFR-AVAIL-005 : Graceful degradation

En cas d'indisponibilité d'un système externe :
- L'application reste accessible
- Les autres fonctionnalités restent opérationnelles
- Un message d'erreur clair est affiché pour le système indisponible
- Les dernières données connues sont affichées (si applicable)

---

## Sécurité

### NFR-SEC-001 : Authentification

| Exigence | Spécification |
|----------|---------------|
| Protocole | OAuth2 / OIDC |
| Provider | Azure AD |
| Token expiration | 1h (access), 24h (refresh) |
| MFA | Supporté via Azure AD |

### NFR-SEC-002 : Autorisation

| Exigence | Spécification |
|----------|---------------|
| Modèle | RBAC (Role-Based Access Control) |
| Granularité | Action-level permissions |
| Validation | Côté serveur obligatoire |

### NFR-SEC-003 : Chiffrement

| Type | Spécification |
|------|---------------|
| En transit | TLS 1.2+ obligatoire |
| Au repos (credentials) | AES-256-GCM |
| Au repos (base de données) | Chiffrement Azure |

### NFR-SEC-004 : Protection des données

| Exigence | Spécification |
|----------|---------------|
| Credentials en base | Chiffrés |
| Credentials en logs | Masqués |
| Tokens en frontend | Mémoire uniquement |
| Headers sensibles | Non loggés |

### NFR-SEC-005 : Headers de sécurité HTTP

| Header | Valeur |
|--------|--------|
| Strict-Transport-Security | max-age=31536000; includeSubDomains |
| X-Content-Type-Options | nosniff |
| X-Frame-Options | DENY |
| Content-Security-Policy | default-src 'self' |
| X-XSS-Protection | 1; mode=block |

### NFR-SEC-006 : Rate limiting

| Endpoint | Limite |
|----------|--------|
| API générale | 100 req/15min/user |
| Actions (purge, workflow) | 10 req/min/user |
| Authentification | 5 req/min/IP |

### NFR-SEC-007 : Audit et traçabilité

| Type d'événement | Rétention |
|------------------|-----------|
| Actions utilisateur | 90 jours |
| Authentification | 30 jours |
| Erreurs applicatives | 30 jours |

---

## Maintenabilité

### NFR-MAINT-001 : Qualité du code

| Métrique | Cible |
|----------|-------|
| Couverture de tests (backend) | > 80% |
| Couverture de tests (frontend) | > 70% |
| Complexité cyclomatique max | 15 |
| Duplication de code | < 3% |

### NFR-MAINT-002 : Documentation

| Type | Exigence |
|------|----------|
| API | Swagger/OpenAPI complet |
| Code | JSDoc pour fonctions publiques |
| Architecture | Diagrammes à jour |
| Déploiement | README avec instructions |

### NFR-MAINT-003 : Standards de code

| Outil | Configuration |
|-------|---------------|
| Linter | ESLint (config stricte) |
| Formatter | Prettier |
| Type checking | TypeScript strict mode |
| Git hooks | Husky + lint-staged |

### NFR-MAINT-004 : Dépendances

| Exigence | Spécification |
|----------|---------------|
| Mise à jour | Mensuelle minimum |
| Audit sécurité | `npm audit` en CI |
| Versions | Pinned (package-lock.json) |

### NFR-MAINT-005 : Logging

| Niveau | Utilisation |
|--------|-------------|
| ERROR | Erreurs bloquantes |
| WARN | Situations anormales non bloquantes |
| INFO | Événements métier importants |
| DEBUG | Détails techniques (dev uniquement) |

Format de log structuré (JSON) :
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "message": "Endpoint check completed",
  "context": {
    "endpointId": "uuid",
    "status": "UP",
    "latencyMs": 45
  },
  "requestId": "uuid"
}
```

---

## Utilisabilité

### NFR-UX-001 : Accessibilité

| Standard | Niveau |
|----------|--------|
| WCAG | 2.1 AA |
| Navigation clavier | Complète |
| Screen reader | Compatible |
| Contraste | Ratio minimum 4.5:1 |

### NFR-UX-002 : Responsive design

| Breakpoint | Support |
|------------|---------|
| Desktop (> 1200px) | Complet |
| Tablet (768px - 1200px) | Complet |
| Mobile (< 768px) | Consultation uniquement |

### NFR-UX-003 : Feedback utilisateur

| Action | Feedback |
|--------|----------|
| Chargement | Spinner ou skeleton |
| Action réussie | Toaster vert (5s) |
| Erreur | Toaster rouge (persistant ou 10s) |
| Confirmation requise | Modal |

### NFR-UX-004 : Internationalisation

| Exigence | Spécification |
|----------|---------------|
| Langue par défaut | Français |
| Langues supportées (MVP) | Français uniquement |
| Format dates | DD/MM/YYYY HH:mm |
| Format nombres | 1 234,56 |

### NFR-UX-005 : Temps de réaction UI

| Action | Temps max |
|--------|-----------|
| Clic bouton → feedback | < 100ms |
| Navigation entre pages | < 300ms |
| Ouverture modal | < 200ms |

---

## Compatibilité

### NFR-COMPAT-001 : Navigateurs supportés

| Navigateur | Version minimum |
|------------|-----------------|
| Chrome | 2 dernières versions |
| Firefox | 2 dernières versions |
| Edge | 2 dernières versions |
| Safari | 2 dernières versions |

### NFR-COMPAT-002 : Résolutions d'écran

| Résolution | Support |
|------------|---------|
| 1920x1080 | Optimal |
| 1366x768 | Complet |
| 1280x720 | Complet |
| Retina/HiDPI | Supporté |

---

## Scalabilité

### NFR-SCALE-001 : Horizontal scaling

| Composant | Scalabilité |
|-----------|-------------|
| Backend API | 1 à 3 instances (auto-scaling) |
| Frontend | CDN (illimité) |
| Base de données | Vertical uniquement (MVP) |

### NFR-SCALE-002 : Limites de données

| Entité | Limite |
|--------|--------|
| Endpoints monitorés | 500 max |
| Historique par endpoint | 7 jours |
| Logs d'actions | 90 jours |
| Taille payload API | 1 MB max |

---

## Monitoring et observabilité

### NFR-OBS-001 : Métriques applicatives

| Métrique | Collecte |
|----------|----------|
| Requêtes HTTP (count, latency) | Application Insights |
| Erreurs (rate, types) | Application Insights |
| Taux de disponibilité connectors | Custom metrics |
| Utilisation CPU/RAM | Azure Monitor |

### NFR-OBS-002 : Alertes

| Condition | Sévérité | Action |
|-----------|----------|--------|
| Error rate > 5% | High | Notification immédiate |
| Latency P95 > 2s | Medium | Notification |
| CPU > 80% (5min) | Medium | Auto-scale |
| Connector down > 5min | High | Notification |

### NFR-OBS-003 : Dashboards

| Dashboard | Contenu |
|-----------|---------|
| Santé applicative | Uptime, errors, latency |
| Performance | Response times, throughput |
| Business | Actions exécutées, utilisateurs actifs |

---

## Tableau récapitulatif des priorités

| Catégorie | Priorité | Justification |
|-----------|----------|---------------|
| Sécurité | Critique | Application interne avec accès à des systèmes sensibles |
| Disponibilité | Haute | Outil opérationnel utilisé en situation de crise |
| Performance | Haute | Besoin de réactivité pour le diagnostic |
| Utilisabilité | Moyenne | Interface simple, utilisateurs techniques |
| Maintenabilité | Moyenne | Projet évolutif avec ajout de connectors |
| Scalabilité | Basse | Usage interne limité (MVP) |

---

## Documents connexes

- [06 - Sécurité et authentification](./06-security-auth.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
- [09 - Critères d'acceptation](./09-acceptance-criteria.md)
