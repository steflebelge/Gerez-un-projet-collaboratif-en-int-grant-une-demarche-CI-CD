# BobApp — Document d'analyse CI/CD

## 1. Étapes du workflow CI/CD

### 1.1 Workflow CI — Déclenchement sur Pull Request vers `main`

Le workflow CI (`ci.yml`) s'exécute automatiquement à chaque ouverture ou mise à jour d'une Pull Request ciblant la branche `main`. Son objectif est de **bloquer le merge si les tests échouent**, garantissant que seul du code validé intègre la branche principale.

| Étape | Job | Objectif |
|-------|-----|----------|
| Checkout du code | `test-back` / `test-front` | Récupérer le code source de la branche source |
| Configuration Java 11 | `test-back` | Préparer l'environnement d'exécution Maven |
| `mvn clean verify` | `test-back` | Compiler le back-end, exécuter les tests unitaires et générer le rapport de couverture JaCoCo |
| Upload artifact JaCoCo | `test-back` | Rendre le rapport de couverture téléchargeable depuis l'interface GitHub |
| Configuration Node.js 18 | `test-front` | Préparer l'environnement npm/Angular |
| `npm install` | `test-front` | Installer les dépendances Angular |
| `ng test --coverage` | `test-front` | Exécuter les tests unitaires Angular avec Karma/ChromeHeadless et générer la couverture lcov |
| Upload artifact coverage | `test-front` | Rendre le rapport de couverture front téléchargeable |

Les deux jobs (`test-back` et `test-front`) s'exécutent **en parallèle** et de façon **indépendante**. Un échec dans l'un ou l'autre bloque le merge de la PR grâce à la protection de branche GitHub.

---

### 1.2 Workflow CD — Déclenchement sur Push vers `main`

Le workflow CD (`cd.yml`) s'exécute à chaque merge sur `main`. Il est **séquentiel** : chaque étape dépend du succès de la précédente via `needs:`. Son objectif est de **déployer automatiquement les images Docker sur Docker Hub** après avoir validé la qualité du code.

```
test-back ──→ sonar-back ──→ docker-back
test-front ──→ sonar-front ──→ docker-front
```

| Étape | Job | Objectif | Dépend de |
|-------|-----|----------|-----------|
| Tests back | `test-back` | Re-valider les tests Maven (filet de sécurité post-merge) | — |
| Tests front | `test-front` | Re-valider les tests Angular | — |
| Analyse SonarCloud back | `sonar-back` | Envoyer métriques (coverage JaCoCo, bugs, code smells) à SonarCloud et attendre le résultat de la Quality Gate | `test-back` |
| Analyse SonarCloud front | `sonar-front` | Envoyer métriques (coverage lcov, bugs, code smells) à SonarCloud et attendre le résultat de la Quality Gate | `test-front` |
| Build & Push Docker back | `docker-back` | Construire l'image Docker Spring Boot et la publier sur Docker Hub (`steflebelge/bobapp-back:latest`) | `sonar-back` |
| Build & Push Docker front | `docker-front` | Construire l'image Docker Angular et la publier sur Docker Hub (`steflebelge/bobapp-front:latest`) | `sonar-front` |

Si la Quality Gate SonarCloud échoue (`sonar.qualitygate.wait=true`), les jobs Docker ne s'exécutent pas. Aucune image dégradée n'est publiée.

---

## 2. KPIs proposés et Quality Gate

Les KPIs suivants ont été définis pour assurer un niveau de qualité minimal acceptable sur le nouveau code introduit à chaque PR.

| KPI | Seuil | Justification |
|-----|-------|---------------|
| **Couverture de code (Coverage)** | ≥ 80 % | Seuil standard de l'industrie garantissant que la majorité des chemins d'exécution sont testés. Une couverture insuffisante augmente le risque de régressions non détectées. |
| **New Blocker Issues** | 0 | Les blockers représentent des défauts critiques susceptibles de provoquer des crashes ou des failles de sécurité majeures. Aucune tolérance. |
| **New Critical Issues** | 0 | Les critiques sont des problèmes graves pouvant impacter la fiabilité ou la sécurité en production. |
| **Duplications (code dupliqué)** | < 3 % | La duplication augmente le coût de maintenance et le risque d'incohérences. Un seuil de 3 % est réaliste pour une petite équipe. |
| **Reliability Rating** | ≥ A | Indique l'absence de bugs bloquants ou critiques dans le nouveau code. |
| **Security Rating** | ≥ A | Indique l'absence de vulnérabilités de sécurité dans le nouveau code. |
| **Maintainability Rating** | ≥ A | Indique une dette technique maîtrisée (temps de résolution < 5 % du temps de développement). |

La **couverture à 80 %** est le KPI le plus structurant car il impose d'écrire des tests au fur et à mesure du développement, plutôt qu'en rattrapage.

---

## 3. Métriques SonarCloud — État actuel

Les métriques ci-dessous ont été obtenues après la première exécution complète du pipeline CD.

### Back-end (Spring Boot)

| Métrique | Valeur | Statut |
|----------|--------|--------|
| Security Rating | A | ✅ |
| Reliability Rating | D | ❌ |
| Maintainability Rating | A | ✅ |
| Coverage | 38,8 % | ❌ (objectif : 80 %) |
| Bugs | 1 | ❌ |
| Code Smells | — | — |

**Observation** : Le back-end présente un bug identifié par SonarCloud (Reliability D), et une couverture de tests très insuffisante (38,8 %). Ces deux points sont les priorités d'amélioration.

### Front-end (Angular)

| Métrique | Valeur | Statut |
|----------|--------|--------|
| Security Rating | A | ✅ |
| Reliability Rating | A | ✅ |
| Maintainability Rating | A | ✅ |
| Coverage | 47,6 % | ❌ (objectif : 80 %) |
| Bugs | 0 | ✅ |

**Observation** : Le front-end est en meilleur état qualitativement (aucun bug), mais la couverture reste insuffisante (47,6 %).

---

## 4. Notes et avis utilisateurs — Analyse

Les retours suivants ont été publiés en ligne par les utilisateurs de BobApp (note globale : 2 étoiles) :

> ⭐ *"Je mets une étoile car je ne peux pas en mettre zéro ! Impossible de poster une suggestion de blague, le bouton tourne et fait planter mon navigateur !"*

> ⭐⭐ *"#BobApp j'ai remonté un bug sur le post de vidéo il y a deux semaines et il est encore présent ! Les dévs vous faites quoi ?????"*

> ⭐ *"Ça fait une semaine que je ne reçois plus rien, j'ai envoyé un email il y a 5 jours mais toujours pas de nouvelles..."*

> ⭐⭐ *"J'ai supprimé ce site de mes favoris ce matin, dommage, vraiment dommage."*

**Synthèse des problèmes identifiés** :

| # | Problème | Source | Catégorie | Priorité |
|---|----------|--------|-----------|----------|
| 1 | Crash du navigateur lors de la soumission d'une suggestion de blague | Avis #1 | Fiabilité / Fonctionnel | 🔴 Critique |
| 2 | Bug sur le post de vidéo non corrigé depuis 2 semaines | Avis #2 | Fiabilité / Réactivité équipe | 🔴 Critique |
| 3 | Absence de contenu reçu depuis une semaine, sans réponse au support | Avis #3 | Fonctionnel / Support | 🟠 Haute |
| 4 | Expérience utilisateur dégradée conduisant à l'abandon de l'app | Avis #4 | UX général | 🟠 Haute |

---

## 5. Actions correctives priorisées

Sur la base des métriques SonarCloud et des retours utilisateurs, voici les actions à mener par ordre de priorité :

### Priorité 1 — Corriger le bug de soumission de suggestion (crash navigateur)

Directement signalé par l'avis #1 et corrélé au bug back-end identifié par SonarCloud (Reliability D). Le bouton de suggestion qui fait planter le navigateur est probablement lié à une exception non gérée côté serveur.

**Action** : Localiser le bug dans SonarCloud, corriger le code incriminé, écrire un test de non-régression. Vérifier également le comportement front (gestion des erreurs HTTP côté Angular).

### Priorité 2 — Corriger le bug sur le post de vidéo

L'avis #2 signale explicitement un bug sur la fonctionnalité "post de vidéo" non corrigé depuis deux semaines, signe d'un manque de visibilité sur les régressions.

**Action** : Reproduire le bug, identifier la cause racine, corriger et ajouter un test. Mettre en place un suivi des issues pour éviter qu'un bug signalé reste sans réponse visible.

### Priorité 3 — Investiguer l'absence de contenu reçu

L'avis #3 indique ne plus recevoir de contenu depuis une semaine. Cela peut pointer vers une tâche planifiée (scheduler) en erreur silencieuse ou un problème de notifications.

**Action** : Vérifier les logs du service de distribution de blagues en production. Ajouter des tests sur le scheduler et les mécanismes de notification. Améliorer la supervision (alertes en cas d'erreur silencieuse).

### Priorité 4 — Augmenter la couverture de tests back-end (38,8 % → 80 %)

La faible couverture laisse une large surface non testée, ce qui explique les régressions répétées constatées dans les avis.

**Action** : Identifier les classes non couvertes via le rapport JaCoCo et écrire des tests unitaires/d'intégration, en priorisant les services métier (suggestions, vidéos, notifications).

### Priorité 5 — Augmenter la couverture de tests front-end (47,6 % → 80 %)

Même logique côté Angular : les composants non testés sont exposés aux régressions visibles par les utilisateurs.

**Action** : Utiliser le rapport lcov pour cibler les composants non couverts. Prioriser les services Angular gérant les appels API et les interactions utilisateur critiques.
