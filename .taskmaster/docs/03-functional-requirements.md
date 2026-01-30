# 03 - Exigences fonctionnelles

## Vue d'ensemble

Ce document détaille les exigences fonctionnelles de l'application App Monitorage, organisées par domaine métier.

---

## F1 - Module Monitoring Talend

### F1.1 - Récupération des jobs en erreur

| ID | F1.1 |
|----|------|
| **Titre** | Afficher les jobs Talend en erreur |
| **Description** | Le système récupère et affiche la liste des jobs Talend actuellement en erreur via l'API Talend TMC. |
| **Priorité** | Haute |
| **Acteur** | Opérateur DevOps, Support N3 |

**Données affichées :**
- Nom du job
- Environnement (dev, staging, prod)
- Date/heure de l'erreur
- Message d'erreur (si disponible via API)
- Lien vers la console Talend TMC

**Règles métier :**
- RG1.1.1 : Les jobs sont triés par date d'erreur (plus récent en premier)
- RG1.1.2 : Un job peut apparaître plusieurs fois si plusieurs exécutions en erreur
- RG1.1.3 : L'affichage se rafraîchit automatiquement (intervalle configurable)

**Connecteur requis :**
- Type : TalendConnector
- Authentification : API Key
- Méthodes : `listResources()` pour récupérer les jobs en erreur

---

## F2 - Module Monitoring APIM

### F2.1 - Healthchecks des endpoints APIM

| ID | F2.1 |
|----|------|
| **Titre** | Monitorer les endpoints healthcheck APIM |
| **Description** | Le système vérifie périodiquement l'état de santé des endpoints exposés derrière Azure API Management. |
| **Priorité** | Haute |
| **Acteur** | Opérateur DevOps, Développeur |

**Données affichées :**
- Nom du service/endpoint
- URL de l'endpoint
- État : UP (vert) / DOWN (rouge) / DEGRADED (orange)
- Code HTTP de la dernière réponse
- Latence (ms) si disponible
- Timestamp du dernier check

**Règles métier :**
- RG2.1.1 : Un endpoint est UP si le code HTTP est 2xx
- RG2.1.2 : Un endpoint est DEGRADED si le temps de réponse > seuil configurable
- RG2.1.3 : Un endpoint est DOWN si timeout ou code HTTP 5xx
- RG2.1.4 : Les endpoints sont configurés en base de données (cf. F10)

**Connecteur requis :**
- Type : ApimConnector
- Authentification : API Key (subscription key)
- Méthodes : `checkHealth()` pour chaque endpoint

---

## F3 - Module GitHub Actions

### F3.1 - Lister les workflows disponibles

| ID | F3.1 |
|----|------|
| **Titre** | Afficher la liste des workflows GitHub Actions |
| **Description** | Le système récupère et affiche les workflows disponibles dans les repositories configurés. |
| **Priorité** | Haute |
| **Acteur** | Opérateur DevOps, Développeur |

**Données affichées :**
- Nom du workflow
- Repository associé
- État du dernier run (success/failure/in_progress)
- Date du dernier run

**Règles métier :**
- RG3.1.1 : Seuls les workflows marqués `workflow_dispatch` sont listés (déclenchables manuellement)
- RG3.1.2 : Les repositories sont configurés en base de données

### F3.2 - Déclencher un workflow

| ID | F3.2 |
|----|------|
| **Titre** | Déclencher un workflow manuellement |
| **Description** | L'utilisateur peut déclencher l'exécution d'un workflow GitHub Actions via un bouton. |
| **Priorité** | Haute |
| **Acteur** | Opérateur DevOps, Développeur |

**Comportement :**
1. Clic sur le bouton "Déclencher"
2. Si le workflow a des inputs : affichage d'un formulaire
3. Confirmation de l'action
4. Appel API GitHub pour déclencher le workflow
5. Affichage du résultat : succès/échec + lien vers le run

**Règles métier :**
- RG3.2.1 : Le lien vers le run GitHub est affiché immédiatement après déclenchement
- RG3.2.2 : Les inputs du workflow sont récupérés dynamiquement
- RG3.2.3 : Un log de l'action est conservé (utilisateur, workflow, timestamp)

**Connecteur requis :**
- Type : GitHubConnector
- Authentification : Personal Access Token (PAT)
- Méthodes : `listResources()`, `executeAction()`

---

## F4 - Module Keycloak

### F4.1 - Purge du cache realm

| ID | F4.1 |
|----|------|
| **Titre** | Purger le cache Keycloak |
| **Description** | Action one-click pour vider le cache d'un realm Keycloak. |
| **Priorité** | Moyenne |
| **Acteur** | Opérateur DevOps, Développeur |

**Comportement :**
1. Affichage du bouton "Purger le cache"
2. Clic → modale de confirmation
3. Confirmation → appel API Keycloak
4. Toaster de succès ou d'erreur

**Règles métier :**
- RG4.1.1 : La confirmation est obligatoire avant exécution
- RG4.1.2 : Le realm ciblé est configurable
- RG4.1.3 : L'action est loguée (utilisateur, timestamp, résultat)

**Connecteur requis :**
- Type : KeycloakConnector
- Authentification : Credentials admin (client_id/client_secret ou user/password)
- Méthodes : `executeAction()` pour la purge

---

## F5 - Module Domain Monitoring

### F5.1 - Healthcheck des domaines publics

| ID | F5.1 |
|----|------|
| **Titre** | Monitorer les domaines applicatifs hors APIM |
| **Description** | Vérification périodique de l'état de domaines via leurs endpoints healthcheck publics. |
| **Priorité** | Haute |
| **Acteur** | Opérateur DevOps, Support N3 |

**Données affichées :**
- Nom du domaine
- URL du healthcheck
- État : UP / DOWN / DEGRADED
- Code HTTP
- Latence (ms)

**Règles métier :**
- RG5.1.1 : Les domaines sont configurés en base de données
- RG5.1.2 : L'intervalle de vérification est configurable par domaine
- RG5.1.3 : Aucune authentification requise (endpoints publics)

**Connecteur requis :**
- Type : DomainHealthConnector
- Authentification : Aucune
- Méthodes : `checkHealth()`

---

## F6 - Module Cloudflare

### F6.1 - Purge du cache Cloudflare

| ID | F6.1 |
|----|------|
| **Titre** | Vider le cache Cloudflare |
| **Description** | Action pour purger le cache Cloudflare d'un ou plusieurs domaines. |
| **Priorité** | Moyenne |
| **Acteur** | Opérateur DevOps |

**Comportement :**
1. Liste des domaines Cloudflare configurés
2. Sélection d'un ou plusieurs domaines
3. Clic sur "Purger" → confirmation
4. Exécution → toaster de résultat

**Règles métier :**
- RG6.1.1 : Possibilité de purger tout le cache ou des URLs spécifiques
- RG6.1.2 : Confirmation obligatoire
- RG6.1.3 : Action loguée

**Connecteur requis :**
- Type : CloudflareConnector
- Authentification : API Key + Zone ID
- Méthodes : `executeAction()` pour la purge

---

## F7 - Module GLPI

### F7.1 - Liste des tickets GLPI

| ID | F7.1 |
|----|------|
| **Titre** | Afficher les tickets GLPI de l'utilisateur |
| **Description** | Récupération et affichage des tickets GLPI associés à l'utilisateur connecté. |
| **Priorité** | Moyenne |
| **Acteur** | Support N3 |

**Données affichées :**
- Numéro du ticket
- Titre
- Statut (Nouveau, En cours, Résolu, Fermé)
- Date de création
- Priorité
- Lien vers GLPI

**Règles métier :**
- RG7.1.1 : Affichage des tickets où l'utilisateur est demandeur ou assigné
- RG7.1.2 : Tri par date de création (plus récent en premier)
- RG7.1.3 : Filtrage possible par statut

**Connecteur requis :**
- Type : GlpiConnector
- Authentification : API Key + User Token
- Méthodes : `listResources()`

---

## F8 - Module Jira

### F8.1 - Tickets du sprint actif

| ID | F8.1 |
|----|------|
| **Titre** | Afficher les tickets du sprint en cours |
| **Description** | Récupération des tickets Jira appartenant au sprint actif du projet configuré. |
| **Priorité** | Haute |
| **Acteur** | Développeur, DevOps, Support N3 |

**Données affichées :**
- Clé du ticket (ex: PROJ-123)
- Titre
- Statut
- Assigné
- Type (Bug, Story, Task)
- Lien vers Jira

**Règles métier :**
- RG8.1.1 : Le projet et le board Jira sont configurables
- RG8.1.2 : Seul le sprint actif est affiché par défaut
- RG8.1.3 : Tri par priorité puis par clé

### F8.2 - Backlog hors sprint

| ID | F8.2 |
|----|------|
| **Titre** | Afficher le backlog hors sprint |
| **Description** | Récupération des tickets Jira non assignés à un sprint. |
| **Priorité** | Moyenne |
| **Acteur** | Développeur, DevOps |

**Données affichées :**
- Identiques à F8.1

**Règles métier :**
- RG8.2.1 : Tickets avec sprint = null ou vide
- RG8.2.2 : Pagination si > 50 tickets

**Connecteur requis :**
- Type : JiraConnector
- Authentification : API Key ou OAuth
- Méthodes : `listResources()`

---

## F9 - Interface utilisateur

### F9.1 - Toolbar et navigation

| ID | F9.1 |
|----|------|
| **Titre** | Toolbar principale avec navigation |
| **Description** | Barre de navigation en haut de l'écran avec accès aux différentes sections. |
| **Priorité** | Haute |
| **Acteur** | Tous |

**Structure de la toolbar :**
```
┌─────────────────────────────────────────────────────────────┐
│  Logo │ Monitoring ▼ │ Tickets ▼ │ Caches ▼ │ GitHub Actions │
└─────────────────────────────────────────────────────────────┘
```

**Sous-menus :**
- **Monitoring** : Talend, APIM, Domaines
- **Tickets** : GLPI, Jira
- **Caches** : Keycloak, Cloudflare
- **GitHub Actions** : Accès direct (pas de sous-menu)

**Règles métier :**
- RG9.1.1 : La page active est mise en évidence visuellement
- RG9.1.2 : Les sous-menus se ferment au clic en dehors

### F9.2 - Pavés de monitoring colorés

| ID | F9.2 |
|----|------|
| **Titre** | Affichage des états sous forme de pavés colorés |
| **Description** | Chaque service monitoré est représenté par un pavé coloré indiquant son état. |
| **Priorité** | Haute |
| **Acteur** | Tous |

**Design :**
- **Vert** : Service UP
- **Orange** : Service DEGRADED
- **Rouge** : Service DOWN
- **Gris** : État inconnu / en cours de check

**Règles métier :**
- RG9.2.1 : Le nom du service est affiché en caption sous le pavé
- RG9.2.2 : Clic sur un pavé → affichage des détails
- RG9.2.3 : Animation subtile lors du rafraîchissement

### F9.3 - Notifications toaster

| ID | F9.3 |
|----|------|
| **Titre** | Retour utilisateur via toaster |
| **Description** | Les actions utilisateur génèrent un feedback visuel sous forme de notification. |
| **Priorité** | Haute |
| **Acteur** | Tous |

**Types de toaster :**
- **Succès** (vert) : Action exécutée avec succès
- **Erreur** (rouge) : Échec de l'action
- **Information** (bleu) : Information neutre
- **Avertissement** (orange) : Action avec réserve

**Règles métier :**
- RG9.3.1 : Disparition automatique après 5 secondes (configurable)
- RG9.3.2 : Possibilité de fermer manuellement
- RG9.3.3 : Empilement si plusieurs notifications

---

## F10 - Administration des endpoints

### F10.1 - Gestion des endpoints monitorés

| ID | F10.1 |
|----|------|
| **Titre** | CRUD des endpoints de monitoring |
| **Description** | Interface d'administration permettant de gérer les endpoints à monitorer. |
| **Priorité** | Haute |
| **Acteur** | Administrateur |

**Fonctionnalités :**
- Lister tous les endpoints configurés
- Ajouter un nouvel endpoint
- Modifier un endpoint existant
- Supprimer un endpoint
- Tester la connectivité d'un endpoint

**Champs d'un endpoint :**
| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| name | string | Oui | Nom d'affichage |
| url | string | Oui | URL du healthcheck |
| type | enum | Oui | APIM / DOMAIN / TALEND |
| checkInterval | number | Non | Intervalle en secondes (défaut: 60) |
| timeout | number | Non | Timeout en ms (défaut: 5000) |
| expectedStatus | number | Non | Code HTTP attendu (défaut: 200) |
| headers | JSON | Non | Headers personnalisés |
| enabled | boolean | Oui | Actif ou non |

**Règles métier :**
- RG10.1.1 : Accès restreint au rôle "admin"
- RG10.1.2 : Validation de l'URL (format valide)
- RG10.1.3 : Test de connectivité optionnel avant sauvegarde
- RG10.1.4 : Modification effective immédiatement après sauvegarde

---

## F11 - Configuration des connectors

### F11.1 - Paramétrage des connectors

| ID | F11.1 |
|----|------|
| **Titre** | Configuration des connectors en base de données |
| **Description** | Chaque connector peut être configuré via l'interface d'administration. |
| **Priorité** | Moyenne |
| **Acteur** | Administrateur |

**Champs de configuration (par connector) :**
| Champ | Type | Description |
|-------|------|-------------|
| connectorType | enum | Type du connector |
| baseUrl | string | URL de base de l'API |
| credentials | JSON (chiffré) | Credentials (API key, tokens, etc.) |
| enabled | boolean | Actif ou non |
| options | JSON | Options spécifiques au connector |

**Règles métier :**
- RG11.1.1 : Les credentials sont stockés chiffrés en base
- RG11.1.2 : Possibilité de tester la connexion
- RG11.1.3 : Logs d'audit pour les modifications

---

## Récapitulatif des priorités

| Priorité | Fonctionnalités |
|----------|-----------------|
| **Haute** | F1, F2, F3, F5, F8.1, F9.1, F9.2, F9.3, F10.1 |
| **Moyenne** | F4, F6, F7, F8.2, F11.1 |
| **Basse** | - |

---

## Documents connexes

- [02 - Personas et parcours utilisateurs](./02-personas-flows.md)
- [04 - Architecture DDD](./04-architecture-ddd.md)
- [09 - Critères d'acceptation](./09-acceptance-criteria.md)
