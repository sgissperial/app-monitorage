# 02 - Personas et parcours utilisateurs

## Personas

### Persona 1 : Opérateur DevOps

| Attribut | Description |
|----------|-------------|
| **Nom** | Marc, Opérateur DevOps |
| **Rôle** | Surveillance quotidienne, interventions de maintenance |
| **Expérience** | 3-5 ans en exploitation/DevOps |
| **Contexte** | Travaille en équipe, gère plusieurs environnements (dev, staging, prod) |
| **Frustrations** | Trop de consoles à surveiller, perte de temps à naviguer entre les outils |
| **Objectifs** | Avoir une vue d'ensemble rapide, agir vite en cas de problème |
| **Outils actuels** | Talend TMC, Azure Portal, GitHub, Keycloak Admin |

**Citation type** : *"Je veux voir en un coup d'œil si tout est vert, et agir immédiatement si quelque chose est rouge."*

---

### Persona 2 : Développeur Full-Stack

| Attribut | Description |
|----------|-------------|
| **Nom** | Sophie, Développeuse Full-Stack |
| **Rôle** | Développement, debug, support niveau 2 |
| **Expérience** | 2-4 ans en développement web |
| **Contexte** | Intervient ponctuellement sur des incidents liés au code |
| **Frustrations** | Doit demander aux DevOps pour certaines actions simples |
| **Objectifs** | Autonomie pour les actions non destructives (purge cache, relance workflow) |
| **Outils actuels** | VS Code, GitHub, Jira |

**Citation type** : *"Je veux pouvoir purger le cache Keycloak moi-même sans ouvrir un ticket."*

---

### Persona 3 : Support Technique N3

| Attribut | Description |
|----------|-------------|
| **Nom** | Julien, Support N3 |
| **Rôle** | Diagnostic avancé, résolution d'incidents complexes |
| **Expérience** | 5+ ans en support applicatif |
| **Contexte** | Intervient sur escalades, analyse les logs et états des services |
| **Frustrations** | Informations dispersées, temps perdu à collecter les données |
| **Objectifs** | Centraliser les informations pour un diagnostic rapide |
| **Outils actuels** | GLPI, Jira, consoles diverses |

**Citation type** : *"Quand je reçois un incident, je veux immédiatement voir l'état de tous les services concernés."*

---

### Persona 4 : Administrateur système

| Attribut | Description |
|----------|-------------|
| **Nom** | Claire, Administratrice système |
| **Rôle** | Configuration, gestion des accès, maintenance infrastructure |
| **Expérience** | 5+ ans en administration système |
| **Contexte** | Responsable de la configuration des endpoints à monitorer |
| **Frustrations** | Modifications de configuration nécessitent des déploiements |
| **Objectifs** | Gérer les endpoints monitorés sans intervention développeur |
| **Outils actuels** | Azure Portal, scripts PowerShell |

**Citation type** : *"Je veux ajouter un nouveau endpoint à surveiller sans toucher au code."*

---

## Parcours utilisateurs (User Flows)

### Flow 1 : Consultation du dashboard de monitoring

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Connexion  │────▶│  Dashboard   │────▶│ Vue d'ensemble  │
│  Azure AD   │     │  principal   │     │ (pavés colorés) │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Clic sur pavé   │
                                         │ = détails       │
                                         └─────────────────┘
```

**Étapes détaillées :**
1. L'utilisateur accède à l'URL de l'application
2. Redirection vers Azure AD pour authentification
3. Après connexion, affichage du dashboard principal
4. Les pavés de monitoring s'affichent avec leur état (vert/orange/rouge)
5. Clic sur un pavé pour voir les détails du service

**Points d'attention :**
- Chargement rapide des états (< 2 secondes)
- Rafraîchissement automatique configurable

---

### Flow 2 : Purge du cache Keycloak

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Menu       │────▶│  Page        │────▶│ Bouton "Purger" │
│  Caches     │     │  Keycloak    │     │                 │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Confirmation    │
                                         │ (modale)        │
                                         └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Exécution +     │
                                         │ Toaster succès  │
                                         └─────────────────┘
```

**Étapes détaillées :**
1. L'utilisateur clique sur "Caches" dans la toolbar
2. Sélection de "Keycloak" dans le sous-menu
3. Affichage de la page avec le bouton "Purger le cache"
4. Clic sur le bouton → modale de confirmation
5. Confirmation → appel API → toaster de succès/échec

**Points d'attention :**
- Confirmation obligatoire avant action
- Feedback visuel immédiat (spinner pendant l'exécution)
- Message d'erreur explicite si échec

---

### Flow 3 : Déclenchement d'un workflow GitHub Actions

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Menu       │────▶│  Liste des   │────▶│ Sélection       │
│  GitHub     │     │  workflows   │     │ workflow        │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Bouton          │
                                         │ "Déclencher"    │
                                         └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Toaster +       │
                                         │ Lien vers run   │
                                         └─────────────────┘
```

**Étapes détaillées :**
1. L'utilisateur clique sur "GitHub Actions" dans la toolbar
2. Liste des workflows disponibles s'affiche
3. Sélection d'un workflow
4. Clic sur "Déclencher"
5. Toaster avec statut + lien direct vers le run GitHub

**Points d'attention :**
- Afficher les paramètres du workflow si requis
- Lien cliquable vers le run pour suivi

---

### Flow 4 : Consultation des tickets Jira

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Menu       │────▶│  Page Jira   │────▶│ Onglet Sprint   │
│  Tickets    │     │              │     │ ou Backlog      │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Liste tickets   │
                                         │ + liens Jira    │
                                         └─────────────────┘
```

**Étapes détaillées :**
1. L'utilisateur clique sur "Tickets" puis "Jira"
2. Affichage par défaut : tickets du sprint actif
3. Possibilité de basculer sur le backlog
4. Chaque ticket est cliquable → ouvre Jira

**Points d'attention :**
- Pagination si beaucoup de tickets
- Filtre par statut possible

---

### Flow 5 : Administration des endpoints monitorés

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Menu       │────▶│  Liste       │────▶│ Ajout/Modif/    │
│  Admin      │     │  endpoints   │     │ Suppression     │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Formulaire      │
                                         │ configuration   │
                                         └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ Sauvegarde +    │
                                         │ Validation      │
                                         └─────────────────┘
```

**Étapes détaillées :**
1. L'administrateur accède au menu "Administration"
2. Liste des endpoints monitorés s'affiche
3. Actions disponibles : Ajouter / Modifier / Supprimer
4. Formulaire avec : URL, type, nom, intervalle de check
5. Validation et sauvegarde en base

**Points d'attention :**
- Validation de l'URL avant sauvegarde
- Test de connectivité optionnel
- Accès restreint aux administrateurs

---

## Matrice Persona × Fonctionnalités

| Fonctionnalité | DevOps | Développeur | Support N3 | Admin |
|----------------|--------|-------------|------------|-------|
| Dashboard monitoring | ✅ Principal | ✅ Fréquent | ✅ Fréquent | ✅ Occasionnel |
| Talend - Jobs erreur | ✅ Principal | ⚪ Rare | ✅ Fréquent | ⚪ Rare |
| APIM - Healthchecks | ✅ Principal | ✅ Fréquent | ✅ Fréquent | ⚪ Rare |
| GitHub Actions | ✅ Principal | ✅ Principal | ⚪ Rare | ⚪ Rare |
| Keycloak - Purge cache | ✅ Principal | ✅ Fréquent | ⚪ Rare | ⚪ Rare |
| Domain monitoring | ✅ Principal | ✅ Fréquent | ✅ Fréquent | ⚪ Rare |
| Cloudflare - Purge | ✅ Principal | ⚪ Rare | ⚪ Rare | ⚪ Rare |
| GLPI - Tickets | ⚪ Rare | ⚪ Rare | ✅ Principal | ⚪ Rare |
| Jira - Tickets | ✅ Fréquent | ✅ Principal | ✅ Fréquent | ⚪ Rare |
| Admin endpoints | ⚪ Rare | ⚪ Non | ⚪ Non | ✅ Principal |

**Légende :**
- ✅ Principal : Utilisation quotidienne, fonctionnalité clé
- ✅ Fréquent : Utilisation régulière
- ⚪ Rare : Utilisation occasionnelle
- ⚪ Non : Pas d'accès ou non pertinent

---

## Documents connexes

- [01 - Vue d'ensemble du produit](./01-overview-prd.md)
- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
