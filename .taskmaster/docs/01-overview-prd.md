# 01 - Vue d'ensemble du produit (PRD)

## Nom du projet

**App Monitorage** – Application web interne de monitoring et d'actions opérationnelles.

---

## Résumé exécutif

App Monitorage est une application web interne destinée aux équipes techniques (DevOps, développeurs, support N2/N3). Elle centralise la surveillance de services applicatifs, la consultation de tickets et l'exécution d'actions techniques ponctuelles (purge de cache, déclenchement de workflows).

L'objectif principal est de **remplacer ou compléter les consoles existantes** (Talend TMC, APIM, Jira, GLPI, etc.) par une **interface unique, rapide et orientée opérations**.

---

## Problématique adressée

Actuellement, les équipes techniques doivent jongler entre plusieurs outils et consoles pour :
- Vérifier l'état de santé des services (APIM, domaines publics, Talend)
- Consulter les tickets en cours (Jira, GLPI)
- Effectuer des actions de maintenance (purge cache Keycloak, Cloudflare)
- Déclencher des workflows CI/CD (GitHub Actions)

Cette fragmentation entraîne :
- Une **perte de temps** dans le diagnostic
- Des **risques d'erreur** lors des interventions
- Une **complexité opérationnelle** croissante

---

## Vision produit

> Une interface unique et actionnable pour surveiller, diagnostiquer et agir sur l'ensemble de l'écosystème technique.

---

## Objectifs stratégiques

| Objectif | Indicateur de succès |
|----------|---------------------|
| Centraliser l'état des systèmes | 100% des services critiques visibles en un coup d'œil |
| Réduire le temps de diagnostic | Temps moyen de détection d'incident réduit de 50% |
| Simplifier les actions récurrentes | Actions one-click pour purge cache, workflows |
| Améliorer l'expérience opérateur | UI lisible, réactive, sans formation nécessaire |

---

## Périmètre fonctionnel (MVP)

### Intégrations prévues

| Système | Type d'intégration | Authentification |
|---------|-------------------|------------------|
| Talend TMC | Lecture jobs en erreur | API Key |
| APIM (Azure) | Healthchecks endpoints | API Key |
| GitHub Actions | Liste + déclenchement workflows | PAT (Personal Access Token) |
| Keycloak | Purge cache realm | Credentials admin |
| Domaines publics | Healthcheck HTTP | Aucune |
| Cloudflare | Purge cache | API Key |
| GLPI | Liste tickets utilisateur | API Key |
| Jira | Tickets sprint + backlog | API Key / OAuth |

### Fonctionnalités transverses

- **Architecture par connectors** : chaque intégration est un module autonome
- **Gestion centralisée des endpoints** : configuration en base de données
- **Interface d'administration** : ajout/modification/suppression des endpoints monitorés

---

## Hors périmètre (MVP)

- Alerting automatique (notifications push, email, SMS)
- Historique détaillé avec graphiques de tendance
- Multi-tenant / multi-organisation
- Application mobile native
- Intégration Slack/Teams pour notifications

---

## Parties prenantes

| Rôle | Responsabilité |
|------|---------------|
| Product Owner | Définition des priorités, validation fonctionnelle |
| Tech Lead | Architecture, choix techniques, revue de code |
| Développeurs | Implémentation backend et frontend |
| DevOps | Configuration infrastructure, CI/CD |
| Utilisateurs finaux | Opérateurs techniques, support N2/N3 |

---

## Contraintes

### Techniques
- Backend : **Node.js 22** avec API REST documentée (Swagger/OpenAPI)
- Frontend : **React + Vite** avec **MUI** (Material-UI)
- Base de données : **PostgreSQL**
- Authentification : **Azure App Registration** (OAuth2/OIDC)
- Architecture : **Mono-repo**

### Organisationnelles
- Respect des **best practices DDD** (Domain-Driven Design)
- Code versionné et documenté
- Tests automatisés

### Déploiement
- **Local** : docker-compose (PostgreSQL + Backend + Frontend)
- **Production** : Azure (Container Apps + PostgreSQL Flexible Server + Static Website)

---

## Hypothèses

1. Les API des systèmes externes (Talend, APIM, GitHub, etc.) sont accessibles depuis l'environnement de déploiement.
2. Les credentials et API keys seront fournis par l'équipe infrastructure.
3. L'Azure App Registration est déjà configurée ou sera configurée avant le déploiement.
4. Les utilisateurs finaux ont un compte Azure AD pour l'authentification.

---

## Dépendances externes

| Dépendance | Impact | Mitigation |
|------------|--------|------------|
| Disponibilité APIs externes | Critique | Gestion des erreurs, timeouts, retries |
| Azure AD | Critique | Fallback local pour dev |
| PostgreSQL Azure | Critique | Backup réguliers |

---

## Glossaire

| Terme | Définition |
|-------|-----------|
| **Connector** | Module autonome responsable de l'intégration avec un système externe |
| **Healthcheck** | Endpoint HTTP vérifiant l'état de santé d'un service |
| **Workflow** | Pipeline GitHub Actions exécutable |
| **Realm cache** | Cache Keycloak associé à un domaine d'authentification |
| **APIM** | Azure API Management |
| **TMC** | Talend Management Console |

---

## Documents connexes

- [02 - Personas et parcours utilisateurs](./02-personas-flows.md)
- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
- [04 - Architecture DDD](./04-architecture-ddd.md)
- [05 - Conception API](./05-api-design.md)
- [06 - Sécurité et authentification](./06-security-auth.md)
- [07 - Stockage et déploiement](./07-storage-deployment.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
- [09 - Critères d'acceptation](./09-acceptance-criteria.md)
- [10 - Risques et roadmap](./10-risks-roadmap.md)
