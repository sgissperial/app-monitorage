Tu es un **Product Manager senior** et un **Lead Software Architect**.
Ta mission est de produire un **Product Requirements Document (PRD)** complet, structuré et actionnable, destiné à être utilisé par un outil comme **claude-task-master** ou directement par un développeur (Claude Code / GitHub Copilot / Cursor).

⚠️ IMPORTANT  
- Tu dois **générer plusieurs fichiers Markdown distincts** (extension `.md`).
- Tous les documents doivent être **en français**.
- Chaque fichier doit avoir un **nom clair** et une **responsabilité précise**.
- Le contenu doit être **suffisamment détaillé pour être découpé automatiquement en tâches techniques**.

---

# 📌 Contexte général

Je souhaite développer une **application web interne de monitoring et d’actions opérationnelles**, permettant de centraliser :
- la surveillance de services applicatifs,
- la consultation de tickets,
- des actions techniques ponctuelles (purge de cache, déclenchement de workflows).

L’objectif est de remplacer ou compléter des consoles existantes (Talend TMC, APIM, Jira, etc.) par une **interface unique, rapide et orientée opérations**.

---

# 🎯 Objectifs du produit
- Centraliser l’état (up/down, erreurs) de plusieurs systèmes.
- Simplifier les actions récurrentes à fort impact opérationnel.
- Réduire le temps de diagnostic et d’intervention.
- Fournir une UI simple, lisible et actionnable.

---

# 🧩 Fonctionnalités attendues

## 1) Talend – Jobs en erreur (API Key)
- Récupérer et afficher les jobs Talend en erreur.
- Afficher : nom, environnement, date, message si disponible.
- Lien vers la console Talend si possible.

## 2) APIM – Healthchecks (API Key)
- Monitorer plusieurs endpoints healthcheck derrière APIM.
- Afficher état, statut HTTP, latence si possible.

## 3) GitHub Actions – Déclenchement de workflows (PAT)
- Lister les workflows disponibles.
- Déclencher un workflow manuellement via un bouton.
- Retour utilisateur immédiat (succès/échec + lien vers le run).

## 4) Keycloak – Purge du realm cache (credentials admin)
- Action “one-click” pour purger le cache.
- Retour visuel (toaster).

## 5) Domain Monitoring – endpoints healthcheck publics
- Monitorer des domaines applicatifs hors APIM.
- Vérification via endpoint healthcheck ouvert.
- Affichage sous forme de pavés colorés.

## 6) Cloudflare – Purge cache (API Key)
- Vider le cache Cloudflare pour une liste de domaines.
- Action immédiate avec confirmation visuelle.

## 7) GLPI – Tickets
- Lister les tickets de l’utilisateur.
- Affichage simple + lien vers GLPI.

## 8) Jira – Tickets
- Lister :
  - tickets du sprint actif,
  - backlog hors sprint.
- Liens directs vers Jira.

---

# 🔌 Architecture par connectors (OBLIGATOIRE)

Toutes les intégrations externes (Talend, APIM, GitHub, Keycloak, Cloudflare, GLPI, Jira, Domain Healthchecks) doivent être pensées comme des **connectors** :

- Un connector = un module autonome responsable d’un système externe.
- Interface commune :
  - `checkHealth()`
  - `listResources()`
  - `executeAction()`
- Gestion centralisée :
  - authentification,
  - erreurs,
  - timeouts,
  - retries.
- Objectif : permettre l’ajout futur de nouveaux systèmes sans modifier le cœur de l’application.

---

# 🌐 Gestion centralisée des endpoints monitorés (OBLIGATOIRE)

Les endpoints de monitoring **ne doivent pas être codés en dur**.

- Un module dédié permet de gérer :
  - domaines,
  - endpoints APIM,
  - types de checks.
- Les données sont stockées en base de données.
- Accès réservé à un rôle administrateur.
- Permet :
  - ajout,
  - modification,
  - suppression d’un endpoint monitoré.

---

# 🗄️ Stockage & base de données

- Base de données relationnelle : **PostgreSQL**.
- Utilisation :
  - configuration des connectors,
  - endpoints monitorés,
  - états de monitoring,
  - éventuellement historique et audit des actions.
- Le schéma doit rester simple et évolutif.
- L’usage de champs **JSON/JSONB** est autorisé pour la configuration dynamique.

---

# 🧱 Contraintes techniques

- Backend : **NodeJS 22**
  - API REST documentée via **Swagger / OpenAPI**.
- Frontend : **React + Vite**, UI avec **MUI**.
- Le Swagger doit permettre de générer automatiquement un **client React**.
- Authentification via **Azure App Registration** (OAuth2 / OIDC).
- Architecture **mono-repo**.
- Respect des **best practices DDD** :
  - côté backend,
  - côté frontend (modules, domain logic, services).

---

# 🐳 Environnements & déploiement

## Environnement de développement local
- Fournir un **docker-compose.yml** avec :
  - un service **PostgreSQL**,
  - un service **backend NodeJS**,
  - un service **frontend React**.
- Les services doivent pouvoir démarrer ensemble pour un setup développeur simple.
- Les secrets sont fournis via variables d’environnement.

## Environnement de production (Azure)
- La base de données PostgreSQL existe déjà sous forme de :
  **Azure Database for PostgreSQL – Flexible Server**.
- Le backend est déployé dans :
  **Azure App Container / Container Apps**.
- Le frontend est déployé dans :
  **Azure Storage Account – Static Website**.
- Le backend consomme la base PostgreSQL via variables d’environnement (connection string).
- Aucun composant de base de données n’est déployé via Docker en production.

---

# 🎨 Exigences UI / UX

- **Toolbar en haut** avec 4 familles :
  - Monitoring (sous-menu)
  - Tickets (sous-menu)
  - Caches (sous-menu)
  - GitHub Actions (accès direct à une page)
- Monitoring : pavés colorés avec nom du service en caption.
- Tickets : listes simples avec liens.
- Cache : clic → action → toaster.
- GitHub Actions : boutons de déclenchement.

---

# 📄 Structure attendue des fichiers PRD

Tu dois générer **plusieurs fichiers Markdown (.md)**, par exemple :

1. `01-overview-prd.md`
2. `02-personas-flows.md`
3. `03-functional-requirements.md`
4. `04-architecture-ddd.md`
5. `05-api-design.md`
6. `06-security-auth.md`
7. `07-storage-deployment.md`
8. `08-non-functional-requirements.md`
9. `09-acceptance-criteria.md`
10. `10-risks-roadmap.md`

Chaque fichier doit être autonome, clair et exploitable.

---

# ✍️ Règles de rédaction
- Français uniquement.
- Markdown propre, prêt à être versionné.
- Précis, structuré, testable.
- Hypothèses explicites quand nécessaire.
- Pas de sur-ingénierie inutile.

Ta réponse doit **uniquement contenir les fichiers PRD**, clairement séparés et nommés.
