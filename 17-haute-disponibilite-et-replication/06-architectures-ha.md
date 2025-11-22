🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6. Architectures HA (Haute Disponibilité)

## Introduction

Vous avez maintenant une infrastructure PostgreSQL avec réplication : un serveur Primary qui accepte les écritures et plusieurs serveurs Standby qui maintiennent des copies synchronisées des données. C'est un excellent point de départ, mais une question fondamentale reste en suspens : **comment transformer cette infrastructure en un système véritablement hautement disponible ?**

La **Haute Disponibilité** (High Availability - HA) ne se résume pas à avoir des serveurs de secours. C'est une philosophie de conception qui vise à **minimiser les interruptions de service** en anticipant les pannes, en automatisant les basculements, et en distribuant intelligemment la charge de travail.

Ce chapitre explore les différentes architectures et stratégies pour construire un système PostgreSQL hautement disponible, capable de résister aux pannes matérielles, aux défaillances réseau, et même aux erreurs humaines.

---

## Qu'est-ce que la Haute Disponibilité ?

### Définition

La **Haute Disponibilité** est la capacité d'un système à rester **opérationnel et accessible** même en cas de défaillance de certains de ses composants.

**Analogie simple :** Imaginez un pont reliant deux villes :
- **Sans HA** : Un seul pont. S'il s'effondre, le trafic est complètement interrompu.
- **Avec HA** : Plusieurs ponts. Si l'un s'effondre, le trafic continue par les autres ponts (peut-être avec un peu plus de congestion, mais sans interruption totale).

Dans le contexte de PostgreSQL :
- **Sans HA** : Un seul serveur. S'il tombe, votre application est en panne jusqu'à sa réparation.
- **Avec HA** : Plusieurs serveurs. Si le Primary tombe, un Standby prend automatiquement le relais en quelques secondes.

### Les Niveaux de Disponibilité

La disponibilité se mesure généralement en pourcentage de temps où le système est opérationnel sur une année :

| Niveau | Disponibilité | Temps d'arrêt par an | Temps d'arrêt par mois | Cas d'usage typique |
|--------|---------------|----------------------|------------------------|---------------------|
| 99% ("deux neuf") | 99.0% | ~3.65 jours | ~7.2 heures | Petites applications |
| 99.9% ("trois neuf") | 99.9% | ~8.76 heures | ~43.2 minutes | Applications standard |
| 99.99% ("quatre neuf") | 99.99% | ~52.56 minutes | ~4.32 minutes | Applications critiques |
| 99.999% ("cinq neuf") | 99.999% | ~5.26 minutes | ~25.9 secondes | Finance, santé, télécoms |
| 99.9999% ("six neuf") | 99.9999% | ~31.5 secondes | ~2.59 secondes | Systèmes ultra-critiques |

**Note importante :** Passer de 99.9% à 99.99% ne signifie pas "un peu mieux", mais **dix fois moins d'arrêts** ! Chaque "neuf" supplémentaire est exponentiellement plus difficile et coûteux à atteindre.

### Pourquoi la Haute Disponibilité est-elle Importante ?

#### 1. Impact Business

Les interruptions de service ont un coût direct :

**E-commerce :**
- Une panne d'1 heure pendant le Black Friday = potentiellement millions de dollars de pertes
- Perte de confiance des clients
- Clients qui partent vers la concurrence

**Services financiers :**
- Impossibilité de traiter les transactions
- Pénalités réglementaires
- Risques légaux

**SaaS / Applications web :**
- Perte de revenus (abonnements)
- Violation des SLA (Service Level Agreements)
- Dégradation de la réputation

#### 2. Exigences Réglementaires

Certains secteurs ont des obligations légales :
- **Banque/Finance** : Bâle III, PSD2 (temps d'arrêt maximum)
- **Santé** : HIPAA, disponibilité des dossiers médicaux
- **Télécommunications** : Réglementations sur la disponibilité du service

#### 3. Expérience Utilisateur

Les utilisateurs modernes ont des attentes élevées :
- Disponibilité 24/7/365
- Temps de réponse rapides
- Tolérance zéro aux interruptions

Une seule panne peut faire perdre des utilisateurs de manière permanente.

---

## Les Concepts Clés de la Haute Disponibilité

### 1. RTO (Recovery Time Objective)

**Définition :** Le **temps maximum acceptable** pour restaurer un service après une panne.

**Question :** "Combien de temps notre système peut-il être en panne avant que cela devienne inacceptable ?"

**Exemples :**
- **E-commerce** : RTO = 5 minutes (basculement automatique rapide)
- **Blog personnel** : RTO = 4 heures (intervention manuelle acceptable)
- **Services d'urgence** : RTO = 30 secondes (basculement quasi-instantané)

**Impact sur l'architecture :**
- RTO court → Besoin d'automatisation (failover automatique)
- RTO long → Intervention manuelle possible

### 2. RPO (Recovery Point Objective)

**Définition :** La **quantité maximale de données** qu'il est acceptable de perdre lors d'une panne.

**Question :** "Combien de données pouvons-nous nous permettre de perdre ?"

**Exemples :**
- **Transactions bancaires** : RPO = 0 (aucune perte acceptable → réplication synchrone)
- **Analytics** : RPO = 1 heure (données reconstituables)
- **Réseau social** : RPO = quelques secondes (réplication asynchrone acceptable)

**Impact sur l'architecture :**
- RPO = 0 → Réplication synchrone obligatoire
- RPO > 0 → Réplication asynchrone acceptable

### 3. Disponibilité Calculée

La disponibilité se calcule ainsi :

```
Disponibilité (%) = (Temps total - Temps d'arrêt) / Temps total × 100
```

**Exemple :**
- Temps total sur 1 an : 365 jours = 8760 heures
- Temps d'arrêt : 1 heure
- Disponibilité : (8760 - 1) / 8760 × 100 = **99.989%**

**Facteurs impactant la disponibilité :**
- Pannes matérielles (disques, RAM, CPU)
- Défaillances réseau
- Bugs logiciels
- Erreurs humaines (mauvaise manipulation)
- Maintenance planifiée
- Catastrophes naturelles

### 4. SPOF (Single Point of Failure)

**Définition :** Un **point de défaillance unique** est un composant dont la panne entraîne l'arrêt complet du système.

**Exemples de SPOF dans PostgreSQL :**
- ❌ Un seul serveur PostgreSQL (pas de réplication)
- ❌ Un seul load balancer (pas de redondance)
- ❌ Un seul switch réseau
- ❌ Une seule source d'alimentation électrique

**Objectif de la HA :** **Éliminer tous les SPOF** de l'architecture.

---

## Les Composants d'une Architecture HA PostgreSQL

Une architecture PostgreSQL hautement disponible se compose de plusieurs briques :

### 1. Réplication des Données

**Objectif :** Maintenir des copies des données sur plusieurs serveurs.

**Types :**
- **Réplication physique (Streaming Replication)** : Copie bloc par bloc du serveur Primary vers les Standby
- **Réplication logique** : Réplication au niveau des transactions (plus flexible)

**Modes :**
- **Synchrone** : Le Primary attend la confirmation des Standby (RPO = 0)
- **Asynchrone** : Le Primary envoie les données sans attendre (meilleure performance)

→ **Voir section 17.6.1** pour le détail des trade-offs

### 2. Détection de Panne (Health Checking)

**Objectif :** Détecter rapidement qu'un serveur est en panne ou ne répond plus.

**Mécanismes :**
- **Health checks réseau** : Vérification régulière de la connectivité (ping, connexion TCP)
- **Health checks applicatifs** : Exécution de requêtes SQL simples (`SELECT 1`)
- **Monitoring des métriques** : CPU, mémoire, I/O, replication lag

**Fréquence typique :** 1 à 10 secondes

**Faux positifs :** Un défi majeur. Un serveur lent ≠ un serveur en panne.

### 3. Failover (Basculement)

**Objectif :** Promouvoir automatiquement un Standby en Primary lorsque le Primary tombe.

**Types :**
- **Failover manuel** : L'administrateur décide et lance la promotion
- **Failover automatique** : Détection et promotion automatiques (ex: Patroni, Repmgr)

**Étapes typiques d'un failover :**
1. Détection de la panne du Primary
2. Élection d'un nouveau Primary parmi les Standby
3. Promotion du Standby élu en Primary
4. Redirection du trafic vers le nouveau Primary
5. Reconfiguration des autres Standby

**Durée :** De quelques secondes à quelques minutes selon l'automatisation.

### 4. Consensus et Coordination

**Objectif :** S'assurer que tous les composants du système ont une vision cohérente de l'état du cluster.

**Problème du "split-brain" :**
Si deux serveurs croient tous les deux être le Primary → corruption de données !

**Solutions :**
- **Algorithmes de consensus** : Raft, Paxos (utilisés par etcd, Consul, Zookeeper)
- **Quorum** : Décisions prises par majorité
- **Fencing** : Isolation forcée du serveur défaillant

**Outils pour PostgreSQL :**
- **Patroni** + etcd/Consul/Zookeeper
- **Repmgr** + repmgrd
- **Pacemaker** + Corosync

### 5. Load Balancing (Répartition de Charge)

**Objectif :** Distribuer les requêtes entre plusieurs serveurs pour :
- Répartir la charge de lecture sur les Standby
- Diriger automatiquement les écritures vers le Primary actuel
- Détecter et retirer les serveurs défaillants

**Solutions :**
- **HAProxy** : Proxy TCP rapide et léger
- **PgPool-II** : Middleware PostgreSQL avec routing intelligent
- **PgBouncer** : Connection pooler (souvent combiné avec HAProxy)

→ **Voir section 17.6.3** pour une comparaison détaillée

### 6. Backup et Recovery

**Objectif :** Protection contre :
- Corruption de données
- Erreurs humaines (DROP TABLE accidentel)
- Catastrophes (perte complète du datacenter)

**Types :**
- **Sauvegardes logiques** : pg_dump (base ou tables spécifiques)
- **Sauvegardes physiques** : pg_basebackup (copie complète)
- **PITR (Point-In-Time Recovery)** : Restauration à un instant précis via WAL

**Note :** La réplication ne remplace PAS les backups ! Elle protège contre les pannes matérielles, pas contre les erreurs logiques.

---

## Vue d'Ensemble des Stratégies d'Architecture HA

### Stratégie 1 : Réplication avec Failover Manuel

```
Primary ← Applications
   ↓ (Streaming Replication)
Standby (Hot Standby)
```

**Caractéristiques :**
- ✅ Simple à mettre en place
- ✅ Coût réduit (2 serveurs minimum)
- ⚠️ RTO élevé (intervention humaine nécessaire)
- ⚠️ Risque d'erreur humaine pendant le failover

**Disponibilité visée :** 99% - 99.9%

**Cas d'usage :** Petites applications, budgets limités, RTO acceptable > 30 minutes

---

### Stratégie 2 : Réplication avec Failover Automatique

```
Primary ← Load Balancer ← Applications
   ↓
Standby1 (+ repmgrd ou Patroni)
   ↓
Standby2
```

**Caractéristiques :**
- ✅ Basculement automatique (30 secondes à 2 minutes)
- ✅ Pas d'intervention humaine nécessaire
- ⚠️ Complexité accrue (système de consensus)
- ⚠️ Coût plus élevé (3+ serveurs recommandés)

**Disponibilité visée :** 99.9% - 99.99%

**Cas d'usage :** Applications critiques, SaaS, e-commerce

---

### Stratégie 3 : Réplication Synchrone avec Quorum

```
          Primary
         /   |   \
        /    |    \
    Standby1 Standby2 Standby3
    (Quorum de 2 requis)
```

**Caractéristiques :**
- ✅ Aucune perte de données (RPO = 0)
- ✅ Tolérance aux pannes (quorum)
- ⚠️ Impact sur les performances (latence synchrone)
- ⚠️ Nécessite 3+ serveurs

**Disponibilité visée :** 99.99% - 99.999%

**Cas d'usage :** Finance, santé, données ultra-critiques

→ **Voir section 17.6.2** pour comprendre le quorum-based commit

---

### Stratégie 4 : Multi-Datacenter avec Réplication Hybride

```
Datacenter 1 (Principal)        Datacenter 2 (Secours)
    Primary                          Standby3
      ↓ (sync)                       ↑ (async)
   Standby1                          |
      ↓ (sync)                       |
   Standby2 -------------------------|
```

**Caractéristiques :**
- ✅ Protection géographique (catastrophe naturelle)
- ✅ RPO faible localement (synchrone local)
- ✅ Résilience maximale
- ⚠️ Complexité élevée
- ⚠️ Coût important (infrastructure multi-sites)

**Disponibilité visée :** 99.99% - 99.999%

**Cas d'usage :** Entreprises critiques, réglementations strictes

---

### Stratégie 5 : Architecture Complète avec Tous les Composants

```
           Applications
                |
         Load Balancer HA
        (HAProxy + Keepalived)
                |
        +-----------------+
        |                 |
     Primary          Standby1
   (Patroni/etcd)   (Patroni/etcd)
        |                 |
     Standby2         Standby3
        |                 |
   (Monitoring)    (Connection Pool)
   (Backup Jobs)
```

**Caractéristiques :**
- ✅ Aucun SPOF
- ✅ Failover automatique multi-niveaux
- ✅ Disponibilité maximale
- ⚠️ Complexité opérationnelle très élevée
- ⚠️ Coût important (personnel, infrastructure)

**Disponibilité visée :** 99.999% - 99.9999%

**Cas d'usage :** Infrastructure critique nationale, systèmes de paiement, télécoms

---

## Les Trade-offs de la Haute Disponibilité

Construire une architecture HA implique toujours des compromis :

### 1. Performance vs Fiabilité

**Réplication synchrone :**
- ➕ RPO = 0 (aucune perte de données)
- ➖ Latence accrue (attente des Standby)

**Réplication asynchrone :**
- ➕ Performance maximale
- ➖ Risque de perte de quelques secondes de données

### 2. Simplicité vs Résilience

**Architecture simple (2 serveurs, failover manuel) :**
- ➕ Facile à comprendre et maintenir
- ➖ RTO élevé, intervention humaine nécessaire

**Architecture complexe (5+ serveurs, multi-datacenter, failover auto) :**
- ➕ RTO très faible, résilience maximale
- ➖ Complexité opérationnelle, coût élevé

### 3. Coût vs Disponibilité

| Niveau | Infrastructure minimale | Coût relatif |
|--------|------------------------|--------------|
| 99% | 1 Primary + backups | 1x |
| 99.9% | 1 Primary + 1 Standby + failover manuel | 2x |
| 99.99% | 1 Primary + 2 Standby + failover auto | 3-4x |
| 99.999% | 1 Primary + 3+ Standby + multi-DC + automation | 5-10x |

**Règle d'or :** Ne sur-architecturez pas ! Visez le niveau de disponibilité dont vous avez **réellement** besoin.

### 4. Automatisation vs Contrôle

**Failover automatique :**
- ➕ Réaction rapide (secondes)
- ➖ Risque de faux positifs (promotion inutile)

**Failover manuel :**
- ➕ Contrôle total (pas de surprise)
- ➖ RTO élevé (attente de l'humain)

---

## Mesurer et Améliorer la Disponibilité

### Calculer la Disponibilité Actuelle

**Formule :**
```
Disponibilité = MTBF / (MTBF + MTTR)

MTBF = Mean Time Between Failures (temps moyen entre pannes)
MTTR = Mean Time To Repair (temps moyen de réparation)
```

**Exemple :**
- Panne tous les 6 mois → MTBF = 4380 heures
- Temps de réparation moyen : 2 heures → MTTR = 2 heures
- Disponibilité = 4380 / (4380 + 2) = **99.95%**

### Identifier les Points d'Amélioration

**Augmenter le MTBF (réduire la fréquence des pannes) :**
- Meilleur matériel (disques redondants, alimentations multiples)
- Monitoring proactif (détecter les problèmes avant la panne)
- Maintenance préventive
- Tests de charge et de stress

**Réduire le MTTR (accélérer la récupération) :**
- Failover automatique
- Runbooks détaillés
- Automatisation des procédures
- Formation des équipes
- Monitoring et alerting efficaces

---

## Tester Votre Architecture HA

**Ne faites JAMAIS confiance à une architecture HA non testée !**

### 1. Tests de Failover Planifiés

**Fréquence recommandée :** Tous les 3 à 6 mois

**Procédure :**
1. Planifier le test en dehors des heures de pointe
2. Documenter l'état initial
3. Déclencher une panne du Primary (arrêt propre)
4. Mesurer le temps de détection + promotion
5. Vérifier l'intégrité des données
6. Tester la réintégration de l'ancien Primary
7. Documenter les observations

**Métriques à mesurer :**
- Temps de détection de la panne
- Temps de promotion du Standby
- RTO réel vs RTO objectif
- Perte de données (RPO)
- Erreurs applicatives pendant le basculement

### 2. Tests de Chaos Engineering

**Principe :** Introduire des pannes aléatoires en production pour vérifier la résilience.

**Exemples de scénarios :**
- Arrêt brutal du Primary (`kill -9`)
- Saturation réseau
- Saturation disque (WAL archiving bloqué)
- Panne d'un Standby pendant un basculement
- Panne du load balancer
- Corruption d'un fichier WAL

**Outils :** Chaos Monkey, Gremlin, LitmusChaos

### 3. Game Days

**Concept :** Simulation d'incidents majeurs avec toute l'équipe.

**Objectifs :**
- Tester les procédures de crise
- Former les équipes
- Identifier les failles du plan de réponse
- Améliorer la communication

---

## Checklist : Évaluer la Maturité de Votre Architecture HA

Utilisez cette checklist pour évaluer où vous en êtes :

### Niveau 0 : Aucune HA
- [ ] Un seul serveur PostgreSQL
- [ ] Pas de réplication
- [ ] Sauvegardes manuelles (ou inexistantes)
- [ ] Pas de monitoring
- **Disponibilité :** ~95% - 99%
- **Action :** Mettre en place des backups automatiques dès maintenant !

### Niveau 1 : HA Basique
- [ ] 1 Primary + 1 Standby en streaming replication
- [ ] Sauvegardes automatisées (pg_dump ou pg_basebackup)
- [ ] Monitoring basique (CPU, mémoire, disque)
- [ ] Procédure de failover documentée (manuel)
- **Disponibilité :** 99% - 99.9%
- **Action :** Tester le failover manuel au moins une fois

### Niveau 2 : HA Intermédiaire
- [ ] 1 Primary + 2+ Standby
- [ ] Réplication streaming + WAL archiving
- [ ] Sauvegardes avec PITR
- [ ] Monitoring avancé (pg_stat_replication, lag, locks)
- [ ] Alerting configuré (PagerDuty, OpsGenie, etc.)
- [ ] Load balancer basique (HAProxy)
- **Disponibilité :** 99.9% - 99.99%
- **Action :** Implémenter un failover automatique

### Niveau 3 : HA Avancée
- [ ] Architecture avec quorum (3+ Standby)
- [ ] Failover automatique (Patroni, Repmgr)
- [ ] Réplication synchrone locale + asynchrone distante
- [ ] Load balancing intelligent (PgPool-II ou HAProxy avec détection auto)
- [ ] Connection pooling (PgBouncer)
- [ ] Monitoring complet (métriques + logs + APM)
- [ ] Tests de failover réguliers
- [ ] Runbooks détaillés
- **Disponibilité :** 99.99% - 99.999%
- **Action :** Effectuer des tests de chaos engineering

### Niveau 4 : HA Expert
- [ ] Multi-datacenter avec réplication géographique
- [ ] Aucun SPOF (load balancers redondants, etc.)
- [ ] Quorum-based commit configuré
- [ ] Disaster Recovery testé régulièrement
- [ ] Automatisation complète (Terraform, Ansible, Kubernetes Operator)
- [ ] Observabilité avancée (traces distribuées)
- [ ] Game days trimestriels
- [ ] RTO < 30 secondes, RPO = 0
- **Disponibilité :** 99.999% - 99.9999%
- **Action :** Partager vos pratiques avec la communauté ! 🎉

---

## Plan de Route vers la Haute Disponibilité

Si vous débutez, voici un plan progressif :

### Phase 1 : Fondations (Mois 1-2)
1. **Mettre en place des backups automatisés**
   - pg_dump quotidien vers un stockage externe
   - Tester la restauration
2. **Implémenter le monitoring de base**
   - Métriques système (CPU, RAM, disque, I/O)
   - Métriques PostgreSQL (connexions, transactions, cache hit ratio)
3. **Documenter l'architecture actuelle**
   - Schémas d'infrastructure
   - Procédures de récupération

### Phase 2 : Réplication (Mois 3-4)
1. **Configurer la streaming replication**
   - 1 Primary + 1 Standby asynchrone
2. **Activer le WAL archiving**
   - Configuration pour PITR
3. **Tester la promotion manuelle**
   - Documenter la procédure
   - Chronométrer le RTO réel

### Phase 3 : Load Balancing (Mois 5-6)
1. **Déployer HAProxy ou PgPool-II**
   - Configuration read/write split
   - Health checks
2. **Ajuster les applications**
   - Utiliser le load balancer
   - Gérer les erreurs de connexion
3. **Monitoring du load balancer**
   - Métriques de distribution
   - Détection de pannes

### Phase 4 : Automatisation (Mois 7-9)
1. **Implémenter un système de consensus**
   - Déployer Patroni + etcd/Consul
2. **Configurer le failover automatique**
   - Tests en environnement de staging
   - Déploiement progressif en production
3. **Automatiser les tests**
   - Scripts de failover planifié
   - CI/CD pour les procédures

### Phase 5 : Optimisation (Mois 10-12)
1. **Affiner la configuration**
   - Réplication synchrone vs asynchrone selon les besoins
   - Quorum-based commit si nécessaire
2. **Améliorer l'observabilité**
   - Dashboards dédiés (Grafana)
   - Alerting intelligent (réduction des faux positifs)
3. **Chaos engineering**
   - Injecter des pannes contrôlées
   - Améliorer la résilience

---

## Ce que Vous Allez Apprendre dans les Sections Suivantes

Cette introduction pose les bases. Les trois sections qui suivent approfondissent les aspects techniques :

### Section 17.6.1 : Synchrone vs Asynchrone - Trade-offs

Vous apprendrez :
- Les différences fondamentales entre réplication synchrone et asynchrone
- Comment choisir entre les deux selon vos besoins
- Les métriques à surveiller
- Les configurations hybrides optimales

**Pourquoi c'est important :** C'est le choix le plus critique pour votre architecture HA. Il détermine votre RPO (perte de données acceptable) et impacte directement vos performances.

### Section 17.6.2 : Quorum-based Commit

Vous apprendrez :
- Comment éliminer le point de défaillance unique de la réplication synchrone
- Configurer un quorum pour la réplication
- Dimensionner correctement votre cluster
- Gérer les scénarios de panne

**Pourquoi c'est important :** Le quorum transforme une architecture fragile (un Standby unique) en un système vraiment résilient.

### Section 17.6.3 : Load Balancing avec HAProxy ou PgPool-II

Vous apprendrez :
- Les différences entre HAProxy et PgPool-II
- Comment configurer chaque solution
- Distribuer intelligemment les lectures et écritures
- Choisir la bonne solution selon votre cas d'usage

**Pourquoi c'est important :** Le load balancing est l'interface entre vos applications et votre infrastructure PostgreSQL. Une mauvaise configuration peut annuler tous les bénéfices de votre réplication.

---

## Conclusion

La Haute Disponibilité n'est pas un produit qu'on achète, c'est une **discipline** qui combine :
- Architecture réfléchie
- Automatisation intelligente
- Monitoring proactif
- Tests réguliers
- Procédures documentées
- Équipes formées

**Points clés à retenir :**

1. 🎯 **Définissez vos objectifs** : RTO, RPO, disponibilité cible
2. 🎯 **Éliminez les SPOF** : Redondance à tous les niveaux
3. 🎯 **Automatisez intelligemment** : Réduisez le MTTR sans créer de faux positifs
4. 🎯 **Testez, testez, testez** : Une HA non testée est une illusion
5. 🎯 **Évoluez progressivement** : Ne sur-architecturez pas dès le départ
6. 🎯 **Documentez tout** : Les runbooks sauvent des vies (et des emplois)

La haute disponibilité est un voyage, pas une destination. Commencez par les fondations solides (backups, monitoring), puis progressez vers l'automatisation et la résilience maximale.

Dans les sections suivantes, nous allons plonger dans les détails techniques de chaque composant pour vous donner les outils nécessaires à la construction d'une architecture PostgreSQL véritablement hautement disponible.

---

**Prochaine section :** 17.6.1. Synchrone vs Asynchrone : Trade-offs

**Sections connexes :**
- Section 17.2 : Réplication Physique (Streaming Replication)
- Section 17.5 : Failover et Promotion
- Section 14 : Observabilité et Monitoring
- Section 16.11 : Sauvegardes et Restauration

⏭️ [Synchrone vs Asynchrone : Trade-offs](/17-haute-disponibilite-et-replication/06.1-synchrone-vs-asynchrone-tradeoffs.md)
