🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.10. Maintenance vitale : Introduction

## Qu'est-ce que la maintenance d'une base de données ?

Imaginez votre base de données PostgreSQL comme une **grande bibliothèque** :
- Chaque jour, de nouveaux livres arrivent (insertions)
- Des livres sont modifiés (mises à jour)
- Des livres sont retirés (suppressions)
- Des lecteurs consultent les livres (requêtes)

Avec le temps, sans maintenance :
- Des livres rayés s'accumulent sur les étagères (lignes mortes)
- Le catalogue devient obsolète (statistiques périmées)
- Les étagères se désorganisent (fragmentation)
- Trouver un livre prend de plus en plus de temps (performances dégradées)

La **maintenance** est l'ensemble des opérations qui **nettoient**, **organisent** et **optimisent** votre base de données pour maintenir des performances optimales et garantir sa santé à long terme.

---

## Pourquoi la maintenance est-elle vitale ?

### 1. PostgreSQL utilise MVCC (Multiversion Concurrency Control)

PostgreSQL a une particularité architecturale fondamentale : le **MVCC**.

**Principe de base** :
Quand vous **modifiez ou supprimez** une ligne, PostgreSQL ne la supprime pas immédiatement. Au lieu de cela :

1. **L'ancienne version** est marquée comme "morte" (obsolète)  
2. **Une nouvelle version** est créée (pour les UPDATE)  
3. Les deux versions coexistent temporairement

**Exemple concret** :

```sql
-- État initial
SELECT * FROM users WHERE id = 1;
-- id | nom    | email
-- 1  | Alice  | alice@exemple.com

-- Vous faites une mise à jour
UPDATE users SET email = 'alice.nouveau@exemple.com' WHERE id = 1;

-- En réalité, PostgreSQL fait ceci :
-- Version 1 (morte, invisible) : 1 | Alice | alice@exemple.com
-- Version 2 (vivante, visible) : 1 | Alice | alice.nouveau@exemple.com
```

**Conséquence critique** : Les anciennes versions s'accumulent et **occupent de l'espace disque** !

### 2. Sans maintenance, les problèmes s'accumulent

#### Problème A : Bloat (Gonflement des tables)

**Définition** : Le "bloat" est l'espace disque gaspillé par les lignes mortes.

**Évolution typique sans maintenance** :

```
Semaine 1 : Table de 100 MB, 5% de bloat     → 105 MB total  
Semaine 2 : Table de 100 MB, 15% de bloat    → 115 MB total  
Semaine 3 : Table de 100 MB, 30% de bloat    → 130 MB total  
Semaine 4 : Table de 100 MB, 50% de bloat    → 150 MB total  
```

**Impact** :
- ❌ **Espace disque gaspillé** : La table occupe 150 MB au lieu de 100 MB  
- ❌ **Requêtes plus lentes** : PostgreSQL doit parcourir 150 MB au lieu de 100 MB  
- ❌ **Sauvegardes plus longues** : Vous sauvegardez 50% de données inutiles  
- ❌ **Cache moins efficace** : La RAM est remplie de données obsolètes

#### Problème B : Statistiques obsolètes

PostgreSQL utilise des **statistiques** pour optimiser vos requêtes :
- Nombre de lignes dans une table
- Distribution des valeurs (quelles valeurs sont fréquentes ?)
- Corrélation entre colonnes

**Sans mise à jour régulière** :

```sql
-- PostgreSQL pense (statistiques obsolètes) :
-- Table "commandes" : 10 000 lignes, 50% à Paris

-- Réalité actuelle :
-- Table "commandes" : 1 000 000 lignes, 5% à Paris

-- Résultat : PostgreSQL fait de MAUVAIS choix
-- Il choisit un plan optimisé pour 10 000 lignes alors qu'il y en a 1 000 000 !
```

**Impact** :
- ❌ **Plans de requêtes sous-optimaux** : PostgreSQL choisit la mauvaise stratégie  
- ❌ **Performances dégradées** : Requêtes 10× à 100× plus lentes  
- ❌ **Ressources gaspillées** : CPU et I/O utilisés inefficacement

#### Problème C : XID Wraparound (Catastrophe potentielle)

PostgreSQL utilise des **Transaction IDs (XID)** pour gérer les versions :
- Chaque transaction reçoit un numéro unique
- Compteur limité : 4 milliards de transactions (2^32)
- Après 4 milliards, le compteur fait le tour (wraparound)

**Sans maintenance préventive** :
- PostgreSQL ne peut plus déterminer quelles données sont anciennes ou récentes
- Pour éviter la corruption, **PostgreSQL s'arrête automatiquement** !

```
WARNING: database "mydb" must be vacuumed within 1000000 transactions  
ERROR: database is not accepting commands to avoid wraparound data loss  
```

**Impact** :
- ❌ **Arrêt complet de la base** : Votre application devient indisponible  
- ❌ **Perte de données potentielle** : Si non traité  
- ❌ **Opération de récupération longue** : Plusieurs heures d'indisponibilité

### 3. La maintenance préventive vs corrective

**Maintenance préventive** (ce que nous voulons) :
```
Maintenance légère quotidienne → Base de données saine → Performances optimales
```
- ✅ Opérations courtes (secondes à minutes)  
- ✅ Peu d'impact sur les utilisateurs  
- ✅ Automatisable  
- ✅ Prévisible

**Maintenance corrective** (ce que nous voulons éviter) :
```
Pas de maintenance → Problèmes s'accumulent → CRISE → Maintenance d'urgence
```
- ❌ Opérations longues (heures)  
- ❌ Impact majeur sur les utilisateurs (downtime)  
- ❌ Stress et urgence  
- ❌ Risques de complications

**Analogie** : Comme pour une voiture
- **Préventif** : Vidange tous les 15 000 km → moteur sain, pas de panne  
- **Correctif** : Jamais de vidange → moteur grippe → réparation coûteuse et longue

---

## Les trois piliers de la maintenance PostgreSQL

### Pilier 1 : VACUUM (Nettoyage physique)

**Objectif** : Nettoyer les lignes mortes et récupérer l'espace disque.

**Ce qu'il fait** :
- Identifie les lignes obsolètes (mortes)
- Marque l'espace comme réutilisable
- Prévient le XID wraparound (gel des anciennes transactions)
- Met à jour les structures internes (FSM, Visibility Map)

**Fréquence** : Automatique via autovacuum (ou manuel si nécessaire)

**Importance** : ⭐⭐⭐⭐⭐ (Critique)

**Nous verrons en détail** : 16.10.1 - VACUUM

### Pilier 2 : ANALYZE (Collecte de statistiques)

**Objectif** : Mettre à jour les statistiques pour optimiser les requêtes.

**Ce qu'il fait** :
- Échantillonne les données de vos tables
- Collecte des statistiques (nombre de lignes, distribution des valeurs)
- Permet au planificateur de faire des choix intelligents
- Met à jour les méta-données système (pg_statistics)

**Fréquence** : Automatique via auto-analyze (ou manuel après gros changements)

**Importance** : ⭐⭐⭐⭐⭐ (Critique)

**Nous verrons en détail** : 16.10.2 - ANALYZE

### Pilier 3 : Autovacuum (Automatisation)

**Objectif** : Automatiser VACUUM et ANALYZE pour vous.

**Ce qu'il fait** :
- Surveille l'activité de vos tables en arrière-plan
- Lance automatiquement VACUUM quand nécessaire
- Lance automatiquement ANALYZE quand nécessaire
- S'adapte à la charge de travail

**Fréquence** : En permanence (processus d'arrière-plan)

**Importance** : ⭐⭐⭐⭐⭐ (Critique - À TOUJOURS laisser activé)

**Nous verrons en détail** : 16.10.3 et 16.10.4 - Améliorations PostgreSQL 18

---

## Vue d'ensemble des opérations de maintenance

### Tableau récapitulatif

| Opération | Objectif | Fréquence typique | Impact performance | Durée | Bloquant ? |
|-----------|----------|-------------------|-------------------|-------|-----------|
| **VACUUM** | Nettoyer les lignes mortes | Quotidien (auto) | Faible | Minutes | Non |
| **VACUUM FULL** | Réorganisation complète | Exceptionnel | Fort | Heures | Oui ⚠️ |
| **ANALYZE** | Mettre à jour statistiques | Quotidien (auto) | Très faible | Secondes | Non |
| **REINDEX** | Reconstruire les index | Mensuel/Annuel | Moyen | Minutes à heures | Concurrent possible |
| **CLUSTER** | Réorganiser physiquement | Rare | Fort | Heures | Oui ⚠️ |

### Le workflow de maintenance idéal

```
┌─────────────────────────────────────────────┐
│  Autovacuum (Automatique, 24/7)             │
│  ├─ Surveille les tables en permanence      │
│  ├─ Lance VACUUM quand nécessaire           │
│  └─ Lance ANALYZE quand nécessaire          │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Vous (Intervention manuelle rare)          │
│  ├─ VACUUM manuel après gros batch          │
│  ├─ ANALYZE avant requête critique          │
│  └─ VACUUM FULL en maintenance planifiée    │
└─────────────────────────────────────────────┘
```

**Philosophie** :
- 95% du temps : **Autovacuum fait tout le travail**
- 5% du temps : **Interventions manuelles ciblées**

---

## Autovacuum : Le gardien automatique

### Qu'est-ce qu'autovacuum ?

Autovacuum est un **système de processus d'arrière-plan** qui :

1. **Se réveille périodiquement** (toutes les 1 minute par défaut, 10s dans PG 18)  
2. **Inspecte vos tables** pour détecter l'activité (insertions, mises à jour, suppressions)  
3. **Décide automatiquement** quelles tables nécessitent un VACUUM ou ANALYZE  
4. **Lance les opérations** de maintenance sans intervention humaine

### Architecture d'autovacuum

```
┌──────────────────────────────────────────┐
│  Autovacuum Launcher (Lanceur)           │
│  ├─ Processus principal                  │
│  ├─ Se réveille toutes les N secondes    │
│  └─ Décide quelles tables traiter        │
└──────────────────────────────────────────┘
            ↓ Démarre ↓
┌──────────────────────────────────────────┐
│  Autovacuum Workers (Travailleurs)       │
│  ├─ Worker 1 → Table A (VACUUM)          │
│  ├─ Worker 2 → Table B (ANALYZE)         │
│  ├─ Worker 3 → Table C (VACUUM)          │
│  └─ ... (jusqu'à N workers)              │
└──────────────────────────────────────────┘
```

### Comment autovacuum décide-t-il ?

**Pour VACUUM** :
```
Seuil = autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × nombre_de_lignes)
```

Si `nombre_de_lignes_mortes > Seuil` → Autovacuum lance VACUUM

**Pour ANALYZE** :
```
Seuil = autovacuum_analyze_threshold + (autovacuum_analyze_scale_factor × nombre_de_lignes)
```

Si `nombre_de_modifications > Seuil` → Autovacuum lance ANALYZE

**Exemple** :
```sql
-- Paramètres par défaut
autovacuum_vacuum_threshold = 50  
autovacuum_vacuum_scale_factor = 0.2  -- 20%  

-- Table avec 100 000 lignes
Seuil = 50 + (0.2 × 100 000) = 20 050

-- Si vous avez 25 000 lignes mortes → Autovacuum VACUUM
-- Si vous avez 15 000 lignes mortes → Autovacuum attend
```

### Paramètres clés d'autovacuum

```sql
-- Vérifier la configuration actuelle
SHOW autovacuum;                           -- on/off (doit être 'on' !)  
SHOW autovacuum_naptime;                   -- Fréquence de réveil (1min défaut)  
SHOW autovacuum_max_workers;               -- Nombre de workers (3 défaut)  
SHOW autovacuum_vacuum_threshold;          -- 50  
SHOW autovacuum_vacuum_scale_factor;       -- 0.2 (20%)  
SHOW autovacuum_analyze_threshold;         -- 50  
SHOW autovacuum_analyze_scale_factor;      -- 0.1 (10%)  
```

### Nouveautés PostgreSQL 18

PostgreSQL 18 apporte des **améliorations majeures** à autovacuum :

1. **Allocation dynamique de workers** (`autovacuum_worker_slots`)
   - Les workers sont créés à la demande
   - Pas de gaspillage de ressources
   - Meilleure adaptation aux pics d'activité

2. **Plafond pour les grandes tables** (`autovacuum_vacuum_max_threshold`)
   - Évite l'accumulation excessive de lignes mortes
   - Protection contre le bloat incontrôlé
   - Vacuum plus fréquents sur les grosses tables

3. **Statistiques enrichies**
   - Nouvelles colonnes dans `pg_stat_all_tables`
   - Meilleure visibilité sur l'activité de maintenance
   - Monitoring plus fin

**Nous détaillerons tout cela dans les sections 16.10.3 et 16.10.4**

---

## Vérifier l'état de santé de votre base

Avant de plonger dans les détails de chaque opération, voici comment **évaluer rapidement** l'état de santé de votre base de données.

### 1. Autovacuum est-il activé ?

```sql
SHOW autovacuum;
```

**Résultat attendu** : `on`

⚠️ **Si 'off'** : Votre base est en danger ! Activez-le immédiatement :
```sql
ALTER SYSTEM SET autovacuum = on;  
SELECT pg_reload_conf();  
```

### 2. Y a-t-il des tables avec beaucoup de bloat ?

```sql
-- Identifier les tables avec beaucoup de lignes mortes
SELECT
    schemaname || '.' || tablename AS table_name,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE n_dead_tup > 1000  
ORDER BY n_dead_tup DESC  
LIMIT 20;  
```

**Indicateurs** :
- `dead_pct < 10%` → ✅ Excellent  
- `dead_pct 10-20%` → ✅ Correct  
- `dead_pct 20-30%` → ⚠️ Attention  
- `dead_pct > 30%` → 🔴 Problème, action requise

### 3. Les statistiques sont-elles à jour ?

```sql
-- Tables avec beaucoup de modifications non analysées
SELECT
    schemaname || '.' || tablename AS table_name,
    n_mod_since_analyze AS modifications,
    n_live_tup AS live_tuples,
    ROUND(n_mod_since_analyze * 100.0 / NULLIF(n_live_tup, 0), 2) AS mod_pct,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables  
WHERE n_mod_since_analyze > 1000  
ORDER BY n_mod_since_analyze DESC  
LIMIT 20;  
```

**Indicateurs** :
- `mod_pct < 10%` → ✅ Statistiques fraîches  
- `mod_pct 10-20%` → ⚠️ Devrait être analysé bientôt  
- `mod_pct > 20%` → 🔴 Statistiques obsolètes, ANALYZE recommandé

### 4. Risque de XID wraparound ?

```sql
-- Vérifier l'âge des transactions
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    CASE
        WHEN age(datfrozenxid) < 100000000 THEN '✅ Sain'
        WHEN age(datfrozenxid) < 150000000 THEN '⚠️ Surveiller'
        WHEN age(datfrozenxid) < 180000000 THEN '🟡 Attention'
        ELSE '🔴 URGENT'
    END AS status
FROM pg_database  
ORDER BY age(datfrozenxid) DESC;  
```

**Seuils critiques** :
- `< 100M` → ✅ Aucun risque  
- `100-150M` → ⚠️ Surveiller  
- `150-180M` → 🟡 Planifier un vacuum  
- `> 180M` → 🔴 Action urgente requise

### 5. Autovacuum tourne-t-il régulièrement ?

```sql
-- Vérifier l'activité autovacuum
SELECT
    schemaname || '.' || tablename AS table_name,
    last_vacuum,
    last_autovacuum,
    autovacuum_count,
    CASE
        WHEN last_autovacuum IS NULL THEN '🔴 Jamais exécuté'
        WHEN last_autovacuum < NOW() - INTERVAL '7 days' THEN '⚠️ Ancien'
        WHEN last_autovacuum < NOW() - INTERVAL '1 day' THEN '✅ Récent'
        ELSE '✅ Très récent'
    END AS status
FROM pg_stat_user_tables  
WHERE schemaname = 'public'  
ORDER BY last_autovacuum DESC NULLS LAST  
LIMIT 20;  
```

---

## Maintenance en environnement de production

### Les bonnes pratiques

#### 1. Faire confiance à autovacuum (mais surveiller)

**Règle d'or** :
```
Autovacuum = 95% du travail  
Interventions manuelles = 5% (cas spécifiques)  
```

❌ **À NE PAS FAIRE** :
- Désactiver autovacuum "pour améliorer les performances"
- Lancer VACUUM FULL tous les jours
- Ignorer les alertes de wraparound

✅ **À FAIRE** :
- Laisser autovacuum activé en permanence
- Surveiller son activité avec des dashboards
- Intervenir manuellement seulement si nécessaire

#### 2. Surveiller activement

**Métriques essentielles à monitorer** :

```
┌────────────────────────────────────────┐
│  Dashboard de Maintenance PostgreSQL   │
├────────────────────────────────────────┤
│  ✓ Bloat par table (%)                 │
│  ✓ Âge XID par database                │
│  ✓ Tables en attente d'autovacuum      │
│  ✓ Durée moyenne des autovacuum        │
│  ✓ Fréquence des autovacuum            │
│  ✓ Tables avec statistiques obsolètes  │
└────────────────────────────────────────┘
```

**Outils recommandés** :
- Prometheus + postgres_exporter + Grafana
- pgBadger (analyse de logs)
- pg_stat_statements (analyse des requêtes)
- Solutions cloud natives (CloudWatch, Azure Monitor, GCP)

#### 3. Planifier les opérations lourdes

Certaines opérations sont **bloquantes** et nécessitent une planification :

**VACUUM FULL** :
```sql
-- ⚠️ Bloquant ! À faire en maintenance programmée
VACUUM FULL ma_table;
```
- Verrou exclusif (aucune lecture/écriture possible)
- Durée : plusieurs heures pour les grosses tables
- Planifier en dehors des heures de pointe

**REINDEX** :
```sql
-- ✅ Option non-bloquante disponible (PG 12+)
REINDEX INDEX CONCURRENTLY mon_index;
```
- Version classique : bloquante
- Version CONCURRENTLY : non-bloquante mais plus lente

#### 4. Ajuster la configuration selon la charge

**Périodes de faible activité** (nuit, week-end) :
```sql
-- Autovacuum plus agressif
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 0;  -- Pas de throttling  
SELECT pg_reload_conf();  
```

**Périodes de forte activité** (heures de pointe) :
```sql
-- Autovacuum plus "gentil"
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 10;  -- Throttling fort  
SELECT pg_reload_conf();  
```

#### 5. Configuration par table pour les cas spécifiques

```sql
-- Table à forte activité : autovacuum très réactif
ALTER TABLE commandes SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5% au lieu de 20%
    autovacuum_vacuum_threshold = 500
);

-- Table volumineuse mais peu active : moins prioritaire
ALTER TABLE logs_archives SET (
    autovacuum_vacuum_scale_factor = 0.3,   -- 30% (moins réactif)
    autovacuum_enabled = true
);
```

---

## Maintenance préventive : Checklist

### Quotidienne (Automatique via monitoring)

- ✅ Vérifier que autovacuum est activé  
- ✅ Surveiller le nombre de tables avec bloat > 20%  
- ✅ Vérifier qu'il n'y a pas de workers autovacuum bloqués  
- ✅ Consulter les logs pour des erreurs de maintenance

### Hebdomadaire (Revue rapide)

- ✅ Analyser les tables avec le plus de bloat  
- ✅ Vérifier les tables qui n'ont pas été vacuum depuis > 7 jours  
- ✅ Contrôler l'évolution de l'âge XID  
- ✅ Identifier les requêtes lentes qui pourraient bénéficier d'un ANALYZE

### Mensuelle (Audit approfondi)

- ✅ Analyser la fréquence des autovacuum par table  
- ✅ Évaluer si la configuration autovacuum est optimale  
- ✅ Identifier les tables candidates pour un REINDEX  
- ✅ Audit des index inutilisés ou redondants  
- ✅ Vérifier la croissance de la taille de la base

### Trimestrielle (Stratégie)

- ✅ Réévaluer les paramètres d'autovacuum selon l'évolution de la charge  
- ✅ Planifier des VACUUM FULL si nécessaire (fenêtre de maintenance)  
- ✅ Revoir les stratégies de partitionnement si applicable  
- ✅ Audit de performance global (avec EXPLAIN ANALYZE)

---

## Les erreurs courantes à éviter

### ❌ Erreur #1 : Désactiver autovacuum

```sql
-- ❌ JAMAIS FAIRE CECI
ALTER SYSTEM SET autovacuum = off;
```

**Conséquence** : Accumulation de bloat, statistiques obsolètes, risque de wraparound.

**Correctif** :
```sql
-- ✅ Toujours laisser activé
ALTER SYSTEM SET autovacuum = on;
```

### ❌ Erreur #2 : VACUUM FULL à répétition

```sql
-- ❌ Mauvaise pratique
-- Tous les jours à 2h du matin
VACUUM FULL; -- Verrou exclusif, très lent !
```

**Conséquence** : Downtime régulier, gaspillage de ressources.

**Correctif** :
```sql
-- ✅ Laisser VACUUM normal faire le travail
-- VACUUM FULL seulement en cas extrême et planifié
```

### ❌ Erreur #3 : Ignorer les alertes de wraparound

```
-- Log PostgreSQL
WARNING: database "production" must be vacuumed within 10000000 transactions
```

**Conséquence** : Arrêt automatique de PostgreSQL pour éviter la corruption !

**Correctif** :
```sql
-- ✅ Vacuum immédiat sur toute la base
VACUUMDB --all --freeze
```

### ❌ Erreur #4 : Ne pas surveiller le bloat

"Ça marche, pourquoi surveiller ?"

**Conséquence** : Dégradation progressive des performances, explosion de la taille disque.

**Correctif** : Mettre en place un monitoring actif.

### ❌ Erreur #5 : Configuration inadaptée

```sql
-- ❌ Configuration par défaut sur une base volumineuse
autovacuum_max_workers = 3  -- Trop peu pour 100+ tables actives
```

**Conséquence** : Autovacuum ne suit pas, bloat s'accumule.

**Correctif** :
```sql
-- ✅ Adapter à votre charge (PostgreSQL 18)
autovacuum_worker_slots = 12;  
autovacuum_vacuum_max_threshold = 2000000;  
```

---

## Résumé : Les points clés

### 1. Pourquoi la maintenance est vitale

- 🧹 **MVCC crée des lignes mortes** qui s'accumulent sans nettoyage  
- 📊 **Les statistiques obsolètes** dégradent les performances des requêtes  
- ⚠️ **Le XID wraparound** peut arrêter votre base de données

### 2. Les trois piliers

| Pilier | Rôle | Automatisé ? |
|--------|------|--------------|
| **VACUUM** | Nettoyer les lignes mortes | ✅ Oui (autovacuum) |
| **ANALYZE** | Mettre à jour les statistiques | ✅ Oui (auto-analyze) |
| **Autovacuum** | Orchestrer VACUUM et ANALYZE | ✅ Processus d'arrière-plan |

### 3. Autovacuum : Votre meilleur ami

- ✅ **Toujours activé** (c'est non négociable)  
- 🤖 **Automatique** : fait 95% du travail pour vous  
- 🎯 **Intelligent** : s'adapte à l'activité de vos tables  
- 🚀 **PostgreSQL 18** : encore plus performant et flexible

### 4. Votre rôle

- 📊 **Surveiller** : Dashboard de monitoring  
- ⚙️ **Configurer** : Ajuster selon votre charge  
- 🛠️ **Intervenir** : Seulement dans les 5% de cas spécifiques

---

## Prochaines étapes

Maintenant que vous comprenez **pourquoi** la maintenance est vitale et **comment** elle fonctionne globalement, nous allons détailler chaque opération :

1. **16.10.1 - VACUUM** : Comprendre en profondeur le nettoyage des lignes mortes et la prévention du XID wraparound

2. **16.10.2 - ANALYZE** : Maîtriser la collecte de statistiques et l'optimisation du planificateur

3. **16.10.3 - Autovacuum et ajustements dynamiques** : Découvrir les améliorations de PostgreSQL 18 avec l'allocation dynamique de workers

4. **16.10.4 - autovacuum_vacuum_max_threshold** : Protéger vos grandes tables contre l'accumulation excessive de bloat

Chaque section approfondit un aspect spécifique avec des exemples concrets, des configurations recommandées, et des stratégies de monitoring.

**Commençons par le fondement de tout : VACUUM !**

---


⏭️ [VACUUM : Récupération d'espace et prévention XID wraparound](/16-administration-configuration-securite/10.1-vacuum.md)
