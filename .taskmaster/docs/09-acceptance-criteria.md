# 09 - Critères d'acceptation

## Vue d'ensemble

Ce document définit les critères d'acceptation détaillés pour chaque fonctionnalité, permettant de valider que l'implémentation répond aux exigences.

---

## Format des critères

Chaque critère suit le format **Given / When / Then** (Étant donné / Quand / Alors) :
- **Given** : Contexte initial, prérequis
- **When** : Action déclencheur
- **Then** : Résultat attendu

---

## AC-001 : Authentification utilisateur

### AC-001.1 : Connexion réussie

```gherkin
Étant donné que l'utilisateur n'est pas connecté
  Et qu'il possède un compte Azure AD valide
Quand il accède à l'application
Alors il est redirigé vers la page de connexion Azure AD
  Et après authentification réussie, il est redirigé vers le dashboard
  Et son nom est affiché dans la toolbar
```

### AC-001.2 : Déconnexion

```gherkin
Étant donné que l'utilisateur est connecté
Quand il clique sur "Déconnexion"
Alors il est déconnecté de l'application
  Et il est redirigé vers la page de connexion
  Et ses tokens sont supprimés
```

### AC-001.3 : Session expirée

```gherkin
Étant donné que l'utilisateur est connecté
  Et que son token a expiré
Quand il effectue une action
Alors le système tente un refresh silencieux du token
  Et si le refresh échoue, l'utilisateur est redirigé vers la connexion
```

---

## AC-002 : Dashboard de monitoring

### AC-002.1 : Affichage initial

```gherkin
Étant donné que l'utilisateur est connecté
Quand il accède au dashboard de monitoring
Alors il voit tous les endpoints actifs sous forme de pavés colorés
  Et chaque pavé affiche le nom du service
  Et la couleur reflète l'état (vert=UP, orange=DEGRADED, rouge=DOWN)
  Et le chargement initial prend moins de 2 secondes
```

### AC-002.2 : Rafraîchissement automatique

```gherkin
Étant donné que le dashboard est affiché
Quand l'intervalle de rafraîchissement est atteint (défaut: 60s)
Alors les statuts des endpoints sont mis à jour
  Et les pavés changent de couleur si l'état a changé
  Et aucun rechargement complet de la page n'a lieu
```

### AC-002.3 : Détails d'un endpoint

```gherkin
Étant donné que le dashboard affiche des endpoints
Quand l'utilisateur clique sur un pavé
Alors un panneau de détails s'affiche avec :
  - Nom complet de l'endpoint
  - URL
  - Statut actuel
  - Code HTTP
  - Latence
  - Timestamp du dernier check
  - Historique des 10 derniers checks
```

### AC-002.4 : Indicateur de problème

```gherkin
Étant donné qu'un ou plusieurs endpoints sont DOWN ou DEGRADED
Quand l'utilisateur consulte le dashboard
Alors un compteur résumé est affiché en haut
  Et les endpoints en erreur sont mis en évidence (bordure, icône)
  Et les endpoints DOWN sont affichés en premier (tri par criticité)
```

---

## AC-003 : Monitoring Talend

### AC-003.1 : Liste des jobs en erreur

```gherkin
Étant donné que le connector Talend est configuré et actif
Quand l'utilisateur accède à la page Talend
Alors il voit la liste des jobs en erreur
  Et chaque job affiche : nom, environnement, date, message d'erreur
  Et les jobs sont triés par date (plus récent en premier)
  Et un lien vers Talend TMC est disponible
```

### AC-003.2 : Aucun job en erreur

```gherkin
Étant donné que le connector Talend est configuré et actif
  Et qu'aucun job n'est en erreur
Quand l'utilisateur accède à la page Talend
Alors un message "Aucun job en erreur" est affiché
  Et une icône de succès est visible
```

### AC-003.3 : Connector non configuré

```gherkin
Étant donné que le connector Talend n'est pas configuré
Quand l'utilisateur accède à la page Talend
Alors un message d'avertissement est affiché
  Et l'utilisateur admin voit un lien vers la configuration
```

---

## AC-004 : Monitoring APIM

### AC-004.1 : Affichage des healthchecks

```gherkin
Étant donné que des endpoints APIM sont configurés
Quand l'utilisateur accède à la section APIM
Alors il voit la liste des endpoints avec leur statut
  Et chaque endpoint affiche : nom, URL, statut, code HTTP, latence
  Et les endpoints sont groupés par catégorie si applicable
```

### AC-004.2 : Endpoint dégradé

```gherkin
Étant donné qu'un endpoint APIM répond en plus de 2 secondes
Quand le check de santé est effectué
Alors l'endpoint est marqué comme DEGRADED (orange)
  Et la latence est affichée en surbrillance
```

---

## AC-005 : GitHub Actions

### AC-005.1 : Liste des workflows

```gherkin
Étant donné que le connector GitHub est configuré
Quand l'utilisateur accède à la page GitHub Actions
Alors il voit la liste des workflows disponibles
  Et chaque workflow affiche : nom, repository, état du dernier run, date
  Et seuls les workflows déclenchables manuellement sont listés
```

### AC-005.2 : Déclenchement d'un workflow simple

```gherkin
Étant donné que l'utilisateur a les permissions "actions:execute"
  Et qu'un workflow sans inputs est sélectionné
Quand l'utilisateur clique sur "Déclencher"
Alors une confirmation est demandée
  Et après confirmation, le workflow est déclenché via API GitHub
  Et un toaster affiche "Workflow déclenché avec succès"
  Et un lien vers le run GitHub est fourni
```

### AC-005.3 : Déclenchement d'un workflow avec inputs

```gherkin
Étant donné que l'utilisateur a les permissions "actions:execute"
  Et qu'un workflow avec inputs est sélectionné
Quand l'utilisateur clique sur "Déclencher"
Alors un formulaire s'affiche avec les inputs requis
  Et l'utilisateur remplit les champs
  Et après soumission, le workflow est déclenché
  Et les inputs sont envoyés à GitHub
```

### AC-005.4 : Échec de déclenchement

```gherkin
Étant donné que l'utilisateur tente de déclencher un workflow
  Et que l'API GitHub renvoie une erreur
Quand la réponse d'erreur est reçue
Alors un toaster rouge affiche le message d'erreur
  Et l'action est loggée avec le résultat FAILURE
```

---

## AC-006 : Purge cache Keycloak

### AC-006.1 : Purge réussie

```gherkin
Étant donné que l'utilisateur a les permissions "actions:execute"
  Et que le connector Keycloak est configuré
Quand l'utilisateur clique sur "Purger le cache Keycloak"
Alors une modale de confirmation s'affiche
  Et après confirmation, l'API Keycloak est appelée
  Et un toaster vert affiche "Cache purgé avec succès"
  Et l'action est loggée (utilisateur, timestamp, résultat)
```

### AC-006.2 : Purge échouée

```gherkin
Étant donné que l'utilisateur tente de purger le cache
  Et que Keycloak renvoie une erreur
Quand la réponse d'erreur est reçue
Alors un toaster rouge affiche le message d'erreur
  Et l'utilisateur peut réessayer
  Et l'action est loggée avec le résultat FAILURE
```

---

## AC-007 : Purge cache Cloudflare

### AC-007.1 : Sélection de zone et purge

```gherkin
Étant donné que l'utilisateur a les permissions "actions:execute"
  Et que plusieurs zones Cloudflare sont configurées
Quand l'utilisateur accède à la page Cloudflare
Alors il voit la liste des zones disponibles
  Et il peut sélectionner une ou plusieurs zones
  Et il peut choisir "Purger tout" ou entrer des URLs spécifiques
  Et après confirmation, la purge est exécutée
```

### AC-007.2 : Confirmation multi-zones

```gherkin
Étant donné que l'utilisateur a sélectionné 3 zones à purger
Quand il clique sur "Purger"
Alors la modale de confirmation liste les 3 zones sélectionnées
  Et l'utilisateur doit confirmer explicitement
```

---

## AC-008 : Tickets GLPI

### AC-008.1 : Affichage des tickets

```gherkin
Étant donné que le connector GLPI est configuré
  Et que l'utilisateur a des tickets associés
Quand l'utilisateur accède à la page GLPI
Alors il voit la liste de ses tickets
  Et chaque ticket affiche : numéro, titre, statut, priorité, date
  Et un lien vers GLPI est disponible pour chaque ticket
```

### AC-008.2 : Filtrage par statut

```gherkin
Étant donné que l'utilisateur consulte ses tickets GLPI
Quand il applique un filtre par statut (ex: "En cours")
Alors seuls les tickets avec ce statut sont affichés
  Et le compteur de résultats est mis à jour
```

---

## AC-009 : Tickets Jira

### AC-009.1 : Tickets du sprint actif

```gherkin
Étant donné que le connector Jira est configuré
  Et qu'un sprint est actif
Quand l'utilisateur accède à la page Jira
Alors il voit les tickets du sprint actif
  Et le nom du sprint et ses dates sont affichés
  Et chaque ticket affiche : clé, titre, statut, assigné, type
  Et un lien vers Jira est disponible
```

### AC-009.2 : Backlog hors sprint

```gherkin
Étant donné que l'utilisateur consulte la page Jira
Quand il sélectionne l'onglet "Backlog"
Alors il voit les tickets non assignés à un sprint
  Et la pagination est disponible si > 50 tickets
```

---

## AC-010 : Administration des endpoints

### AC-010.1 : Accès restreint

```gherkin
Étant donné que l'utilisateur n'a pas le rôle "admin"
Quand il tente d'accéder à /admin/endpoints
Alors il reçoit une erreur 403 Forbidden
  Et il est redirigé vers le dashboard
```

### AC-010.2 : Création d'un endpoint

```gherkin
Étant donné que l'utilisateur admin accède à l'administration
Quand il clique sur "Ajouter un endpoint"
Alors un formulaire s'affiche avec les champs :
  - Nom (obligatoire)
  - Type (APIM/DOMAIN/TALEND)
  - URL (obligatoire, validée)
  - Intervalle de check
  - Timeout
  - Code HTTP attendu
  - Headers (JSON)
  - Actif/Inactif
Et après validation, l'endpoint est créé
Et il apparaît immédiatement dans le monitoring
```

### AC-010.3 : Test de connectivité

```gherkin
Étant donné que l'utilisateur admin remplit le formulaire d'endpoint
Quand il clique sur "Tester la connexion"
Alors un appel HTTP est effectué vers l'URL
  Et le résultat (succès/échec, code HTTP, latence) est affiché
  Et l'utilisateur peut décider de sauvegarder ou modifier
```

### AC-010.4 : Modification d'un endpoint

```gherkin
Étant donné qu'un endpoint existe
Quand l'utilisateur admin modifie ses propriétés et sauvegarde
Alors les modifications sont appliquées immédiatement
  Et le monitoring utilise les nouvelles valeurs
  Et l'action est loggée
```

### AC-010.5 : Suppression avec confirmation

```gherkin
Étant donné qu'un endpoint existe
Quand l'utilisateur admin clique sur "Supprimer"
Alors une modale de confirmation s'affiche
  Et le nom de l'endpoint est répété pour éviter les erreurs
  Et après confirmation, l'endpoint est supprimé
  Et l'historique associé est conservé 30 jours
```

---

## AC-011 : Interface utilisateur

### AC-011.1 : Navigation toolbar

```gherkin
Étant donné que l'utilisateur est connecté
Quand il consulte la toolbar
Alors il voit les menus : Monitoring, Tickets, Caches, GitHub Actions
  Et les sous-menus se déplient au survol/clic
  Et la page active est mise en évidence
```

### AC-011.2 : Toaster notifications

```gherkin
Étant donné qu'une action est exécutée
Quand l'action se termine (succès ou échec)
Alors un toaster s'affiche dans le coin supérieur droit
  Et la couleur reflète le résultat (vert/rouge/orange)
  Et le toaster disparaît après 5 secondes (succès) ou reste visible (erreur)
  Et l'utilisateur peut le fermer manuellement
```

### AC-011.3 : Responsive design

```gherkin
Étant donné que l'utilisateur accède à l'application sur tablette
Quand la largeur d'écran est entre 768px et 1200px
Alors l'interface s'adapte (colonnes réorganisées)
  Et toutes les fonctionnalités restent accessibles
  Et la navigation utilise un menu hamburger si nécessaire
```

---

## AC-012 : Logs d'actions

### AC-012.1 : Consultation des logs (admin)

```gherkin
Étant donné que l'utilisateur admin accède aux logs d'actions
Quand la page se charge
Alors il voit les 100 dernières actions
  Et chaque log affiche : action, cible, utilisateur, résultat, date
  Et il peut filtrer par type d'action, utilisateur, période
```

### AC-012.2 : Traçabilité complète

```gherkin
Étant donné qu'une action sensible est exécutée (purge, workflow)
Quand l'action est terminée
Alors un log est créé avec :
  - Type d'action
  - Cible (connector, endpoint)
  - ID utilisateur et email
  - Rôles de l'utilisateur
  - Adresse IP
  - User agent
  - Paramètres de l'action
  - Résultat (SUCCESS/FAILURE)
  - Message d'erreur si applicable
  - Timestamp
```

---

## AC-013 : Gestion des erreurs

### AC-013.1 : Erreur réseau

```gherkin
Étant donné que l'utilisateur effectue une action
  Et que le réseau est indisponible
Quand l'appel API échoue
Alors un message d'erreur explicite est affiché
  Et l'utilisateur peut réessayer
  Et l'interface reste stable
```

### AC-013.2 : Erreur serveur 500

```gherkin
Étant donné que l'API renvoie une erreur 500
Quand la réponse est reçue
Alors un message générique est affiché ("Une erreur est survenue")
  Et les détails techniques ne sont pas exposés à l'utilisateur
  Et l'erreur est loggée côté serveur avec le requestId
```

### AC-013.3 : Connector indisponible

```gherkin
Étant donné qu'un connector externe est indisponible
Quand l'utilisateur accède à la fonctionnalité associée
Alors un message spécifique est affiché ("Service temporairement indisponible")
  Et les autres fonctionnalités restent accessibles
  Et le dernier état connu est affiché si disponible
```

---

## Matrice de couverture

| Fonctionnalité | Critères d'acceptation | Priorité |
|----------------|------------------------|----------|
| Authentification | AC-001.1, AC-001.2, AC-001.3 | Critique |
| Dashboard monitoring | AC-002.1, AC-002.2, AC-002.3, AC-002.4 | Haute |
| Talend | AC-003.1, AC-003.2, AC-003.3 | Haute |
| APIM | AC-004.1, AC-004.2 | Haute |
| GitHub Actions | AC-005.1, AC-005.2, AC-005.3, AC-005.4 | Haute |
| Keycloak | AC-006.1, AC-006.2 | Moyenne |
| Cloudflare | AC-007.1, AC-007.2 | Moyenne |
| GLPI | AC-008.1, AC-008.2 | Moyenne |
| Jira | AC-009.1, AC-009.2 | Haute |
| Admin endpoints | AC-010.1 à AC-010.5 | Haute |
| UI/UX | AC-011.1, AC-011.2, AC-011.3 | Haute |
| Logs | AC-012.1, AC-012.2 | Moyenne |
| Erreurs | AC-013.1, AC-013.2, AC-013.3 | Haute |

---

## Documents connexes

- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
- [08 - Exigences non fonctionnelles](./08-non-functional-requirements.md)
- [10 - Risques et roadmap](./10-risks-roadmap.md)
