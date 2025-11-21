🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.6. Métriques Vitales

## Introduction

Imaginez que vous pilotez un avion. Sur votre tableau de bord, parmi des dizaines d'indicateurs, certains sont **absolument critiques** : l'altitude, la vitesse, le niveau de carburant, la température des moteurs. Ces quelques métriques vous disent immédiatement si tout va bien ou si vous devez agir.

PostgreSQL, c'est pareil. Il existe des centaines de statistiques disponibles, mais seulement **quelques métriques vitales** vous donnent l'essentiel de l'information sur la santé de votre base de données.

Dans ce chapitre, nous allons découvrir les **5 métriques vitales** que tout administrateur de base de données doit surveiller en priorité. Ce sont les indicateurs qui font la différence entre un système performant et un système qui va bientôt rencontrer des problèmes.

---

## Qu'est-ce qu'une Métrique Vitale ?

### Définition

Une **métrique vitale** est un indicateur de performance qui :

1. **Révèle l'état de santé global** du système
2. **Prédit les problèmes** avant qu'ils deviennent critiques
3. **Guide les décisions** d'optimisation
4. **Se lit rapidement** et se comprend facilement

### Analogie Médicale

Quand vous allez chez le médecin, il prend d'abord vos **signes vitaux** :

| Signe Vital | Signification | Alerte si... |
|-------------|---------------|--------------|
| **Température** | Infection/Inflammation | > 38°C |
| **Tension artérielle** | Santé cardiovasculaire | > 140/90 |
| **Fréquence cardiaque** | Effort du cœur | > 100 bpm au repos |
| **Saturation O₂** | Oxygénation | < 95% |

Ces 4 mesures simples donnent une vision globale de votre état de santé. Le médecin n'a pas besoin de faire 50 examens pour savoir si vous allez bien.

**PostgreSQL a aussi ses signes vitaux !**

---

## Pourquoi ces Métriques sont Essentielles

### 1. Détection Précoce des Problèmes

Les métriques vitales agissent comme un **système d'alerte précoce**.

**Sans monitoring :**
```
Jour 1 : Tout semble normal
Jour 5 : Quelques lenteurs occasionnelles
Jour 10 : Base de données très lente
Jour 15 : 💥 Crash complet - Clients furieux - Urgence 3h du matin
```

**Avec monitoring des métriques vitales :**
```
Jour 1 : Alerte "Cache Hit Ratio en baisse" → Investigation
Jour 2 : Identification du problème (bloat) → Action planifiée
Jour 3 : Correction appliquée en heures creuses
Jour 4+ : Tout fonctionne normalement ✅
```

**Bénéfice :** Vous passez de la **gestion de crise** à la **gestion proactive**.

### 2. Priorisation des Efforts

Vous ne pouvez pas tout optimiser en même temps. Les métriques vitales vous disent **où agir en priorité**.

**Exemple de dashboard :**
```
Cache Hit Ratio       : 99.2% 🟢 → OK, ne rien faire
Bloat                : 8%    🟢 → OK, surveiller
I/O Wait             : 45%   🔴 → PRIORITÉ 1 : Problème critique !
Connexions actives   : 65%   🟡 → À surveiller
Checkpoints forcés   : 5%    🟢 → OK
```

**Conclusion immédiate :** Concentrez vos efforts sur l'I/O avant tout le reste.

### 3. Mesure de l'Impact des Changements

Quand vous faites une modification (ajout d'index, changement de configuration), les métriques vitales vous disent **si ça a fonctionné**.

**Exemple :**
```
AVANT l'ajout d'un index :
- Cache Hit Ratio : 92%
- I/O Wait : 25%
- Requêtes lentes : 150/min

APRÈS l'ajout d'un index :
- Cache Hit Ratio : 97%  ✅ +5%
- I/O Wait : 12%         ✅ -13%
- Requêtes lentes : 20/min ✅ -87%
```

**Verdict :** L'optimisation est un succès, mesurable et quantifiable.

### 4. Communication avec les Parties Prenantes

Les métriques vitales permettent d'expliquer simplement la situation aux non-techniciens.

**Mauvaise communication (trop technique) :**
> "Le ratio de dirty pages dans le shared buffer pool a augmenté de 15 points de base suite à une augmentation du facteur de charge transactionnelle..."

**Bonne communication (métrique vitale) :**
> "Notre base de données utilise 95% de ses connexions disponibles. Nous devons passer à un système de pooling cette semaine pour éviter une saturation."

---

## Vue d'Ensemble des 5 Métriques Vitales

Voici les 5 métriques que nous allons explorer en détail dans les sections suivantes :

### 1. Cache Hit Ratio (Taux de Succès du Cache)

**Question :** Mes données sont-elles en mémoire ou PostgreSQL attend-il constamment après le disque ?

**Indicateur :** Pourcentage de lectures servies depuis la RAM vs le disque

**Seuil cible :** > 99%

**Impact si problème :**
- Requêtes lentes
- I/O élevé
- Serveur qui "rame"

**Ce que vous apprendrez :**
- Comment mesurer le cache hit ratio
- Interpréter les résultats
- Augmenter l'efficacité du cache
- Optimiser shared_buffers

### 2. Table et Index Bloat (Gonflement)

**Question :** Ma base de données contient-elle beaucoup d'espace mort inutilisé ?

**Indicateur :** Pourcentage de "déchets" dans vos tables et index

**Seuil cible :** < 20%

**Impact si problème :**
- Tables anormalement volumineuses
- Performances dégradées progressivement
- Sauvegardes plus longues
- Cache moins efficace

**Ce que vous apprendrez :**
- Comprendre le bloat et pourquoi il se produit
- Le mesurer précisément
- Le prévenir avec VACUUM
- Le corriger si nécessaire

### 3. I/O Wait et Disk Latency (Attente Disque)

**Question :** Mon système passe-t-il trop de temps à attendre le disque ?

**Indicateur :** Pourcentage de temps CPU en attente I/O + latence disque

**Seuil cible :** I/O Wait < 10%, Latency < 5ms (SSD)

**Impact si problème :**
- Lenteurs généralisées
- CPU inactif mais système lent
- Gaspillage de ressources

**Ce que vous apprendrez :**
- Mesurer l'I/O Wait avec iostat et top
- Comprendre la latence disque
- Identifier les requêtes problématiques
- Optimiser (SSD, index, cache)

### 4. Connexions Actives et Pools Saturés

**Question :** Ai-je trop ou pas assez de connexions disponibles ?

**Indicateur :** Nombre de connexions actives vs limite maximale

**Seuil cible :** < 70% de max_connections

**Impact si problème :**
- Erreurs "too many clients"
- Timeouts fréquents
- Saturation mémoire
- Performances imprévisibles

**Ce que vous apprendrez :**
- Comprendre le coût d'une connexion
- Détecter les connection leaks
- Implémenter PgBouncer (pooling)
- Dimensionner correctement

### 5. Checkpoints et WAL Generation

**Question :** Mes checkpoints causent-ils des ralentissements périodiques ?

**Indicateur :** Fréquence et durée des checkpoints + volume WAL

**Seuil cible :** Checkpoints forcés < 10%, durée < timeout

**Impact si problème :**
- Pics de lenteur réguliers (ex: toutes les 5 min)
- I/O burst pendant checkpoints
- Expérience utilisateur dégradée

**Ce que vous apprendrez :**
- Comprendre le WAL et les checkpoints
- Configurer max_wal_size et checkpoint_timeout
- Lisser les écritures
- Monitorer efficacement

---

## Comment Utiliser ce Chapitre

### Pour les Débutants

**Approche recommandée :**

1. **Lisez l'introduction** (cette page) pour comprendre la vue d'ensemble
2. **Étudiez chaque métrique** dans l'ordre (14.6.1 → 14.6.5)
3. **Appliquez immédiatement** : Mesurez ces métriques sur votre système
4. **Créez un dashboard simple** avec ces 5 indicateurs
5. **Revenez régulièrement** : Ces métriques doivent devenir une seconde nature

**Temps estimé :** 5-8 heures pour maîtriser les 5 métriques

### Pour les Administrateurs Expérimentés

**Utilisation en référence :**

- Consultez directement la métrique qui vous intéresse
- Utilisez les requêtes SQL fournies
- Intégrez les alertes recommandées dans votre monitoring
- Adaptez les seuils à votre contexte spécifique

### Pour les DevOps/SRE

**Focus opérationnel :**

- Automatisez la collecte de ces métriques
- Créez des dashboards Grafana
- Configurez des alertes PagerDuty/Slack
- Documentez vos seuils dans des runbooks

---

## La Pyramide du Monitoring

### Niveau 1 : Métriques Vitales (Ce chapitre)

**5 métriques essentielles**
- Cache Hit Ratio
- Bloat
- I/O Wait
- Connexions
- Checkpoints

**Fréquence :** Surveillance continue (temps réel)

**Action :** Si l'une est dans le rouge → Investigation immédiate

### Niveau 2 : Métriques Secondaires

**20-30 métriques importantes**
- Query duration (p50, p95, p99)
- Table sizes
- Index usage
- Replication lag
- Transaction rate
- etc.

**Fréquence :** Consultation quotidienne

**Action :** Tendances et optimisations planifiées

### Niveau 3 : Métriques Détaillées

**100+ métriques spécifiques**
- Toutes les vues pg_stat_*
- Métriques système complètes
- Métriques applicatives

**Fréquence :** Consultation lors d'investigations

**Action :** Debug approfondi et troubleshooting

**Principe 80/20 :** 80% de la valeur vient des 20% de métriques (les métriques vitales).

---

## Outils Recommandés pour le Monitoring

### Pour Débuter (Gratuit et Simple)

#### 1. Requêtes SQL Manuelles

Chaque section fournit des requêtes SQL à copier-coller.

**Avantages :**
- ✅ Pas d'installation
- ✅ Fonctionne partout
- ✅ Apprentissage des concepts

**Inconvénients :**
- ❌ Manuel (pas de temps réel)
- ❌ Pas d'historique
- ❌ Pas d'alertes

#### 2. pgAdmin

Interface graphique avec monitoring intégré.

**Avantages :**
- ✅ Visuel et intuitif
- ✅ Dashboard simple
- ✅ Gratuit

**Inconvénients :**
- ❌ Limité en fonctionnalités
- ❌ Pas d'alertes avancées

### Pour la Production (Professionnel)

#### 1. Stack Prometheus + Grafana

**Architecture :**
```
PostgreSQL → postgres_exporter → Prometheus → Grafana
```

**Avantages :**
- ✅ Open source
- ✅ Très puissant
- ✅ Alertes complètes
- ✅ Historique long terme
- ✅ Standard de l'industrie

**Effort :** 1-2 jours d'installation et configuration

#### 2. Solutions Cloud Natives

**AWS CloudWatch RDS**, **Azure Monitor**, **GCP Cloud Monitoring**

**Avantages :**
- ✅ Intégré si vous utilisez RDS/Cloud SQL
- ✅ Zéro configuration
- ✅ Alertes automatiques

**Inconvénients :**
- ❌ Coût supplémentaire
- ❌ Moins de flexibilité

#### 3. Solutions Commerciales

**DataDog**, **New Relic**, **AppDynamics**

**Avantages :**
- ✅ Tout-en-un
- ✅ Support professionnel
- ✅ IA et prédictions

**Inconvénients :**
- ❌ Coûteux (50-500$/mois)

---

## Méthodologie : De la Mesure à l'Action

### Étape 1 : État des Lieux (Baseline)

**Objectif :** Connaître vos métriques actuelles

**Actions :**
1. Exécuter les requêtes de mesure pour chaque métrique
2. Noter les valeurs dans un tableau
3. Comparer aux seuils recommandés
4. Identifier les métriques "dans le rouge"

**Durée :** 30 minutes

### Étape 2 : Priorisation

**Objectif :** Déterminer quoi corriger en premier

**Règle de priorisation :**
```
Priorité 1 (🔴) : Critique - Action sous 24h
    - Cache Hit Ratio < 90%
    - I/O Wait > 30%
    - Connexions > 90%

Priorité 2 (🟠) : Important - Action sous 1 semaine
    - Cache Hit Ratio < 95%
    - Bloat > 30%
    - Checkpoints forcés > 30%

Priorité 3 (🟡) : À surveiller - Action planifiable
    - Toute métrique en zone orange
```

### Étape 3 : Investigation

**Objectif :** Comprendre la cause racine

**Approche :**
1. Lire la section détaillée de la métrique problématique
2. Exécuter les requêtes de diagnostic fournies
3. Identifier la cause (requête lente, bloat, config, etc.)
4. Documenter vos découvertes

**Durée :** 1-4 heures selon complexité

### Étape 4 : Action

**Objectif :** Corriger le problème

**Types d'actions :**

**Actions immédiates (< 1 heure) :**
- Ajout d'index
- VACUUM manuel
- Terminer connexions zombie
- Ajustement configuration (reload)

**Actions planifiées (fenêtre maintenance) :**
- VACUUM FULL
- REINDEX
- Migration SSD
- Redimensionnement serveur

### Étape 5 : Validation

**Objectif :** Vérifier que la correction a fonctionné

**Actions :**
1. Re-mesurer la métrique après l'action
2. Comparer avant/après
3. Surveiller pendant 24-48h
4. Documenter le résultat

**Critère de succès :** Métrique revenue dans la zone verte

### Étape 6 : Prévention

**Objectif :** Éviter que le problème se reproduise

**Actions :**
- Mettre en place alertes automatiques
- Documenter dans un runbook
- Automatiser la correction si possible
- Revoir la configuration pour prévenir récurrence

---

## Exemple de Dashboard Simple

Voici un exemple de dashboard minimaliste mais efficace que vous pouvez créer :

### Dashboard "PostgreSQL Health"

**Section 1 : Métriques Vitales (Gauges)**

```
┌─────────────────────────────────────────────────┐
│  Cache Hit Ratio        Bloat         I/O Wait  │
│     [99.2%]🟢          [12%]🟢        [8%]🟢    │
│                                                 │
│  Connexions          Checkpoints Forcés         │
│    [45/100]🟢           [7%]🟢                  │
└─────────────────────────────────────────────────┘
```

**Section 2 : Graphiques Temporels**

```
Cache Hit Ratio (24h)
100% |----------------------------------------
 95% |========================================
 90% |----------------------------------------
     +----------------------------------------
       0h     6h     12h    18h    24h
      Stable à 99% toute la journée ✅


I/O Wait % (24h)
 30% |----------------------------------------
 15% |          ██
  5% |████████████████████████████████████████
  0% +----------------------------------------
       0h     6h     12h    18h    24h
      Pic à 15% pendant checkpoint (normal) ✅
```

**Section 3 : Top N**

```
Top 5 Tables par Bloat          Top 5 Requêtes I/O
1. orders       : 35% 🔴      1. SELECT ... : 5.2s
2. logs         : 28% 🟠      2. UPDATE ... : 3.8s
3. sessions     : 15% 🟡      3. INSERT ... : 2.1s
4. products     : 8%  🟢      4. DELETE ... : 1.5s
5. users        : 5%  🟢      5. SELECT ... : 1.2s
```

**Légende Couleurs :**
- 🟢 Vert : OK
- 🟡 Jaune : À surveiller
- 🟠 Orange : Attention
- 🔴 Rouge : Action requise

---

## Fréquence de Consultation Recommandée

### Monitoring Actif (Production)

**Temps réel (Grafana/DataDog) :**
- Dashboard affiché en permanence dans la salle de contrôle
- Alertes envoyées automatiquement
- Consultation réflexe en cas de problème

### Revue Quotidienne (10 minutes)

**Chaque matin :**
1. Consulter le dashboard des métriques vitales
2. Vérifier que tout est dans le vert
3. Noter les tendances (amélioration/dégradation)
4. Si orange/rouge : Planifier investigation

### Revue Hebdomadaire (30 minutes)

**Chaque lundi :**
1. Analyser les tendances de la semaine
2. Identifier les patterns (jour/heure problématiques)
3. Planifier optimisations si nécessaire
4. Documenter les incidents résolus

### Revue Mensuelle (2 heures)

**Début de chaque mois :**
1. Analyse approfondie des 5 métriques
2. Comparaison mois N vs mois N-1
3. Ajustement des seuils d'alerte si besoin
4. Planning d'optimisations pour le mois
5. Rapport pour le management

---

## Checklist de Démarrage

Voici une checklist pour bien démarrer avec les métriques vitales :

### Phase 1 : Préparation (Jour 1)

- [ ] Lire cette introduction complètement
- [ ] Choisir un outil de monitoring (commencer simple)
- [ ] S'assurer d'avoir accès en lecture à PostgreSQL
- [ ] Créer un document pour noter les résultats

### Phase 2 : Mesure Initiale (Jour 1-2)

- [ ] Mesurer le Cache Hit Ratio
- [ ] Mesurer le Bloat des tables principales
- [ ] Mesurer l'I/O Wait
- [ ] Compter les connexions actives
- [ ] Vérifier les statistiques de checkpoints
- [ ] Documenter toutes les valeurs (baseline)

### Phase 3 : Analyse (Jour 2-3)

- [ ] Identifier les métriques problématiques
- [ ] Prioriser les actions (🔴 > 🟠 > 🟡)
- [ ] Lire en détail les sections concernées
- [ ] Planifier les corrections

### Phase 4 : Action (Semaine 1)

- [ ] Appliquer les corrections prioritaires
- [ ] Re-mesurer après chaque action
- [ ] Valider les améliorations
- [ ] Documenter ce qui a fonctionné

### Phase 5 : Automatisation (Semaine 2-3)

- [ ] Mettre en place un dashboard
- [ ] Configurer des alertes automatiques
- [ ] Établir une routine de consultation
- [ ] Former l'équipe

### Phase 6 : Amélioration Continue (Permanent)

- [ ] Revue quotidienne (10 min)
- [ ] Revue hebdomadaire (30 min)
- [ ] Revue mensuelle (2h)
- [ ] Ajustements et optimisations continues

---

## Ce que Vous Allez Apprendre

En maîtrisant ces 5 métriques vitales, vous serez capable de :

### Compétences Techniques

- ✅ **Diagnostiquer** 90% des problèmes de performance PostgreSQL
- ✅ **Mesurer** précisément la santé de votre base de données
- ✅ **Optimiser** de manière ciblée et efficace
- ✅ **Prévenir** les pannes avant qu'elles surviennent
- ✅ **Expliquer** les problèmes en termes compréhensibles

### Compétences Opérationnelles

- ✅ **Monitorer** efficacement en production
- ✅ **Alerter** au bon moment (ni trop, ni trop peu)
- ✅ **Prioriser** les efforts d'optimisation
- ✅ **Documenter** l'état de santé du système
- ✅ **Communiquer** avec les stakeholders

### État d'Esprit

- ✅ **Proactivité** : Agir avant que ça casse
- ✅ **Data-driven** : Décisions basées sur des mesures
- ✅ **Pragmatisme** : Focus sur ce qui compte vraiment
- ✅ **Confiance** : Savoir où regarder quand il y a un problème

---

## Transition vers les Sections Détaillées

Vous êtes maintenant prêt à plonger dans le détail de chaque métrique vitale.

**Ordre recommandé :**

1. **Cache Hit Ratio** (14.6.1) - Le plus universel et le plus impactant
2. **I/O Wait et Disk Latency** (14.6.3) - Souvent lié au cache
3. **Table et Index Bloat** (14.6.2) - Affecte cache et I/O
4. **Connexions et Pools** (14.6.4) - Critique pour la disponibilité
5. **Checkpoints et WAL** (14.6.5) - Optimisation avancée

Chaque section est **autonome** et peut être consultée indépendamment, mais elles sont aussi **interconnectées** : améliorer l'une impacte souvent les autres positivement.

**Bon monitoring et bonnes optimisations !** 🚀

---

## Points Clés à Retenir

✅ **5 métriques vitales** suffisent pour 90% du monitoring PostgreSQL

✅ **Principe 80/20** : Ces quelques métriques donnent l'essentiel de l'information

✅ **Monitoring proactif** : Détecter les problèmes avant qu'ils deviennent critiques

✅ **Mesure → Action → Validation** : Approche méthodique et quantifiable

✅ **Commencer simple** : Requêtes SQL manuelles puis automatisation progressive

✅ **Dashboard minimal** : 5 indicateurs bien choisis > 50 indicateurs inutilisés

✅ **Routine essentielle** : Consultation quotidienne (10 min) + revue hebdomadaire (30 min)

✅ **Chaque métrique a un seuil** : Savoir ce qui est OK, à surveiller, ou critique

---


⏭️ [Cache Hit Ratio (Buffer Cache)](/14-observabilite-et-monitoring/06.1-cache-hit-ratio.md)
