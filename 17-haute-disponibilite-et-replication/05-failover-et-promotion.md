🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5. Failover et Promotion : Introduction

## Vue d'Ensemble

Dans une architecture de haute disponibilité PostgreSQL, le **failover** et la **promotion** sont des mécanismes essentiels qui permettent à votre système de continuer à fonctionner même lorsque le serveur principal (Primary) rencontre un problème.

Cette section explore les différentes stratégies pour gérer ces situations critiques, du plus simple (intervention manuelle) au plus sophistiqué (automatisation complète avec consensus distribué).

---

## Qu'est-ce que le Failover et la Promotion ?

### Définitions

#### Promotion

La **promotion** est l'action de transformer un serveur Standby (réplica en lecture seule) en serveur Primary (serveur principal acceptant les écritures).

**Analogie** : Si le capitaine d'une équipe est blessé, un autre joueur devient le nouveau capitaine.

**Techniquement** :
- Le Standby arrête de recevoir les WAL (Write-Ahead Logs) du Primary
- Il termine de rejouer tous les WAL en attente
- Il passe en mode "lecture-écriture"
- Il devient le nouveau Primary et commence à générer ses propres WAL

#### Failover

Le **failover** est le processus complet de basculement depuis un Primary défaillant vers un Standby promu.

**Le failover inclut** :
1. Détection de la panne du Primary
2. Sélection d'un Standby approprié
3. Promotion de ce Standby en nouveau Primary
4. Reconfiguration des autres Standby pour suivre le nouveau Primary
5. Redirection des applications vers le nouveau Primary

**Analogie** : C'est comme un plan de succession dans une entreprise. Si le PDG démissionne soudainement, il faut non seulement nommer un nouveau PDG (promotion), mais aussi réorganiser toute l'entreprise autour de ce nouveau leader (failover).

### Promotion vs Failover : Quelle Différence ?

| Aspect | Promotion | Failover |
|--------|-----------|----------|
| **Portée** | Une seule action | Processus complet |
| **Acteur** | Le Standby qui devient Primary | Tout le cluster |
| **Commande** | `pg_ctl promote` | Orchestration complexe |
| **Durée** | Quelques secondes | Quelques secondes à minutes |

**En résumé** :
- La **promotion** est l'acte technique de transformer un Standby
- Le **failover** est le processus organisationnel complet

---

## Pourquoi le Failover est-il Crucial ?

### Les Pannes Sont Inévitables

En production, les pannes ne sont pas une question de "si", mais de "quand" :

**Causes de panne courantes** :
- 💥 **Panne matérielle** : Disque dur défaillant, RAM défectueuse, alimentation coupée
- 🔥 **Incident datacenter** : Incendie, inondation, coupure électrique
- 🌐 **Problème réseau** : Câble sectionné, routeur en panne, DDoS
- 🐛 **Bug logiciel** : Corruption de données, bug PostgreSQL (rare), bug du kernel
- 👤 **Erreur humaine** : Suppression accidentelle, mauvaise commande, mauvaise configuration
- 🔄 **Maintenance** : Mise à jour OS, migration hardware, patch de sécurité

### Impact d'une Panne sans HA

Sans mécanisme de failover, une panne du Primary signifie :

```
Primary tombe en panne à 10h00
         ↓
Applications ne peuvent plus écrire
         ↓
Business en pause
         ↓
Équipe intervient manuellement
         ↓
Promotion manuelle d'un Standby (10-30 minutes)
         ↓
Reconfiguration des applications
         ↓
Retour à la normale à 10h45

→ Perte de 45 minutes de productivité
→ Perte potentielle de transactions
→ Impact client majeur
```

### Objectifs du Failover

Un bon système de failover vise à :

1. **Minimiser le RTO** (Recovery Time Objective) : Temps d'indisponibilité
2. **Minimiser le RPO** (Recovery Point Objective) : Perte de données
3. **Éviter le split-brain** : Deux Primary actifs simultanément
4. **Automatiser** : Réduire l'intervention humaine
5. **Assurer la cohérence** : Garantir l'intégrité des données

---

## RTO et RPO : Comprendre les Objectifs

### RTO (Recovery Time Objective)

Le **RTO** est le temps maximum acceptable d'indisponibilité de votre système.

**Question** : "Combien de temps mon application peut-elle être en panne ?"

**Exemples** :
- **Site e-commerce** : RTO = 1-2 minutes (chaque minute = perte de revenus)
- **Application interne** : RTO = 15-30 minutes (acceptable pendant heures de bureau)
- **Service financier** : RTO = 30 secondes (critique)
- **Blog personnel** : RTO = 1 heure (moins critique)

**Impact du RTO sur l'architecture** :

| RTO Cible | Solution Recommandée | Coût |
|-----------|---------------------|------|
| < 30 secondes | Patroni + etcd + Fencing automatique | 💰💰💰💰 Élevé |
| 1-2 minutes | Patroni ou Repmgr avec failover automatique | 💰💰💰 Moyen-élevé |
| 5-10 minutes | Repmgr avec intervention humaine rapide | 💰💰 Moyen |
| 15-30 minutes | Promotion manuelle avec procédure documentée | 💰 Faible |

### RPO (Recovery Point Objective)

Le **RPO** est la quantité maximale de données que vous pouvez vous permettre de perdre.

**Question** : "Combien de transactions puis-je perdre en cas de panne ?"

**Exemples** :
- **Banque** : RPO = 0 (perte de transactions = catastrophe)
- **Réseau social** : RPO = 1-5 minutes (quelques posts perdus = acceptable)
- **Application de logs** : RPO = 15 minutes (logs récents perdus = acceptable)
- **Analytics** : RPO = 1 heure (données agrégées = recalculables)

**Impact du RPO sur la réplication** :

| RPO Cible | Mode de Réplication | Trade-off |
|-----------|---------------------|-----------|
| 0 (zéro perte) | Réplication **synchrone** | ⚠️ Performances réduites |
| < 1 minute | Réplication **asynchrone** avec Standby proche | ⚠️ Perte de quelques transactions |
| 5-15 minutes | Réplication **asynchrone** standard | ✅ Meilleures performances |

### Visualisation RTO vs RPO

```
                Panne du Primary
                       │
                       ▼
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
    Dernière                      Retour
    transaction                  en ligne
    sauvegardée                 du service
        │                             │
        └─────────RPO─────────────────┘
        │                             │
        └──────────────RTO────────────┘

RPO = Données perdues (axe temporel)
RTO = Durée d'indisponibilité (axe temporel)
```

**Objectif idéal** : RTO et RPO les plus courts possibles

**Réalité** : Trade-off entre :
- Coût (infrastructure, complexité)
- Performance (réplication synchrone = plus lent)
- Risque (automatisation = risque de faux positifs)

---

## Le Problème du Split-Brain

### Qu'est-ce que le Split-Brain ?

Le **split-brain** (cerveau divisé) est une situation catastrophique où **deux serveurs PostgreSQL se considèrent tous les deux comme Primary** et acceptent des écritures simultanément.

**Analogie** : Imaginez une entreprise avec deux PDG qui prennent des décisions contradictoires. Le chaos est garanti.

### Comment le Split-Brain Se Produit-il ?

**Scénario classique** :

```
État Initial :
┌──────────────┐         ┌──────────────┐
│  Primary     │────────►│  Standby     │
│  (postgres1) │         │  (postgres2) │
└──────────────┘         └──────────────┘

Étape 1 : Perte de connexion réseau
┌──────────────┐    ✗    ┌──────────────┐
│  Primary     │         │  Standby     │
│  (postgres1) │         │  (postgres2) │
└──────────────┘         └──────────────┘
     ▲                        ▲
     │                        │
"Je suis toujours       "Je ne vois plus
 le Primary"             le Primary"

Étape 2 : Promotion du Standby (sans vérification)
┌──────────────┐    ✗    ┌──────────────┐
│  Primary     │         │  Primary     │ 💥
│  (postgres1) │         │  (postgres2) │ 💥
└──────────────┘         └──────────────┘
     ▲                        ▲
     │                        │
Écrit dans               Écrit dans
Timeline 1               Timeline 2

→ SPLIT-BRAIN !
→ Deux Primary actifs
→ Divergence des données
→ Corruption garantie
```

### Conséquences du Split-Brain

**Impact immédiat** :
- 💥 Deux copies de données qui divergent
- 💥 Impossible de les réconcilier automatiquement
- 💥 Perte de données garantie
- 💥 Corruption potentielle

**Impact à long terme** :
- 🚨 Restauration complexe (heures/jours)
- 🚨 Perte de confiance des utilisateurs
- 🚨 Analyse forensique nécessaire
- 🚨 Possible perte définitive de données

### Comment Éviter le Split-Brain ?

**Mécanisme 1 : Fencing (Isolation)**

Le **fencing** consiste à isoler physiquement l'ancien Primary pour garantir qu'il ne peut plus accepter d'écritures.

**Techniques de fencing** :
- **STONITH** (Shoot The Other Node In The Head) : Éteindre physiquement le serveur
- **Isolation réseau** : Couper la connexion réseau
- **PostgreSQL pause** : Mettre PostgreSQL en pause automatiquement

**Exemple avec Patroni** :
```
Patroni détecte la perte du leader lock
       ↓
Patroni met automatiquement PostgreSQL en pause
       ↓
PostgreSQL ne peut plus accepter d'écritures
       ↓
Pas de split-brain possible
```

**Mécanisme 2 : Consensus Distribué**

Utiliser un système de consensus (etcd, Consul) qui garantit qu'**un seul serveur à la fois** peut détenir le "leader lock".

**Exemple avec Patroni + etcd** :
```
3 nœuds PostgreSQL + 3 nœuds etcd

Pour devenir Primary, il faut :
1. Obtenir le leader lock dans etcd
2. Seul 1 nœud peut détenir ce lock
3. Le lock a une durée de vie (TTL = 30s)
4. Si le Primary ne renouvelle pas le lock → il expire
5. Un autre nœud peut alors l'obtenir

→ Impossible d'avoir 2 Primary (garanti par etcd)
```

**Mécanisme 3 : Witness Node (Arbitre)**

Utiliser un **serveur témoin** qui sert d'arbitre en cas de doute.

**Exemple avec Repmgr** :
```
Standby ne voit plus le Primary
       ↓
Standby demande au Witness : "Est-ce que tu vois le Primary ?"
       ↓
Si Witness dit "Non" → C'est une vraie panne → OK pour promouvoir
Si Witness dit "Oui" → C'est juste un problème réseau local → NE PAS promouvoir
```

---

## Types de Failover

### 1. Failover Non Planifié (Unplanned Failover)

**Contexte** : Le Primary tombe en panne de manière inattendue.

**Caractéristiques** :
- ⚠️ Aucune préparation
- ⚠️ Risque de perte de données (RPO > 0)
- ⚠️ Urgence maximum
- ⚠️ Stress élevé pour l'équipe

**Déclencheurs** :
- Panne matérielle soudaine
- Crash du système
- Corruption de données
- Incident datacenter

**Objectif** : Minimiser le RTO (revenir en ligne le plus vite possible)

### 2. Switchover Planifié (Planned Switchover)

**Contexte** : Vous décidez volontairement de basculer vers un Standby.

**Caractéristiques** :
- ✅ Maintenance planifiée
- ✅ Aucune perte de données (RPO = 0)
- ✅ Processus contrôlé
- ✅ Downtime minimal

**Cas d'usage** :
- Mise à jour de l'OS du Primary
- Migration hardware
- Changement de datacenter
- Test de la procédure de failover

**Processus** :
```
1. Annoncer une fenêtre de maintenance
2. Arrêter proprement le Primary (pg_ctl stop -m fast)
3. Attendre que le Standby soit 100% à jour
4. Promouvoir le Standby
5. Reconfigurer les applications
6. (Optionnel) Transformer l'ancien Primary en Standby
```

**Avantage** : RTO minimal (quelques secondes) et RPO = 0

---

## Les Trois Approches de Failover

Cette section présente trois approches pour gérer le failover, de la plus simple à la plus sophistiquée.

### Approche 1 : Promotion Manuelle (Section 17.5.1)

**Principe** : Un administrateur détecte la panne et lance manuellement la promotion.

**Outil** : `pg_ctl promote`

**Avantages** :
- ✅ Simplicité maximale
- ✅ Pas d'infrastructure additionnelle
- ✅ Contrôle total
- ✅ Pas de faux positifs

**Inconvénients** :
- ❌ Intervention humaine requise 24/7
- ❌ RTO élevé (plusieurs minutes minimum)
- ❌ Risque d'erreur humaine
- ❌ Pas de protection split-brain automatique

**Cas d'usage** :
- Environnements de développement/test
- Switchover planifié
- Clusters simples avec équipe disponible
- Budget très limité

**Architecture** :
```
┌──────────────┐         ┌──────────────┐
│  Primary     │────────►│  Standby     │
│              │         │              │
└──────────────┘         └──────────────┘
                              ▲
                              │
                         Administrateur
                       (exécute pg_ctl promote)
```

### Approche 2 : Automatisation Complète avec Patroni (Section 17.5.2)

**Principe** : Un système de consensus (etcd/Consul) coordonne automatiquement le failover.

**Outil** : Patroni + etcd/Consul/ZooKeeper

**Avantages** :
- ✅ Failover automatique (15-30 secondes)
- ✅ Protection split-brain excellente
- ✅ Fencing automatique
- ✅ Pas d'intervention humaine nécessaire
- ✅ Idéal pour production critique

**Inconvénients** :
- ❌ Complexité élevée
- ❌ Infrastructure additionnelle (etcd/Consul)
- ❌ Courbe d'apprentissage importante
- ❌ Coût d'exploitation plus élevé

**Cas d'usage** :
- Production critique (99.99%+ uptime)
- Clusters complexes (5+ nœuds)
- Déploiements Kubernetes
- Applications financières, e-commerce

**Architecture** :
```
┌─────────────────────────────────────────┐
│   Système de Consensus (etcd/Consul)    │
│   "Un seul Primary à la fois"           │
└─────────────────────────────────────────┘
                  ▲
                  │ Leader Lock
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Patroni 1│ │ Patroni 2│ │ Patroni 3│
│          │ │          │ │          │
│ Primary  │ │ Standby  │ │ Standby  │
└──────────┘ └──────────┘ └──────────┘
```

### Approche 3 : Solution Intermédiaire avec Repmgr (Section 17.5.3)

**Principe** : Gestion simplifiée avec failover optionnellement automatique, sans système de consensus externe.

**Outil** : Repmgr + repmgrd

**Avantages** :
- ✅ Simplicité relative
- ✅ Pas d'infrastructure externe obligatoire
- ✅ Failover automatique possible (avec witness)
- ✅ Courbe d'apprentissage courte
- ✅ Excellent pour cloner des Standby

**Inconvénients** :
- ⚠️ Protection split-brain moins robuste
- ⚠️ Pas de fencing automatique
- ⚠️ RTO un peu plus long (30-60 secondes)
- ⚠️ Nécessite un witness pour être fiable

**Cas d'usage** :
- PME et startups
- Clusters de petite à moyenne taille (2-5 nœuds)
- Équipes privilégiant la simplicité
- Budget moyen

**Architecture** :
```
┌──────────────┐         ┌──────────────┐
│  repmgrd 1   │         │  repmgrd 2   │
│      │       │         │      │       │
│      ▼       │         │      ▼       │
│  Primary     │────────►│  Standby     │
└──────────────┘         └──────────────┘
        │                        │
        └────────┬───────────────┘
                 ▼
          ┌──────────────┐
          │   Witness    │ ← Arbitre
          │   (optionnel)│
          └──────────────┘
```

### Comparaison des Trois Approches

| Critère | Manuelle (17.5.1) | Patroni (17.5.2) | Repmgr (17.5.3) |
|---------|-------------------|------------------|-----------------|
| **Complexité** | ⭐ Très simple | ⭐⭐⭐⭐⭐ Complexe | ⭐⭐⭐ Moyenne |
| **Infrastructure** | Aucune | etcd/Consul/ZK | Aucune (+ witness) |
| **RTO** | 5-30 minutes | 15-30 secondes | 30-60 secondes |
| **Failover auto** | ❌ | ✅ | ⚠️ Optionnel |
| **Split-brain protection** | ❌ | ✅✅✅ | ⚠️ Partielle |
| **Fencing auto** | ❌ | ✅ | ❌ |
| **Coût** | 💰 Très faible | 💰💰💰💰 Élevé | 💰💰 Moyen |
| **Maintenance** | Faible | Moyenne | Faible |
| **Production critique** | ❌ | ✅ | ⚠️ PME |
| **Idéal pour** | Dev/Test | Entreprise | Petits clusters |

---

## Facteurs de Décision : Quelle Approche Choisir ?

### Matrice de Décision

**Utilisez la promotion MANUELLE (17.5.1) si** :
- 📌 Vous êtes en environnement de développement ou test
- 📌 Votre RTO acceptable est > 15 minutes
- 📌 Vous avez une équipe disponible 24/7
- 📌 Votre budget est très limité
- 📌 Vous voulez comprendre les mécanismes de base

**Utilisez PATRONI (17.5.2) si** :
- 📌 Votre RTO doit être < 1 minute
- 📌 Vous avez des exigences de uptime critiques (99.99%+)
- 📌 Vous gérez un cluster complexe (5+ nœuds)
- 📌 Vous déployez dans Kubernetes
- 📌 Vous avez les ressources pour gérer etcd/Consul
- 📌 Le split-brain doit être impossible

**Utilisez REPMGR (17.5.3) si** :
- 📌 Vous voulez plus que du manuel mais moins que Patroni
- 📌 Votre RTO acceptable est 1-5 minutes
- 📌 Vous gérez un cluster simple (2-5 nœuds)
- 📌 Vous privilégiez la simplicité
- 📌 Votre budget est moyen
- 📌 Vous voulez faciliter le clonage de Standby

### Questions à Se Poser

**1. Quel est mon RTO cible ?**
- < 30 secondes → Patroni obligatoire
- 1-2 minutes → Patroni ou Repmgr
- 5-15 minutes → Repmgr ou Manuel
- > 15 minutes → Manuel acceptable

**2. Quel est mon RPO cible ?**
- 0 (zéro perte) → Réplication synchrone + Patroni
- < 1 minute → Réplication asynchrone + Patroni/Repmgr
- 5-15 minutes → Réplication asynchrone standard

**3. Quelle est ma criticité métier ?**
- Financier/E-commerce → Patroni
- SaaS/Web app → Patroni ou Repmgr
- Application interne → Repmgr ou Manuel
- Dev/Test → Manuel

**4. Quelle est la taille de mon équipe ?**
- DevOps expérimentés → Patroni
- Équipe polyvalente → Repmgr
- Équipe réduite → Manuel (avec procédure documentée)

**5. Quel est mon budget infrastructure ?**
- Élevé → Patroni (etcd + monitoring complet)
- Moyen → Repmgr (+ witness si possible)
- Faible → Manuel (+ documentation)

---

## Prérequis Communs aux Trois Approches

Quelle que soit l'approche choisie, certains éléments sont indispensables :

### 1. Réplication PostgreSQL Configurée

**Minimum requis** :
- Au moins 1 Primary + 1 Standby configurés
- Réplication en streaming fonctionnelle
- Tests réguliers de la réplication

### 2. Monitoring

**Métriques essentielles** :
- État du Primary (up/down)
- État des Standby (up/down)
- Retard de réplication (replication lag)
- Connexions actives
- Logs d'erreurs

### 3. Documentation

**Documents obligatoires** :
- Architecture du cluster (schéma)
- Procédure de failover (runbook)
- Contacts d'urgence
- Historique des incidents

### 4. Tests Réguliers

**Plan de tests** :
- Switchover planifié (mensuel)
- Failover non planifié (trimestriel)
- Reconstruction d'un Standby (trimestriel)
- Test du split-brain (annuel)

### 5. Backup et Recovery

**Essentiels** :
- Backups réguliers automatisés
- Tests de restauration
- Point-In-Time Recovery (PITR) configuré
- Backups hors site (3-2-1 rule)

---

## Concepts Avancés à Connaître

### Timeline PostgreSQL

Après chaque promotion, PostgreSQL incrémente la **timeline** :

```
Timeline 1 : Primary original
     │
     ├─ Promotion → Timeline 2 : Nouveau Primary
     │                   │
     │                   ├─ Promotion → Timeline 3 : Nouveau Primary
     │
Historique tracé dans l'historique des timelines
```

**Utilité** :
- Tracer l'historique des promotions
- Éviter la corruption lors de la réintégration d'un ancien Primary
- Permettre le Point-In-Time Recovery correct

### LSN (Log Sequence Number)

Le **LSN** est la position dans les fichiers WAL.

**Format** : `0/3000000` (segment/offset)

**Utilité** :
- Mesurer le retard de réplication
- Choisir le meilleur Standby lors d'un failover (LSN le plus élevé)
- Vérifier la cohérence après failover

### pg_rewind

**pg_rewind** permet de resynchroniser un ancien Primary sans reconstruction complète.

**Scénario** :
```
1. postgres1 était Primary (Timeline 1)
2. Failover → postgres2 devient Primary (Timeline 2)
3. postgres1 revient en ligne
4. pg_rewind peut resynchroniser postgres1 avec postgres2
   sans refaire un pg_basebackup complet
```

**Avantage** : Réintégration rapide (minutes au lieu d'heures)

---

## Checklist Pré-Failover

Avant de mettre en place un système de failover, assurez-vous que :

### Infrastructure

- [ ] Au moins 1 Primary + 1 Standby configurés et testés
- [ ] Réplication en streaming fonctionnelle
- [ ] pg_rewind activé (`wal_log_hints = on` ou checksums)
- [ ] Réseau stable entre les nœuds
- [ ] Latence réseau acceptable (< 10ms pour synchro)

### Sécurité

- [ ] Mots de passe robustes pour réplication
- [ ] pg_hba.conf correctement configuré
- [ ] SSL/TLS activé pour la réplication (recommandé)
- [ ] Firewall configuré

### Monitoring

- [ ] Monitoring du Primary (heartbeat)
- [ ] Monitoring des Standby (lag, état)
- [ ] Alertes configurées (Primary down, lag élevé)
- [ ] Logs centralisés

### Documentation

- [ ] Architecture documentée
- [ ] Runbook de failover rédigé et testé
- [ ] Contacts d'urgence à jour
- [ ] Procédure de rollback documentée

### Tests

- [ ] Test de switchover planifié réussi
- [ ] Test de failover non planifié réussi
- [ ] Test de réintégration de l'ancien Primary
- [ ] Test de reconstruction complète d'un Standby

### Backup

- [ ] Backups automatisés actifs
- [ ] Test de restauration récent (< 1 mois)
- [ ] PITR configuré et testé
- [ ] Backups stockés hors site

---

## Structure de Cette Section

Cette section 17.5 est organisée en trois parties progressives :

### 17.5.1. Promotion Manuelle (pg_ctl promote)

**Vous allez apprendre** :
- La commande `pg_ctl promote` en détail
- Procédure complète de promotion manuelle
- Scénarios de failover manuel
- Troubleshooting des problèmes courants

**Prérequis** : Aucun (base de départ)

**Durée d'apprentissage** : 1-2 heures

---

### 17.5.2. Patroni : HA Automatisé avec Consensus

**Vous allez apprendre** :
- Architecture de Patroni
- Systèmes de consensus (etcd, Consul, ZooKeeper)
- Configuration complète
- Failover automatique
- Load balancing avec HAProxy

**Prérequis** : Maîtrise de 17.5.1

**Durée d'apprentissage** : 4-8 heures

---

### 17.5.3. Repmgr : Gestion de Réplication Simplifiée

**Vous allez apprendre** :
- Architecture de Repmgr
- Configuration sans système externe
- Failover semi-automatique
- Witness node pour éviter les faux positifs
- Comparaison avec Patroni

**Prérequis** : Maîtrise de 17.5.1, lecture de 17.5.2 recommandée

**Durée d'apprentissage** : 3-6 heures

---

## Parcours d'Apprentissage Recommandé

### Pour les Débutants

```
Jour 1 : Lire l'introduction (cette section)
         → Comprendre les concepts (RTO, RPO, split-brain)

Jour 2-3 : Section 17.5.1 (Promotion Manuelle)
           → Pratiquer sur un cluster de test
           → Faire un failover manuel complet

Jour 4 : Décision stratégique
         → Ai-je besoin d'automatisation ?
         → Quel est mon RTO cible ?

Si automatisation nécessaire :
    Jour 5-8 : Section 17.5.2 (Patroni) OU Section 17.5.3 (Repmgr)
```

### Pour les Intermédiaires

```
Jour 1 : Révision rapide de l'introduction
         → Focus sur RTO/RPO et split-brain

Jour 2 : Comparaison Patroni vs Repmgr
         → Choix selon contexte

Jour 3-5 : Implémentation de la solution choisie
           → Configuration complète
           → Tests de failover

Jour 6 : Production
         → Monitoring
         → Documentation
```

### Pour les Avancés

```
Jour 1 : Évaluation architecturale
         → RTO/RPO réels de mon système actuel
         → Identification des risques

Jour 2 : Migration vers solution HA
         → Planification de la transition
         → Tests en environnement de staging

Jour 3-4 : Mise en production
           → Déploiement progressif
           → Monitoring renforcé

Jour 5+ : Optimisation continue
          → Tuning des paramètres
          → Amélioration du RTO/RPO
```

---

## Ressources Complémentaires

### Documentation Officielle

- **PostgreSQL** : https://www.postgresql.org/docs/current/warm-standby.html
- **Patroni** : https://patroni.readthedocs.io/
- **Repmgr** : https://www.repmgr.org/docs/current/
- **etcd** : https://etcd.io/docs/
- **Consul** : https://www.consul.io/docs

### Blogs Techniques Recommandés

- **2ndQuadrant** : https://www.2ndquadrant.com/en/blog/
- **Percona** : https://www.percona.com/blog/
- **CrunchyData** : https://www.crunchydata.com/blog
- **Zalando** : https://engineering.zalando.com/tags/postgresql.html

### Conférences

- **PGConf** (conférences régionales et internationales)
- **PostgreSQL sessions** (France)
- **FOSDEM** (Bruxelles) - Track PostgreSQL

---

## Conclusion de l'Introduction

Le failover et la promotion sont au cœur de toute stratégie de haute disponibilité PostgreSQL. La compréhension de ces mécanismes est essentielle pour :

- **Protéger votre business** contre les interruptions
- **Minimiser les pertes de données** en cas d'incident
- **Dormir tranquille** en sachant que votre système peut se réparer (partiellement ou totalement)

Cette section vous donne les clés pour choisir et implémenter la solution adaptée à vos besoins, que vous privilégiez :
- La **simplicité** (promotion manuelle)
- L'**automatisation totale** (Patroni)
- Le **compromis** (Repmgr)

**Prochaine étape** : Passez à la section 17.5.1 pour découvrir la promotion manuelle, fondement de toutes les solutions de failover.

---

**Points clés à retenir** :
- ✅ Le failover est inévitable en production
- ✅ RTO et RPO doivent guider votre choix d'architecture
- ✅ Le split-brain est la pire situation possible (évitez-le à tout prix)
- ✅ Trois approches existent : manuelle, Patroni, Repmgr
- ✅ Testez régulièrement votre procédure de failover
- ✅ La documentation et les backups sont aussi importants que la solution technique

Bonne lecture et bon apprentissage ! 🚀

⏭️ [Promotion manuelle (pg_ctl promote)](/17-haute-disponibilite-et-replication/05.1-promotion-manuelle.md)
