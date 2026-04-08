🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe D : Configuration de Référence par Cas d'Usage

## Introduction

PostgreSQL est un système de gestion de base de données extrêmement flexible qui peut s'adapter à une grande variété de charges de travail. Cependant, cette flexibilité signifie qu'il n'existe pas de configuration "universelle" optimale. **Une configuration idéale pour une application dépend du type de charge de travail qu'elle doit gérer.**

Cette annexe présente des configurations de référence **prêtes à l'emploi** pour les principaux cas d'usage de PostgreSQL. Chaque configuration est expliquée en détail avec :

- Les objectifs et priorités pour ce cas d'usage
- Les paramètres critiques et leurs valeurs recommandées
- Les compromis et décisions de conception
- Des exemples concrets d'application
- Des checklist de vérification

---

## Qu'est-ce qu'un "Cas d'Usage" ?

Un **cas d'usage** (ou *workload* en anglais) définit le **type de charge de travail** que votre base de données doit supporter. Il détermine :

- **Le type d'opérations** : Lectures, écritures, ou les deux ?  
- **La fréquence** : Milliers de requêtes par seconde ou quelques analyses par jour ?  
- **La complexité** : Requêtes simples ou agrégations massives ?  
- **La latence acceptable** : Millisecondes ou minutes ?  
- **Le volume de données** : Gigaoctets ou téraoctets ?

### Pourquoi la Configuration doit-elle s'Adapter au Cas d'Usage ?

PostgreSQL utilise des **ressources limitées** (RAM, CPU, disque). Selon votre cas d'usage, vous devez **prioriser** certains aspects :

| Ressource | OLTP | OLAP | Mixed | Dev Local |
|-----------|------|------|-------|-----------|
| **RAM** | Cache données chaudes | Tris et agrégations massifs | Compromis | Économiser pour IDE |
| **CPU** | Parallélisme limité | Parallélisme maximal | Modéré | Limité (laptop) |
| **Disque** | Écritures fréquentes (WAL) | Lectures séquentielles massives | Équilibré | Performance > durabilité |
| **Latence** | **Critique** (<100ms) | Acceptable (secondes) | Critique pour OLTP | Non critique |
| **Débit** | Élevé (transactions/sec) | **Critique** (GB/sec) | Équilibré | Faible |

**Exemple concret :**

Imaginons un paramètre simple : `work_mem` (mémoire pour tris et groupements).

- **OLTP** : `work_mem = 32 MB`
  - Pourquoi ? Beaucoup de connexions (200-500), peu de mémoire par requête
  - Requêtes simples (1-1000 lignes), pas de gros tris

- **OLAP** : `work_mem = 1 GB`
  - Pourquoi ? Peu de connexions (20-50), beaucoup de mémoire disponible
  - Requêtes complexes (millions de lignes), tris et groupements massifs

- **Mixed** : `work_mem = 64 MB` (global) avec ajustements par rôle
  - Pourquoi ? Compromis entre les deux
  - Rôles OLTP : 32 MB, Rôles OLAP : 512 MB-1 GB

- **Dev Local** : `work_mem = 64 MB`
  - Pourquoi ? Confort de développement, mais économiser RAM pour l'IDE

**Une seule valeur de `work_mem` ne peut pas être optimale pour tous les cas !**

---

## Les Quatre Cas d'Usage Principaux

Cette annexe couvre les quatre configurations les plus courantes en production et développement :

### 1. OLTP (Online Transaction Processing)

**🎯 Objectif :** Gérer des milliers de transactions courtes par seconde avec une latence minimale.

**Caractéristiques :**
- **Opérations** : INSERT, UPDATE, DELETE, SELECT simples  
- **Volume par requête** : 1-1000 lignes  
- **Fréquence** : Très élevée (milliers/seconde)  
- **Latence** : < 100ms (critique)  
- **Concurrence** : Très élevée (200-500 connexions)  
- **Exemples** : Applications Web, e-commerce, API REST, applications mobiles

**Priorités de configuration :**
1. **Latence faible** : Minimiser le temps de réponse  
2. **Haute concurrence** : Gérer beaucoup de connexions simultanées  
3. **Écritures fréquentes** : Optimiser WAL et checkpoints  
4. **Cache efficace** : Maximiser le cache hit ratio  
5. **Autovacuum agressif** : Nettoyer rapidement les tuples morts

**Compromis acceptés :**
- Parallélisation limitée (overhead pour petites requêtes)
- work_mem faible (beaucoup de connexions)
- Pas de grandes agrégations (pas le cas d'usage)

---

### 2. OLAP (Online Analytical Processing)

**🎯 Objectif :** Analyser et agréger des millions/milliards de lignes avec un débit de données maximal.

**Caractéristiques :**
- **Opérations** : SELECT complexes avec GROUP BY, JOIN multiples, agrégations  
- **Volume par requête** : Millions à milliards de lignes  
- **Fréquence** : Faible (quelques requêtes/heure)  
- **Latence** : Secondes à minutes (acceptable)  
- **Concurrence** : Faible (10-50 connexions)  
- **Exemples** : Data warehouse, BI, rapports, analytics

**Priorités de configuration :**
1. **Débit élevé** : Lire et traiter des GB/s de données  
2. **Parallélisation maximale** : Utiliser tous les CPU  
3. **Mémoire généreuse** : work_mem élevé pour tris et groupements  
4. **Scan séquentiel optimisé** : Lire de grandes portions de tables  
5. **Statistiques détaillées** : Plans de requête optimaux

**Compromis acceptés :**
- Latence plus élevée (requêtes de 10-60 secondes OK)
- Peu de connexions simultanées
- Autovacuum moins agressif

---

### 3. Mixed Workload (Charge Mixte)

**🎯 Objectif :** Équilibrer OLTP et OLAP sur la même instance.

**Caractéristiques :**
- **Opérations** : Mix de transactions rapides ET d'analyses complexes  
- **Volume** : Variable selon le type de requête  
- **Fréquence** : Élevée pour OLTP, faible pour OLAP  
- **Latence** : Critique pour OLTP, acceptable pour OLAP  
- **Concurrence** : Moyenne à élevée  
- **Exemples** : Applications SaaS avec dashboards intégrés, CRM, ERP

**Priorités de configuration :**
1. **Compromis intelligent** : 70% OLTP, 30% OLAP  
2. **Séparation logique** : Rôles et pools différenciés  
3. **Ajustements dynamiques** : work_mem par rôle/session  
4. **Protection OLTP** : Empêcher OLAP de bloquer OLTP  
5. **Monitoring différencié** : Distinguer les deux charges

**Compromis nécessaires :**
- Aucune des deux charges n'est 100% optimisée
- Configuration plus complexe
- Monitoring intensif requis

**Architecture recommandée :**
- Si possible : Séparation physique (réplication)
- Si budget limité : Séparation logique (rôles, pooling)

---

### 4. Développement Local

**🎯 Objectif :** Environnement simple et confortable pour développer et apprendre.

**Caractéristiques :**
- **Matériel** : Laptop (8-32 GB RAM, 4-8 CPU)  
- **Utilisateurs** : Un seul développeur  
- **Données** : Volume faible, données de test  
- **Sécurité** : Minimale (pas de vraies données)  
- **Durabilité** : Optionnelle (accepter perte si crash)  
- **Exemples** : Développement d'applications, apprentissage SQL, tests

**Priorités de configuration :**
1. **Simplicité** : Configuration minimale, compréhensible  
2. **Feedback rapide** : Voir immédiatement les résultats  
3. **Logs détaillés** : Apprendre en observant PostgreSQL  
4. **Performance raisonnable** : Confort sans surcharger le laptop  
5. **Économie de ressources** : Laisser de la RAM pour l'IDE

**Compromis acceptés :**
- Durabilité sacrifiée (fsync = off ⚠️)
- Sécurité minimale (trust authentication)
- Pas d'optimisation extrême
- Configuration non-recommandée pour production

---

## Comment Utiliser Cette Annexe ?

### Étape 1 : Identifier Votre Cas d'Usage

Posez-vous ces questions :

1. **Quel est le type principal d'opérations ?**
   - Transactions courtes (INSERT/UPDATE/DELETE) → **OLTP**
   - Analyses et rapports (SELECT complexes) → **OLAP**
   - Les deux en proportions significatives → **Mixed**
   - Développement sur laptop → **Dev Local**

2. **Quelle est la latence acceptable ?**  
   - < 100ms obligatoire → **OLTP**
   - Quelques secondes OK → **OLAP**
   - Variable selon l'opération → **Mixed**
   - Non critique → **Dev Local**

3. **Combien d'utilisateurs simultanés ?**
   - Centaines/milliers → **OLTP**
   - Quelques dizaines → **OLAP**
   - Mix → **Mixed**
   - Un seul (vous) → **Dev Local**

4. **Quel est le volume de données traité par requête ?**
   - 1-1000 lignes → **OLTP**
   - Millions de lignes → **OLAP**
   - Variable → **Mixed**
   - Faible (tests) → **Dev Local**

### Étape 2 : Appliquer la Configuration Recommandée

Chaque section de cette annexe fournit :

1. **Configuration complète** : Fichier postgresql.conf prêt à l'emploi  
2. **Explications détaillées** : Pourquoi chaque paramètre a cette valeur  
3. **Dimensionnement** : Comment adapter selon votre matériel  
4. **Optimisations supplémentaires** : Index, partitionnement, etc.  
5. **Monitoring** : Métriques à surveiller pour ce cas d'usage  
6. **Checklist** : Points de vérification avant mise en production

### Étape 3 : Mesurer et Ajuster

**Principe fondamental :** Les configurations de cette annexe sont des **points de départ**, pas des vérités absolues.

**Processus itératif :**

1. **Appliquer** la configuration recommandée  
2. **Mesurer** les performances (voir section Monitoring)  
3. **Identifier** les goulots d'étranglement  
4. **Ajuster** les paramètres progressivement  
5. **Valider** l'amélioration  
6. Répéter

**Outils de mesure :**
- `pg_stat_statements` : Analyser les requêtes  
- `EXPLAIN ANALYZE` : Comprendre les plans
- Monitoring système : CPU, RAM, I/O
- Logs PostgreSQL : Détecter anomalies

---

## Tableau Récapitulatif des Configurations

### Ressources Matérielles (Exemple : 64 GB RAM, 16 CPU, SSD)

| Paramètre | OLTP | OLAP | Mixed | Dev Local |
|-----------|------|------|-------|-----------|
| **shared_buffers** | 16 GB (25%) | 16 GB (25%) | 16 GB (25%) | 512 MB (3%) |
| **effective_cache_size** | 48 GB (75%) | 48 GB (75%) | 48 GB (75%) | 4 GB (25%) |
| **work_mem** | 32 MB | 1 GB | 64 MB* | 64 MB |
| **maintenance_work_mem** | 2 GB | 4 GB | 2 GB | 256 MB |
| **max_connections** | 300 | 50 | 250 | 20 |
| **max_parallel_workers_per_gather** | 2 | 8 | 4 | 2 |
| **max_parallel_workers** | 8 | 16 | 12 | 4 |

*\*Mixed : 64 MB global, mais ajusté par rôle (OLTP: 32 MB, OLAP: 512 MB)*

### Paramètres Critiques

| Paramètre | OLTP | OLAP | Mixed | Dev Local |
|-----------|------|------|-------|-----------|
| **fsync** | **on** | **on** | **on** | **off** ⚠️ |
| **synchronous_commit** | on | on | on | off ⚠️ |
| **checkpoint_timeout** | 15min | 30min | 15min | 30min |
| **autovacuum_naptime** | 10s | 5min | 30s | 1min |
| **log_statement** | none | none | none | **all** |
| **log_min_duration_statement** | 100ms | 5s | 1s | 0 |

### Optimisations Spécifiques

| Aspect | OLTP | OLAP | Mixed | Dev Local |
|--------|------|------|-------|-----------|
| **Index** | B-Tree partout | B-Tree + BRIN | Mixte | B-Tree basique |
| **Partitionnement** | Optionnel | **Obligatoire** | Recommandé | Non nécessaire |
| **Vues matérialisées** | Rare | **Fréquent** | Pour OLAP | Non nécessaire |
| **Connection pooling** | **Obligatoire** | Optionnel | **Obligatoire** | Non nécessaire |
| **Réplication** | HA recommandée | Read replica OLAP | Séparation OLTP/OLAP | Non applicable |

---

## Avertissements et Bonnes Pratiques

### ⚠️ Avertissements Critiques

1. **Ne JAMAIS copier une config de production vers développement**
   - Config prod optimisée pour 128 GB RAM → crash sur laptop 16 GB
   - Toujours utiliser la config appropriée au contexte

2. **Ne JAMAIS utiliser la config dev en production**  
   - `fsync = off` cause des pertes de données si crash  
   - `log_statement = all` remplit le disque en quelques heures
   - Authentification `trust` = aucune sécurité

3. **Ne JAMAIS modifier tous les paramètres en même temps**
   - Changer un paramètre à la fois
   - Mesurer l'impact avant le suivant
   - Documenter chaque modification

4. **Ne JAMAIS ignorer le monitoring**
   - Configuration sans mesure = optimisation à l'aveugle
   - Activer `pg_stat_statements` dès le début
   - Surveiller : CPU, RAM, I/O, cache hit ratio, requêtes lentes

### ✅ Bonnes Pratiques

1. **Commencer conservateur, augmenter progressivement**
   - Valeurs par défaut PostgreSQL sont déjà bonnes
   - Augmenter un paramètre seulement si monitoring le justifie

2. **Documenter votre configuration**
   - Pourquoi chaque paramètre a cette valeur
   - Quand et pourquoi il a été modifié
   - Impact observé de la modification

3. **Tester en environnement de staging**
   - Jamais de changement direct en production
   - Valider impact sur charge réelle
   - Avoir un plan de rollback

4. **Automatiser avec des outils**  
   - **PGTune** : https://pgtune.leopard.in.ua/
     - Génère config selon matériel et cas d'usage
     - Bon point de départ
   - **pgtop** : Monitoring temps réel  
   - **pgBadger** : Analyse de logs

5. **Maintenir la configuration à jour**
   - Réviser tous les 6 mois
   - Ajuster si matériel change
   - Adapter si charge évolue

---

## Comprendre les Compromis

Chaque configuration implique des **compromis**. Il n'existe pas de "meilleure configuration universelle".

### Exemple : Parallélisation

**OLTP :**
- ❌ Parallélisation faible (`max_parallel_workers_per_gather = 2`)  
- ✅ Pourquoi ? Requêtes courtes, overhead de parallélisation > gain  
- ✅ Avantage : Plus de CPU disponible pour gérer concurrence

**OLAP :**
- ✅ Parallélisation maximale (`max_parallel_workers_per_gather = 8`)  
- ✅ Pourquoi ? Requêtes longues sur millions de lignes  
- ✅ Avantage : Diviser le travail = requête 5-8× plus rapide  
- ❌ Inconvénient : Moins de ressources pour autres requêtes

**Mixed :**
- 🤝 Parallélisation modérée (`max_parallel_workers_per_gather = 4`)  
- 🤝 Compromis entre les deux  
- 🤝 + Ajustement par rôle (OLTP: 0, OLAP: 8)

### Exemple : work_mem

**OLTP :**
- ❌ work_mem faible (32 MB)  
- ✅ Pourquoi ? 300 connexions × 32 MB × 3 ops = 28 GB max (acceptable)  
- ❌ Inconvénient : Grandes agrégations écrivent sur disque (rare en OLTP)

**OLAP :**
- ✅ work_mem élevé (1 GB)  
- ✅ Pourquoi ? 50 connexions × 1 GB × 2 ops = 100 GB max (mais rarement atteint)  
- ✅ Avantage : Tris et groupements en RAM (10-100× plus rapide)  
- ❌ Risque : OOM si trop de connexions utilisent 1 GB simultanément

**Solution Mixed :**
- 🤝 work_mem global modéré (64 MB)  
- 🤝 + Ajustement par rôle (OLTP: 32 MB, OLAP: 512 MB-1 GB)  
- 🤝 Protection : timeouts pour limiter durée requêtes OLAP

---

## Évolution et Migration entre Configurations

### Quand Changer de Configuration ?

**Signaux indiquant un changement nécessaire :**

1. **OLTP → Mixed**
   - Apparition de rapports/dashboards intégrés
   - Utilisateurs demandent des analyses
   - Requêtes de plus en plus complexes et longues

2. **Mixed → Séparation (OLTP + OLAP)**
   - OLAP consomme > 30% des ressources
   - Latence OLTP dégradée par requêtes OLAP
   - Locks fréquents (OLAP bloque OLTP)
   - Budget permet un second serveur

3. **Dev → Production**
   - Application prête à déployer
   - Besoin de durabilité et sécurité
   - Charge réelle (utilisateurs)

### Processus de Migration

1. **Identifier le nouveau cas d'usage**  
2. **Préparer nouvelle configuration** (fichier postgresql.conf)  
3. **Tester en staging** avec charge simulée  
4. **Planifier fenêtre de maintenance**  
5. **Appliquer configuration** + redémarrage  
6. **Monitorer intensément** pendant 24-48h  
7. **Ajuster si nécessaire**

**Exemple : Dev → Production**

```bash
# 1. Sauvegarder config actuelle
cp /etc/postgresql/18/main/postgresql.conf postgresql.conf.dev

# 2. Appliquer config production OLTP
cp postgresql.conf.oltp /etc/postgresql/18/main/postgresql.conf

# 3. Vérifier syntaxe
/usr/lib/postgresql/18/bin/postgres -C config_file=/etc/postgresql/18/main/postgresql.conf

# 4. Redémarrer
sudo systemctl restart postgresql@18-main

# 5. Vérifier paramètres actifs
psql -c "SHOW shared_buffers;"  
psql -c "SHOW work_mem;"  
psql -c "SHOW fsync;"  # DOIT être 'on' en production !  
```

---

## Organisation de Cette Annexe

Cette annexe est divisée en **quatre sections principales**, une pour chaque cas d'usage :

### Section 1 : OLTP (High Concurrency, Low Latency)
- Configuration complète pour charges transactionnelles
- Optimisation latence et concurrence
- Connection pooling (PgBouncer)
- Index et autovacuum agressif
- Monitoring cache hit ratio et requêtes lentes

### Section 2 : OLAP (Data Warehouse, Analytics)
- Configuration complète pour charges analytiques
- Optimisation débit et parallélisation
- Partitionnement et index BRIN
- Vues matérialisées
- Monitoring fichiers temporaires et parallélisation

### Section 3 : Mixed Workload (Équilibre OLTP/OLAP)
- Configuration compromis intelligente
- Séparation logique par rôles
- Ajustements dynamiques work_mem
- Protection OLTP contre OLAP
- Monitoring différencié par type de charge

### Section 4 : Développement Local
- Configuration simplifiée pour laptop
- Logs détaillés pour apprentissage
- Scripts de seed et reset
- Intégration avec outils de développement
- Outils graphiques (pgAdmin, DBeaver)

---

## Avant de Commencer

### Prérequis

1. **PostgreSQL 18 installé**
   - Version 18 recommandée (dernière version stable)
   - Versions 16-17 compatibles (quelques paramètres diffèrent)

2. **Accès superutilisateur**
   - Modifier postgresql.conf
   - Modifier pg_hba.conf
   - Redémarrer PostgreSQL

3. **Connaissances de base**
   - SQL fondamental
   - Ligne de commande (bash/powershell)
   - Concepts PostgreSQL de base (tables, index, transactions)

4. **Outils recommandés**  
   - `psql` (client ligne de commande)
   - Éditeur de texte
   - Outil graphique optionnel (pgAdmin, DBeaver)

### Structure des Fichiers

Chaque configuration est fournie dans un fichier markdown séparé :

```
annexe-d-configurations/
├── 00-introduction.md                          (ce fichier)
├── 01-configuration-oltp.md                    (Section 1)
├── 02-configuration-olap.md                    (Section 2)
├── 03-configuration-mixed-workload.md          (Section 3)
└── 04-configuration-developpement-local.md     (Section 4)
```

**Chaque section contient :**
- Introduction au cas d'usage
- Configuration postgresql.conf complète
- Explications détaillées de chaque paramètre
- Optimisations supplémentaires (index, partitionnement, etc.)
- Requêtes de monitoring
- Checklist de validation
- Erreurs courantes à éviter

---

## Comment Lire les Configurations

### Format des Paramètres

Les configurations utilisent ce format :

```ini
# Commentaire explicatif
parametre_postgresql = valeur    # Raison de cette valeur
```

**Exemple :**
```ini
# Mémoire pour opérations de tri et groupement
work_mem = 32MB                  # Faible car beaucoup de connexions (OLTP)
```

### Symboles Utilisés

- ✅ **Recommandé** : Configuration idéale pour ce cas d'usage  
- ❌ **Déconseillé** : À éviter pour ce cas d'usage  
- ⚠️ **Attention** : Paramètre critique, lire avertissement  
- 🤝 **Compromis** : Équilibre entre plusieurs objectifs  
- 💡 **Astuce** : Conseil pratique  
- 🐛 **Debug** : Aide au débogage

### Niveaux de Priorité

Les paramètres sont classés par importance :

1. **🔴 CRITIQUE** : Impact majeur, à configurer en premier  
2. **🟠 IMPORTANT** : Impact significatif, à configurer rapidement  
3. **🟡 RECOMMANDÉ** : Amélioration notable, configurer si temps  
4. **🟢 OPTIONNEL** : Amélioration mineure, configurer si besoin spécifique

---

## Ressources Complémentaires

### Documentation Officielle PostgreSQL
- **Configuration** : https://www.postgresql.org/docs/18/runtime-config.html  
- **Performance Tips** : https://www.postgresql.org/docs/18/performance-tips.html  
- **Monitoring** : https://www.postgresql.org/docs/18/monitoring.html

### Outils de Configuration
- **PGTune** : https://pgtune.leopard.in.ua/  
- **pgBench** : Outil de benchmark intégré à PostgreSQL  
- **pgBadger** : Analyse de logs

### Communauté
- **Mailing list pgsql-performance** : Discussions sur la performance  
- **Reddit r/PostgreSQL** : Communauté active  
- **Stack Overflow** : Tag [postgresql-performance]

### Blogs Recommandés
- **Percona Blog** : https://www.percona.com/blog/  
- **CrunchyData Blog** : https://www.crunchydata.com/blog  
- **2ndQuadrant Blog** : Experts PostgreSQL

---

## Prêt à Commencer ?

Maintenant que vous comprenez :
- Les différents cas d'usage PostgreSQL
- Pourquoi la configuration doit s'adapter
- Les compromis inhérents à chaque choix
- Comment utiliser cette annexe

**→ Passez à la section correspondant à votre cas d'usage :**

1. **OLTP** si vous gérez des transactions rapides et nombreuses  
2. **OLAP** si vous faites de l'analyse et du reporting  
3. **Mixed** si vous combinez les deux  
4. **Dev Local** si vous développez sur votre machine

Chaque section fournit une configuration complète, expliquée et prête à l'emploi.

**Bon paramétrage PostgreSQL ! 🐘⚙️**

---


⏭️ [OLTP (High concurrency, low latency)](/annexes/configuration-reference/01-configuration-oltp.md)
