# 10 - Risques et roadmap

## Vue d'ensemble

Ce document identifie les risques du projet et propose une roadmap de développement par phases.

---

## Analyse des risques

### Risques techniques

#### RISK-T01 : Disponibilité des APIs externes

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Moyenne |
| **Impact** | Élevé |
| **Description** | Les APIs des systèmes externes (Talend, GitHub, Keycloak, etc.) peuvent être indisponibles, modifiées ou limitées en termes de rate limiting. |

**Mitigations :**
- Implémentation de circuit breakers
- Cache des dernières données connues
- Retry avec backoff exponentiel
- Monitoring proactif des statuts des APIs
- Documentation des comportements en mode dégradé

---

#### RISK-T02 : Complexité de l'intégration Azure AD

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Moyenne |
| **Impact** | Élevé |
| **Description** | La configuration OAuth2/OIDC avec Azure AD peut être complexe, notamment pour la gestion des tokens, le refresh silencieux et les rôles. |

**Mitigations :**
- POC d'authentification en début de projet
- Utilisation de librairies éprouvées (MSAL)
- Documentation détaillée de la configuration Azure
- Tests d'intégration automatisés

---

#### RISK-T03 : Performance du monitoring temps réel

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Basse |
| **Impact** | Moyen |
| **Description** | Le rafraîchissement périodique de nombreux endpoints peut générer une charge importante et des latences. |

**Mitigations :**
- Vérification parallèle des endpoints
- Pooling de connexions HTTP
- Limitation du nombre d'endpoints (500 max)
- Monitoring de la performance des checks

---

#### RISK-T04 : Compatibilité des navigateurs

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Basse |
| **Impact** | Faible |
| **Description** | Certaines fonctionnalités React/MUI peuvent avoir un comportement différent selon les navigateurs. |

**Mitigations :**
- Tests sur les navigateurs cibles (Chrome, Firefox, Edge)
- Utilisation de polyfills si nécessaire
- CI avec tests cross-browser

---

### Risques organisationnels

#### RISK-O01 : Dépendance aux équipes infrastructure

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Moyenne |
| **Impact** | Moyen |
| **Description** | La fourniture des credentials (API keys, PAT, etc.) et la configuration Azure dépendent d'autres équipes. |

**Mitigations :**
- Identification précoce des dépendances
- Communication claire des besoins
- Environnement de dev avec mocks si nécessaire
- Documentation des credentials requises

---

#### RISK-O02 : Adoption par les utilisateurs

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Basse |
| **Impact** | Moyen |
| **Description** | Les utilisateurs peuvent préférer leurs outils habituels malgré la fragmentation. |

**Mitigations :**
- Implication des utilisateurs dans la conception
- Interface simple et intuitive
- Formation minimale grâce à l'UX
- Valeur ajoutée claire (gain de temps)

---

### Risques sécurité

#### RISK-S01 : Exposition des credentials

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Basse |
| **Impact** | Critique |
| **Description** | Les credentials des connectors pourraient être exposées via logs, erreurs ou failles. |

**Mitigations :**
- Chiffrement AES-256 en base
- Aucun credential dans les logs
- Audit de sécurité avant production
- Rotation régulière des secrets
- Principe du moindre privilège

---

#### RISK-S02 : Élévation de privilèges

| Attribut | Valeur |
|----------|--------|
| **Probabilité** | Basse |
| **Impact** | Élevé |
| **Description** | Un utilisateur pourrait accéder à des fonctionnalités non autorisées. |

**Mitigations :**
- Validation des permissions côté serveur systématique
- Tests d'autorisation automatisés
- Audit des rôles Azure AD
- Logs d'accès pour détection d'anomalies

---

### Matrice des risques

```
Impact
  ▲
  │
Critique │                          RISK-S01
  │
Élevé    │     RISK-T02      RISK-T01           RISK-S02
  │
Moyen    │     RISK-O02      RISK-T03    RISK-O01
  │
Faible   │                   RISK-T04
  │
         └─────────────────────────────────────────▶ Probabilité
              Basse        Moyenne       Haute
```

---

## Roadmap de développement

### Vue d'ensemble des phases

```
Phase 1: Fondations          Phase 2: Core Features       Phase 3: Extensions
────────────────────────     ────────────────────────     ────────────────────
  - Setup projet               - Dashboard monitoring       - GLPI connector
  - Architecture DDD           - Talend connector           - Jira connector
  - Auth Azure AD              - APIM connector             - Cloudflare connector
  - Base de données            - Domain health              - Historique avancé
  - CI/CD                      - Admin endpoints            - Améliorations UX
                               - GitHub Actions
                               - Keycloak
```

---

### Phase 1 : Fondations (Sprint 1-2)

**Objectif :** Mettre en place l'infrastructure technique et l'authentification.

#### Sprint 1 : Setup et architecture

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Initialiser le mono-repo (packages/backend, frontend, shared) | Haute | 0.5j |
| Configurer TypeScript, ESLint, Prettier | Haute | 0.5j |
| Créer le Dockerfile.dev et docker-compose.yml | Haute | 1j |
| Implémenter la structure DDD backend | Haute | 1j |
| Créer le schéma PostgreSQL initial | Haute | 0.5j |
| Configurer les migrations | Moyenne | 0.5j |

**Livrables :**
- Projet buildable et exécutable en local
- Structure de code conforme au DDD
- Base de données fonctionnelle

#### Sprint 2 : Authentification et API de base

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Configurer Azure App Registration | Haute | 1j |
| Implémenter l'auth backend (middleware JWT) | Haute | 1j |
| Implémenter l'auth frontend (MSAL) | Haute | 1j |
| Créer le endpoint GET /auth/me | Haute | 0.5j |
| Configurer Swagger/OpenAPI | Moyenne | 0.5j |
| Créer le layout principal (toolbar, routing) | Haute | 1j |

**Livrables :**
- Authentification fonctionnelle bout en bout
- Documentation API Swagger accessible
- Shell de l'application frontend

---

### Phase 2 : Core Features (Sprint 3-5)

**Objectif :** Développer les fonctionnalités de monitoring et d'actions principales.

#### Sprint 3 : Monitoring endpoints

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Implémenter l'interface IConnector | Haute | 0.5j |
| Créer le DomainHealthConnector | Haute | 1j |
| Créer l'ApimConnector | Haute | 1j |
| Implémenter le service de health check | Haute | 1j |
| Créer les endpoints API monitoring | Haute | 0.5j |
| Développer le dashboard frontend (pavés colorés) | Haute | 1.5j |
| Implémenter le rafraîchissement automatique | Moyenne | 0.5j |

**Livrables :**
- Dashboard de monitoring fonctionnel
- Connectors Domain et APIM opérationnels
- Affichage temps réel des statuts

#### Sprint 4 : Administration et actions

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Créer les endpoints CRUD /admin/endpoints | Haute | 1j |
| Développer l'interface admin (liste, formulaire) | Haute | 1.5j |
| Implémenter la validation et le test de connectivité | Moyenne | 0.5j |
| Créer le KeycloakConnector | Haute | 1j |
| Développer la page de purge Keycloak | Haute | 0.5j |
| Implémenter le système de logs d'actions | Haute | 1j |

**Livrables :**
- Administration des endpoints complète
- Purge Keycloak fonctionnelle
- Traçabilité des actions

#### Sprint 5 : GitHub Actions et Talend

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Créer le GitHubConnector | Haute | 1.5j |
| Développer la page workflows (liste + trigger) | Haute | 1.5j |
| Gérer les inputs de workflow | Moyenne | 0.5j |
| Créer le TalendConnector | Haute | 1j |
| Développer la page jobs en erreur | Haute | 1j |

**Livrables :**
- Déclenchement de workflows GitHub fonctionnel
- Affichage des jobs Talend en erreur

---

### Phase 3 : Extensions (Sprint 6-7)

**Objectif :** Ajouter les connectors secondaires et améliorer l'expérience utilisateur.

#### Sprint 6 : Tickets et Cloudflare

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Créer le JiraConnector | Haute | 1j |
| Développer la page Jira (sprint + backlog) | Haute | 1j |
| Créer le GlpiConnector | Moyenne | 1j |
| Développer la page GLPI | Moyenne | 0.5j |
| Créer le CloudflareConnector | Moyenne | 1j |
| Développer la page purge Cloudflare | Moyenne | 0.5j |

**Livrables :**
- Intégration Jira et GLPI complètes
- Purge Cloudflare fonctionnelle

#### Sprint 7 : Polish et déploiement

| Tâche | Priorité | Estimation |
|-------|----------|------------|
| Améliorer les messages d'erreur | Moyenne | 0.5j |
| Optimiser les performances (lazy loading, memoization) | Moyenne | 1j |
| Ajouter les tests unitaires manquants | Haute | 1.5j |
| Configurer le CI/CD Azure | Haute | 1j |
| Déployer en staging | Haute | 0.5j |
| Tests d'acceptation utilisateur | Haute | 1j |
| Corrections post-UAT | Haute | 1j |

**Livrables :**
- Application testée et optimisée
- Pipeline CI/CD opérationnel
- Déploiement en production

---

### Récapitulatif du planning

| Phase | Sprints | Durée estimée |
|-------|---------|---------------|
| Phase 1 : Fondations | 1-2 | 2 sprints (4 semaines) |
| Phase 2 : Core Features | 3-5 | 3 sprints (6 semaines) |
| Phase 3 : Extensions | 6-7 | 2 sprints (4 semaines) |
| **Total** | **7 sprints** | **14 semaines** |

---

### Jalons clés

| Jalon | Date relative | Critère de validation |
|-------|---------------|----------------------|
| **M1 - Fondations** | Fin Sprint 2 | Auth fonctionnelle, projet structuré |
| **M2 - MVP Monitoring** | Fin Sprint 4 | Dashboard + admin + 1 action |
| **M3 - Feature Complete** | Fin Sprint 6 | Tous les connectors implémentés |
| **M4 - Production Ready** | Fin Sprint 7 | Déployé, testé, documenté |

---

## Évolutions futures (post-MVP)

### Backlog produit V2

| Fonctionnalité | Priorité | Complexité |
|----------------|----------|------------|
| Alerting par email/Teams | Haute | Moyenne |
| Historique avec graphiques (tendances) | Moyenne | Haute |
| Notifications push navigateur | Moyenne | Basse |
| Export des données (CSV, PDF) | Basse | Basse |
| Thème sombre | Basse | Basse |
| Multi-langue (EN) | Basse | Moyenne |
| Tableau de bord personnalisable | Moyenne | Haute |
| Intégration Slack | Basse | Moyenne |
| Application mobile (PWA) | Basse | Haute |

---

## Indicateurs de succès

### KPIs techniques

| Indicateur | Cible |
|------------|-------|
| Couverture de tests | > 80% |
| Temps de build | < 5min |
| Uptime application | > 99.5% |
| Temps de réponse P95 | < 500ms |

### KPIs produit

| Indicateur | Cible |
|------------|-------|
| Utilisateurs actifs hebdomadaires | > 10 |
| Actions exécutées par semaine | > 50 |
| Temps moyen de diagnostic | -50% vs avant |
| Satisfaction utilisateur (NPS) | > 7/10 |

---

## Conclusion

Ce PRD fournit une base solide pour le développement de l'application App Monitorage. Les documents sont structurés pour permettre un découpage automatique en tâches techniques via des outils comme claude-task-master.

### Points d'attention prioritaires

1. **Authentification Azure AD** : Valider le POC en début de projet
2. **Architecture connectors** : S'assurer de la flexibilité pour les ajouts futurs
3. **Performance** : Surveiller la charge générée par les health checks
4. **Sécurité** : Audit des credentials et des accès avant production

### Prochaines étapes

1. Validation du PRD par les parties prenantes
2. Setup de l'environnement de développement
3. Kick-off du Sprint 1

---

## Documents connexes

- [01 - Vue d'ensemble du produit](./01-overview-prd.md)
- [03 - Exigences fonctionnelles](./03-functional-requirements.md)
- [09 - Critères d'acceptation](./09-acceptance-criteria.md)
