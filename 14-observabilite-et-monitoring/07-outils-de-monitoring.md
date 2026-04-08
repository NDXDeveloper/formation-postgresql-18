🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7. Outils de monitoring

## Introduction

Imaginez que vous conduisez une voiture sans tableau de bord : pas de compteur de vitesse, pas de jauge d'essence, pas de voyant moteur. Vous pourriez rouler pendant un moment, mais tôt ou tard, vous seriez confronté à un problème que vous n'avez pas vu venir. **Une base de données sans monitoring, c'est exactement la même chose.**

Le monitoring (surveillance en français) est l'art de **garder un œil constant sur votre base de données PostgreSQL** pour :
- 🔍 Détecter les problèmes avant qu'ils ne deviennent critiques  
- 📈 Comprendre comment votre base évolue dans le temps  
- ⚡ Identifier les goulots d'étranglement de performance  
- 🛡️ Anticiper les besoins en ressources  
- 🚨 Être alerté immédiatement en cas d'incident

Dans ce chapitre, nous allons explorer les **outils essentiels** pour surveiller PostgreSQL efficacement, des solutions open-source aux plateformes cloud natives.

---

## Pourquoi le monitoring est-il crucial ?

### Le coût de l'ignorance

**Sans monitoring, vous êtes aveugle :**

```
Scénario typique sans monitoring :
─────────────────────────────────────────────────
09:00 ➜ "Tout va bien" (en apparence)
10:00 ➜ La base ralentit (personne ne le voit)
11:00 ➜ Le disque se remplit (toujours invisible)
12:00 ➜ 💥 CRASH : Disque plein, base inaccessible
12:01 ➜ Panique générale, clients impactés
12:05 ➜ "Depuis quand c'est comme ça ?"
12:10 ➜ "On aurait dû voir ça venir..."
─────────────────────────────────────────────────
Résultat : 2 heures de downtime, perte de revenus
```

**Avec monitoring, vous êtes proactif :**

```
Scénario avec monitoring :
─────────────────────────────────────────────────
09:00 ➜ Dashboard montre tout en vert
10:00 ➜ Alerte : "Disque à 70%, tendance inquiétante"
10:05 ➜ Investigation : logs volumineux trouvés
10:15 ➜ Action : nettoyage automatisé déclenché
10:20 ➜ Résolu : espace disque revenu à 45%
─────────────────────────────────────────────────
Résultat : Zéro downtime, problème anticipé
```

### Les bénéfices concrets du monitoring

#### 1. Détection précoce des problèmes

**Problèmes détectables** :
- 💾 Espace disque qui se remplit progressivement  
- 🔥 CPU qui monte anormalement  
- 🐌 Requêtes qui ralentissent avec le temps  
- 🔒 Verrous (locks) qui s'accumulent  
- 📊 Tables qui gonflent (bloat)

**Exemple** : Votre monitoring détecte que l'espace disque passe de 50% à 85% en une semaine. Vous avez le temps d'investiguer et d'agir **avant** d'atteindre 100%.

#### 2. Optimisation des performances

**Questions auxquelles le monitoring répond** :
- Quelles sont mes requêtes les plus lentes ?
- Quelles tables sont les plus sollicitées ?
- Mon cache est-il bien dimensionné ?
- Mes index sont-ils utilisés ?

**Impact** : Identifier qu'une seule requête lente peut améliorer les performances globales de 50%.

#### 3. Planification de capacité

**Tendances visibles** :
```
Janvier  : 100 connexions/jour, 50 GB de données  
Février  : 120 connexions/jour, 60 GB de données  
Mars     : 145 connexions/jour, 72 GB de données  

→ Projection : Juin = 200 connexions, 100 GB
→ Action : Planifier l'upgrade du serveur en Mai
```

**Bénéfice** : Anticiper les besoins plutôt que réagir dans l'urgence.

#### 4. Conformité et audit

**Exigences** :
- Tracer qui accède à quoi
- Prouver la disponibilité (SLA)
- Démontrer les performances
- Justifier les investissements infrastructure

**Exemple** : "Notre SLA garantit 99.9% de uptime" → Le monitoring fournit la preuve.

#### 5. Réduction du MTTR (Mean Time To Repair)

**Sans monitoring** :
```
Incident → Détection → Investigation → Reproduction → Résolution
           30 min      2 heures        1 heure      30 min
           ────────────────────────────────────────────────
           Total : 4 heures
```

**Avec monitoring** :
```
Incident → Alerte immédiate → Dashboard historique → Action ciblée
           < 1 min              5 min                15 min
           ─────────────────────────────────────────────────
           Total : 20 minutes
```

**Résultat** : 12× plus rapide à résoudre.

---

## Les dimensions du monitoring PostgreSQL

Le monitoring d'une base de données n'est pas unidimensionnel. Il faut surveiller plusieurs aspects en parallèle.

### 1. Métriques système (Infrastructure)

**Que surveille-t-on ?**
- 🖥️ **CPU** : Utilisation, load average, processus actifs  
- 💾 **Mémoire** : RAM utilisée, swap, cache  
- 💿 **Disque** : Espace libre, IOPS, latence, throughput  
- 🌐 **Réseau** : Bande passante, latence, paquets perdus

**Pourquoi c'est important ?**

Un problème PostgreSQL est souvent **causé par** un problème système :
- CPU à 100% → Requêtes lentes ou non optimisées
- Disque saturé → Checkpoints lents, requêtes bloquées
- Mémoire insuffisante → Swap, performances dégradées
- Réseau lent → Réplication en retard

**Exemple** :
```
Symptôme : Requêtes lentes depuis 1h  
Monitoring système montre : I/O disque à 100%  
Cause racine : Checkpoint trop fréquents (max_wal_size trop petit)  
Solution : Augmenter max_wal_size  
```

### 2. Métriques PostgreSQL (Base de données)

**Que surveille-t-on ?**
- 🔌 **Connexions** : Nombre actif, idle, max_connections  
- 📝 **Transactions** : Commits, rollbacks, taux  
- 🔍 **Requêtes** : Temps d'exécution, requêtes lentes  
- 💾 **Cache** : Hit ratio, lectures disque vs cache  
- 🔄 **Réplication** : Lag, état des replicas  
- 🧹 **Maintenance** : VACUUM, ANALYZE, bloat

**Pourquoi c'est important ?**

Ces métriques reflètent directement la **santé de votre base** :
- Connexions saturées → Application ne peut plus se connecter
- Cache hit ratio faible → Trop de lectures disque (lent)
- Réplication en retard → Données obsolètes sur les replicas

**Exemple** :
```
Symptôme : Application qui timeout  
Monitoring PostgreSQL montre : Connexions = 200/200 (saturé)  
Cause racine : Connection pooling mal configuré  
Solution : Déployer PgBouncer  
```

### 3. Logs (Événements)

**Que surveille-t-on ?**
- ❌ **Erreurs** : Deadlocks, violations de contraintes  
- 🐌 **Requêtes lentes** : duration > seuil  
- 🔐 **Connexions** : Tentatives échouées, authentification  
- 🔧 **Changements** : DDL (CREATE, ALTER, DROP)  
- 🚨 **Alertes** : Checkpoints, VACUUM, wraparound

**Pourquoi c'est important ?**

Les logs racontent **l'histoire** de ce qui s'est passé :
- Deadlock détecté → Conflit de verrous entre transactions
- Connexion refusée → Tentative d'intrusion ?
- Requête lente loggée → Candidat à l'optimisation

**Exemple** :
```
Log : "ERROR: deadlock detected"  
Log : "DETAIL: Process 12345 waits for ShareLock on transaction 67890"  
→ Investigation : Deux transactions se bloquent mutuellement
→ Solution : Revoir l'ordre des opérations dans le code
```

### 4. Métriques applicatives (APM)

**Que surveille-t-on ?**
- ⏱️ **Latence** : Temps de réponse des API  
- 📊 **Throughput** : Requêtes par seconde  
- 🎯 **Taux d'erreur** : 4xx, 5xx  
- 🔗 **Dépendances** : Appels entre services

**Pourquoi c'est important ?**

La base de données ne vit pas en isolation. Comprendre **l'impact applicatif** est crucial :
- Requête PostgreSQL lente → API timeout
- Cache applicatif inefficace → Surcharge PostgreSQL
- Pic de trafic → Saturation des connexions

**Exemple** :
```
APM montre : Endpoint /api/orders à 5s (normalement 200ms)  
Trace distribuée : 4.8s passées dans PostgreSQL  
Monitoring PostgreSQL : Requête sans index sur orders.date  
Solution : CREATE INDEX idx_orders_date ON orders(date)  
```

---

## Les catégories d'outils de monitoring

Il existe de nombreux outils pour monitorer PostgreSQL. Ils peuvent être classés en plusieurs catégories selon leur approche et leur objectif.

### 1. Outils d'analyse de logs

**Principe** : Analyser les fichiers de logs PostgreSQL pour en extraire des insights.

**Caractéristiques** :
- 📄 Analyse a posteriori (après que les événements se sont produits)  
- 🎨 Génèrent des rapports visuels (HTML, PDF)  
- 🔍 Excellent pour l'audit et le troubleshooting  
- 💾 Pas d'impact sur la base (analyse hors ligne)

**Outil principal** : **pgBadger**

**Cas d'usage** :
- Analyser les performances de la semaine dernière
- Identifier les requêtes qui ont posé problème
- Auditer les tentatives de connexion
- Comprendre les patterns d'utilisation

**Avantages** :
- ✅ Gratuit et open-source  
- ✅ Très détaillé  
- ✅ Facile à utiliser (un seul binaire)  
- ✅ Génère de beaux rapports HTML

**Limites** :
- ⏱️ Pas de temps réel (analyse post-mortem)  
- 📊 Nécessite que PostgreSQL logue suffisamment d'informations  
- 🔄 Doit être exécuté manuellement ou via cron

### 2. Extensions PostgreSQL internes

**Principe** : Extensions installées dans PostgreSQL qui collectent des métriques système.

**Caractéristiques** :
- 📊 Métriques en temps réel  
- 🔗 Intégration native avec PostgreSQL  
- 💡 Corrélation entre métriques SQL et système  
- ⚡ Léger impact sur les performances (<2%)

**Outil principal** : **pg_stat_kcache**

**Cas d'usage** :
- Déterminer si une requête est CPU-bound ou I/O-bound
- Mesurer la consommation réelle de ressources par requête
- Optimiser en fonction des métriques système
- Corréler problèmes PostgreSQL et infrastructure

**Avantages** :
- ✅ Métriques précises au niveau requête  
- ✅ Complète pg_stat_statements  
- ✅ Idéal pour l'optimisation fine  
- ✅ Open-source

**Limites** :
- 🐧 Linux uniquement (utilise /proc)  
- 🔧 Nécessite installation d'extension  
- 📊 Requiert pg_stat_statements  
- 💾 Métriques cumulatives (pas d'historique natif)

### 3. Stack de monitoring temps réel

**Principe** : Architecture complète pour collecter, stocker et visualiser les métriques en continu.

**Caractéristiques** :
- ⏰ Monitoring en temps réel (rafraîchissement toutes les 15-60s)  
- 📈 Historique des métriques (rétention configurable)  
- 🚨 Alerting automatique  
- 🎨 Dashboards interactifs

**Stack principal** : **Prometheus + postgres_exporter + Grafana**

**Cas d'usage** :
- Surveiller plusieurs instances PostgreSQL
- Créer des dashboards pour toute l'équipe
- Recevoir des alertes Slack/email en cas de problème
- Analyser les tendances sur plusieurs semaines/mois

**Avantages** :
- ✅ Standard de l'industrie (CNCF)  
- ✅ Open-source et gratuit  
- ✅ Très flexible et extensible  
- ✅ Grande communauté  
- ✅ Dashboards partagés (Grafana)

**Limites** :
- 🔧 Installation et configuration plus complexes  
- 🖥️ Infrastructure à gérer (serveurs Prometheus/Grafana)  
- 📚 Courbe d'apprentissage (PromQL)  
- 💾 Consommation de ressources (stockage des métriques)

### 4. Solutions cloud natives

**Principe** : Services de monitoring intégrés fournis par les clouds providers (AWS, Azure, GCP).

**Caractéristiques** :
- ☁️ Zéro infrastructure à gérer  
- 🔗 Intégration automatique avec les services cloud  
- 📊 Métriques collectées par défaut  
- 💳 Modèle de tarification pay-as-you-go

**Outils principaux** :
- **AWS CloudWatch** (avec Performance Insights)  
- **Azure Monitor** (avec Query Performance Insights)  
- **Google Cloud Monitoring** (avec Query Insights)

**Cas d'usage** :
- PostgreSQL hébergé sur RDS, Azure Database, Cloud SQL
- Monitoring multi-services (DB + apps + infra)
- Équipe sans expertise monitoring
- Besoin de démarrer rapidement

**Avantages** :
- ✅ Déploiement instantané  
- ✅ Scalabilité automatique  
- ✅ Maintenance zéro  
- ✅ Intégration native avec le cloud  
- ✅ Support professionnel

**Limites** :
- 💰 Coût potentiellement élevé  
- 🔒 Vendor lock-in  
- 🎨 Moins de flexibilité qu'une solution open-source  
- 📊 Limité aux services cloud du provider

---

## Tableau comparatif des approches

| Critère | Analyse logs (pgBadger) | Extensions (pg_stat_kcache) | Stack temps réel (Prometheus) | Cloud natives (CloudWatch) |
|---------|-------------------------|----------------------------|-------------------------------|---------------------------|
| **Complexité** | ⭐ Très simple | ⭐⭐ Simple | ⭐⭐⭐⭐ Avancée | ⭐⭐ Simple |
| **Coût** | Gratuit | Gratuit | Gratuit (infra à gérer) | Payant |
| **Temps réel** | ❌ Non (post-mortem) | ✅ Oui | ✅ Oui | ✅ Oui |
| **Historique** | ✅ Illimité (logs) | ❌ Non (cumulatif) | ✅ Configurable | ✅ Configurable |
| **Alerting** | ❌ Non | ❌ Non | ✅ Oui | ✅ Oui |
| **Maintenance** | ⭐ Aucune | ⭐ Minimale | ⭐⭐⭐⭐ Importante | ⭐ Aucune |
| **Multi-instances** | ⭐⭐ Possible | ⭐⭐⭐ Possible | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ Excellent |
| **Visualisation** | 📄 Rapports HTML statiques | 📊 SQL queries | 📈 Dashboards Grafana | 📊 Dashboards cloud |
| **Performance impact** | ✅ Aucun (hors ligne) | ⚠️ Léger (~1-2%) | ⚠️ Léger (~1-2%) | ⚠️ Léger |
| **Courbe d'apprentissage** | ⭐ Facile | ⭐⭐ Moyenne | ⭐⭐⭐⭐ Élevée | ⭐⭐ Moyenne |

---

## Quelle approche choisir ?

Il n'y a pas **une seule bonne réponse**. Le choix dépend de votre contexte.

### Recommandations par contexte

#### Startup / Petit projet

**Besoins** : Simple, gratuit, démarrage rapide

**Recommandation** :
```
Base : pg_stat_statements (intégré PostgreSQL)
↓
Analyse ponctuelle : pgBadger (quand problème)
↓
Si PostgreSQL sur cloud : Utiliser CloudWatch/Azure Monitor/GCP
```

**Pourquoi** : Minimal viable, zéro coût, efficace.

#### Projet en croissance

**Besoins** : Monitoring continu, alerting, équipe qui grandit

**Recommandation** :
```
Stack : Prometheus + postgres_exporter + Grafana
↓
Compléter avec : pgBadger pour analyses approfondies
↓
+ pg_stat_kcache si besoin d'optimisation fine
```

**Pourquoi** : Évolutif, pas de vendor lock-in, gratuit.

#### Entreprise / Production critique

**Besoins** : Fiabilité, SLA, support, multi-environnements

**Recommandation** :
```
Option A (Cloud) : CloudWatch/Azure Monitor/GCP (selon cloud)  
Option B (Hybride) : Prometheus + Grafana + pgBadger  
Option C (Commercial) : Datadog, New Relic (APM complet)  
```

**Pourquoi** : Support professionnel, SLA garantis, features avancées.

#### Multi-cloud / Hybride

**Besoins** : Vue unifiée sur plusieurs clouds/on-premise

**Recommandation** :
```
Hub central : Grafana (avec plusieurs data sources)
↓
Data sources : Prometheus (on-prem) + CloudWatch (AWS) + Azure Monitor
↓
OU : Solution tierce (Datadog, New Relic) pour unification complète
```

**Pourquoi** : Seule façon d'avoir une vue cohérente.

---

## Stratégie de monitoring complète

**L'idéal n'est pas de choisir UN outil, mais de combiner plusieurs approches** en fonction de leurs forces respectives.

### Stack recommandée (Production)

```
┌─────────────────────────────────────────────────────────┐
│                    Monitoring complet                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Temps réel & Alerting                                  │
│  ┌────────────────────────────────────────────┐         │
│  │ Prometheus + Grafana                       │         │
│  │ • Dashboards live                          │         │
│  │ • Alertes Slack/email                      │         │
│  │ • Historique 30 jours                      │         │
│  └────────────────────────────────────────────┘         │
│                                                         │
│  Analyse approfondie                                    │
│  ┌────────────────────────────────────────────┐         │
│  │ pgBadger (hebdomadaire)                    │         │
│  │ • Analyse détaillée des requêtes           │         │
│  │ • Audit de sécurité                        │         │
│  │ • Tendances sur la semaine                 │         │
│  └────────────────────────────────────────────┘         │
│                                                         │
│  Optimisation fine                                      │
│  ┌────────────────────────────────────────────┐         │
│  │ pg_stat_kcache                             │         │
│  │ • Métriques CPU/I/O par requête            │         │
│  │ • Identification CPU-bound vs I/O-bound    │         │
│  └────────────────────────────────────────────┘         │
│                                                         │
│  Métriques système                                      │
│  ┌────────────────────────────────────────────┐         │
│  │ node_exporter (Prometheus)                 │         │
│  │ • CPU, RAM, Disque, Réseau                 │         │
│  │ • Corrélation infra/PostgreSQL             │         │
│  └────────────────────────────────────────────┘         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Workflow type

**1. Monitoring quotidien** :
- Consulter les dashboards Grafana
- Vérifier les alertes (s'il y en a)
- Observer les tendances (CPU, connexions, cache hit ratio)

**2. Analyse hebdomadaire** :
- Générer un rapport pgBadger
- Identifier les nouvelles requêtes lentes
- Comparer avec la semaine précédente

**3. Optimisation ciblée** :
- Utiliser pg_stat_kcache pour analyser une requête spécifique
- Déterminer si le problème est CPU, I/O, ou autre
- Appliquer l'optimisation appropriée

**4. Planification mensuelle** :
- Analyser les tendances de croissance
- Anticiper les besoins futurs (CPU, RAM, Disque)
- Ajuster les ressources si nécessaire

---

## Les métriques essentielles à surveiller

Quel que soit l'outil choisi, certaines métriques sont **incontournables**.

### Métriques critiques (alertes obligatoires)

| Métrique | Seuil alerte | Impact si dépassé |
|----------|--------------|-------------------|
| **Espace disque disponible** | < 20% | Base inaccessible si 0% |
| **CPU utilisation** | > 80% sur 10min | Ralentissement généralisé |
| **Connexions actives** | > 80% de max_connections | Nouvelles connexions refusées |
| **Réplication lag** | > 60 secondes | Données obsolètes sur replicas |
| **Cache hit ratio** | < 90% | Performances dégradées (I/O) |

### Métriques importantes (surveillance régulière)

| Métrique | Valeur cible | Pourquoi |
|----------|--------------|----------|
| **Transactions committées** | Stable ou croissant | Santé de l'activité |
| **Transactions rollbackées** | < 5% du total | Problèmes applicatifs |
| **Checkpoints forcés** | < 20% du total | Configuration WAL sous-optimale |
| **Dead tuples ratio** | < 10% | Besoin de VACUUM |
| **Temps moyen de requête** | < 100ms | Performances utilisateur |

### Métriques de tendance (planification)

| Métrique | Observation | Action |
|----------|-------------|--------|
| **Croissance de la base** | GB/jour, GB/mois | Prévoir scaling disque |
| **Évolution des connexions** | Pics, moyennes | Dimensionner connection pool |
| **Patterns d'utilisation** | Heures de pointe | Planifier maintenance hors pics |
| **Requêtes lentes** | Nouvelles vs récurrentes | Prioriser optimisations |

---

## Préparer PostgreSQL pour le monitoring

Avant de déployer des outils de monitoring, il faut **configurer PostgreSQL** pour qu'il expose les bonnes informations.

### Configuration minimale recommandée

**Dans postgresql.conf** :

```ini
# ============================================
# Configuration pour le monitoring
# ============================================

# 1. Logging
logging_collector = on  
log_destination = 'stderr'  
log_directory = 'log'  
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  
log_rotation_age = 1d  
log_rotation_size = 100MB  

# Format des logs (pour pgBadger et autres)
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Quoi logger
log_connections = on  
log_disconnections = on  
log_duration = off  # off car on utilise log_min_duration_statement  
log_min_duration_statement = 1000  # 1 seconde (ajuster selon contexte)  
log_checkpoints = on  
log_lock_waits = on  
log_temp_files = 0  # Logger tous les fichiers temporaires  

# 2. Statistiques (pg_stat_statements)
shared_preload_libraries = 'pg_stat_statements'  
pg_stat_statements.max = 10000  
pg_stat_statements.track = all  
pg_stat_statements.track_utility = on  

# 3. Autres paramètres utiles
track_activities = on  
track_counts = on  
track_io_timing = on  # Peut avoir léger impact, mais très utile  
track_functions = all  
```

**Pourquoi chaque paramètre** :

- `logging_collector = on` : Active la collecte de logs (essentiel pour pgBadger)  
- `log_min_duration_statement` : Logger les requêtes lentes uniquement (réduit le volume)  
- `log_checkpoints` : Comprendre les patterns de checkpoints  
- `pg_stat_statements` : Extension de base pour toutes les statistiques de requêtes  
- `track_io_timing` : Mesurer les temps I/O (crucial pour optimisation)

### Créer l'extension pg_stat_statements

```sql
-- Se connecter en tant que superuser
psql -U postgres -d mydatabase

-- Créer l'extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Vérifier
SELECT * FROM pg_stat_statements LIMIT 5;
```

### Redémarrer PostgreSQL

```bash
# Redémarrage nécessaire pour shared_preload_libraries
sudo systemctl restart postgresql

# Ou
sudo pg_ctlcluster 18 main restart
```

---

## Les sections suivantes

Dans les sous-sections qui suivent, nous allons explorer en détail chaque catégorie d'outils :

### 14.7.1. pgBadger : Analyse de logs

**Vous apprendrez** :
- Installer et configurer pgBadger
- Générer des rapports HTML détaillés
- Identifier les requêtes lentes et les problèmes
- Automatiser l'analyse quotidienne

**Cas d'usage principal** : Analyse post-mortem, audit, compréhension des patterns.

### 14.7.2. pg_stat_kcache : Métriques système

**Vous apprendrez** :
- Installer l'extension pg_stat_kcache
- Collecter les métriques CPU et I/O par requête
- Distinguer les requêtes CPU-bound des requêtes I/O-bound
- Optimiser en fonction des métriques système

**Cas d'usage principal** : Optimisation fine des requêtes, corrélation SQL/système.

### 14.7.3. Prometheus + postgres_exporter + Grafana

**Vous apprendrez** :
- Déployer la stack complète (Docker ou natif)
- Configurer postgres_exporter pour PostgreSQL
- Créer des dashboards Grafana personnalisés
- Mettre en place des alertes automatiques

**Cas d'usage principal** : Monitoring temps réel, alerting, dashboards équipe.

### 14.7.4. Solutions cloud natives

**Vous apprendrez** :
- Utiliser AWS CloudWatch pour RDS PostgreSQL
- Configurer Azure Monitor pour Azure Database
- Exploiter GCP Cloud Monitoring pour Cloud SQL
- Comparer les trois solutions et choisir la bonne

**Cas d'usage principal** : PostgreSQL hébergé sur le cloud, zéro infrastructure.

---

## Checklist avant de commencer

Avant de plonger dans les outils, assurez-vous d'avoir :

**Prérequis techniques** :
- [ ] PostgreSQL 18 installé et fonctionnel  
- [ ] Accès superuser ou privilèges suffisants  
- [ ] Configuration postgresql.conf éditable  
- [ ] Possibilité de redémarrer PostgreSQL  
- [ ] Espace disque pour les logs (au moins 10GB)

**Connaissances** :
- [ ] Bases de SQL (SELECT, FROM, WHERE)  
- [ ] Notions de performance (index, EXPLAIN)  
- [ ] Ligne de commande Linux (cd, ls, cat)  
- [ ] Concepts de monitoring (métriques, alertes, dashboards)

**Environnement recommandé** :
- [ ] Environnement de test/staging (ne pas commencer en production)  
- [ ] Données réalistes (pour voir des métriques significatives)  
- [ ] Navigateur web (pour Grafana, pgBadger)  
- [ ] Éditeur de texte (vim, nano, VSCode)

---

## Conclusion de l'introduction

Le monitoring n'est pas un luxe, c'est une **nécessité absolue** pour toute base de données en production. Sans lui, vous pilotez à l'aveugle et vous découvrirez les problèmes quand il sera trop tard.

**Les points clés à retenir** :

- ✅ **Monitoring = Visibilité** : Vous ne pouvez pas améliorer ce que vous ne mesurez pas  
- ✅ **Proactif > Réactif** : Détecter avant que ça casse  
- ✅ **Plusieurs outils** : Combiner les approches selon les besoins  
- ✅ **Commencer simple** : pgBadger + pg_stat_statements suffit pour débuter  
- ✅ **Évoluer progressivement** : Ajouter Prometheus/Grafana quand le besoin se fait sentir

**Dans les sections suivantes**, nous allons passer de la théorie à la pratique en explorant chaque outil en détail, avec des exemples concrets et des configurations prêtes à l'emploi.

**Prêt à prendre le contrôle de votre PostgreSQL ?** Passons à la première section : pgBadger.

---

## Ressources complémentaires

### Documentation officielle PostgreSQL

- **Monitoring Database Activity** : https://www.postgresql.org/docs/current/monitoring.html  
- **Statistics Collector** : https://www.postgresql.org/docs/current/monitoring-stats.html  
- **pg_stat_statements** : https://www.postgresql.org/docs/current/pgstatstatements.html

### Livres recommandés

- "PostgreSQL: Up and Running" - Regina Obe, Leo Hsu (Chapitre Monitoring)  
- "PostgreSQL 14 Administration Cookbook" - Simon Riggs (Recipes monitoring)  
- "The Art of PostgreSQL" - Dimitri Fontaine (Performance monitoring)

### Blogs et articles

- **2ndQuadrant Blog** : https://www.2ndquadrant.com/en/blog/ (excellent contenu monitoring)  
- **Percona PostgreSQL Blog** : https://www.percona.com/blog/category/postgresql/  
- **Cybertec PostgreSQL** : https://www.cybertec-postgresql.com/en/blog/

### Communautés

- **PostgreSQL Mailing Lists** : pgsql-general, pgsql-performance  
- **Reddit** : r/PostgreSQL  
- **Discord** : Serveurs PostgreSQL francophones et anglophones  
- **Stack Overflow** : Tag [postgresql]

---


⏭️ [pgBadger : Analyse de logs](/14-observabilite-et-monitoring/07.1-pgbadger.md)
