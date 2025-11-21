🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Requêtes d'Analyse PostgreSQL
## Statistiques et Tailles des Tables

---

## Table des Matières

1. [Introduction à l'Analyse PostgreSQL](#1-introduction-%C3%A0-lanalyse-postgresql)
2. [Analyse de la Taille des Objets](#2-analyse-de-la-taille-des-objets)
3. [Statistiques des Tables](#3-statistiques-des-tables)
4. [Statistiques des Colonnes](#4-statistiques-des-colonnes)
5. [Analyse de la Distribution des Données](#5-analyse-de-la-distribution-des-donn%C3%A9es)
6. [Analyse de la Fragmentation](#6-analyse-de-la-fragmentation)
7. [Statistiques du Planificateur](#7-statistiques-du-planificateur)
8. [Mise en Pratique : Scénarios d'Analyse](#8-mise-en-pratique--sc%C3%A9narios-danalyse)
9. [Maintenance des Statistiques](#9-maintenance-des-statistiques)
10. [Bonnes Pratiques](#10-bonnes-pratiques)

---

## 1. Introduction à l'Analyse PostgreSQL

### 1.1. Pourquoi Analyser Votre Base de Données ?

L'analyse régulière de votre base PostgreSQL est essentielle pour :

1. **Comprendre la croissance des données**
   - Prévoir les besoins en stockage
   - Planifier l'archivage des données anciennes
   - Anticiper les limitations matérielles

2. **Optimiser les performances**
   - Identifier les tables et index surdimensionnés
   - Détecter les tables sous-indexées ou sur-indexées
   - Comprendre la distribution des données pour l'optimisation

3. **Planifier la capacité**
   - Estimer les besoins futurs en stockage
   - Décider quand partitionner une table
   - Optimiser les coûts cloud (stockage)

4. **Maintenir la santé de la base**
   - Détecter la fragmentation
   - Identifier les problèmes de statistiques obsolètes
   - Valider l'efficacité des index

### 1.2. Les Deux Grandes Familles d'Analyse

#### Analyse de Taille (Size Analysis)

Mesure l'**espace disque** occupé par vos objets PostgreSQL :

- Tables (données brutes)
- Index (structures d'accélération)
- TOAST (données volumineuses externalisées)
- Bases de données complètes

**Unités courantes** :
- **Bytes** : Unité de base
- **KB** (Kilobytes) : 1024 bytes
- **MB** (Megabytes) : 1024 KB
- **GB** (Gigabytes) : 1024 MB
- **TB** (Terabytes) : 1024 GB

#### Analyse Statistique (Statistical Analysis)

Examine les **caractéristiques des données** :

- Nombre de lignes (tuples)
- Distribution des valeurs
- Cardinalité (nombre de valeurs uniques)
- NULL prevalence
- Fréquence d'accès

### 1.3. Les Vues Système PostgreSQL

PostgreSQL expose ces informations via des **vues système** :

| Vue | Contenu |
|-----|---------|
| `pg_class` | Catalogue des tables, index, vues |
| `pg_database` | Liste des bases de données |
| `pg_tables` | Tables utilisateur (vue simplifiée) |
| `pg_indexes` | Index utilisateur |
| `pg_stats` | Statistiques par colonne |
| `pg_stat_user_tables` | Statistiques d'accès par table |
| `pg_statio_user_tables` | Statistiques I/O par table |

### 1.4. Fonctions de Taille PostgreSQL

PostgreSQL fournit des fonctions natives pour mesurer la taille :

```sql
-- Taille d'une table (sans index ni TOAST)
SELECT pg_relation_size('ma_table');

-- Taille d'une table + ses index + TOAST
SELECT pg_total_relation_size('ma_table');

-- Taille de tous les index d'une table
SELECT pg_indexes_size('ma_table');

-- Taille d'une base de données complète
SELECT pg_database_size('ma_base');

-- Taille du tablespace
SELECT pg_tablespace_size('pg_default');
```

**Note** : Ces fonctions retournent des valeurs en **bytes**. Utilisez `pg_size_pretty()` pour un affichage lisible.

---

## 2. Analyse de la Taille des Objets

### 2.1. Taille de Toutes les Bases de Données

Pour avoir une vue d'ensemble de l'espace occupé par chaque base :

```sql
SELECT
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size,
    pg_database_size(datname) AS size_bytes
FROM
    pg_database
WHERE
    datname NOT IN ('template0', 'template1')  -- Exclure les templates
ORDER BY
    pg_database_size(datname) DESC;
```

**Interprétation** :

- Identifiez votre base la plus volumineuse
- Surveillez la croissance mensuelle
- Planifiez l'archivage si nécessaire

**Exemple de résultat** :

```
database_name | size    | size_bytes
--------------+---------+-------------
production    | 450 GB  | 483183820800
staging       | 125 GB  | 134217728000
development   | 15 GB   | 16106127360
```

### 2.2. Taille des Tables (Top 20)

Pour identifier vos tables les plus volumineuses :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS indexes_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename) -
                   pg_indexes_size(schemaname||'.'||tablename)) AS toast_size,
    pg_total_relation_size(schemaname||'.'||tablename) AS total_bytes
FROM
    pg_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

**Colonnes importantes** :

- **total_size** : Taille complète (table + index + TOAST)
- **table_size** : Données de la table uniquement
- **indexes_size** : Tous les index combinés
- **toast_size** : Données TOAST (colonnes larges externalisées)

**Analyse** :

- Si `indexes_size > table_size` : Vous avez probablement trop d'index
- Si `toast_size` est élevé : Table avec beaucoup de colonnes TEXT/BYTEA volumineuses

**Exemple de résultat** :

```
tablename | total_size | table_size | indexes_size | toast_size
----------+------------+------------+--------------+-----------
orders    | 45 GB      | 25 GB      | 18 GB        | 2 GB
logs      | 120 GB     | 115 GB     | 5 GB         | 0 bytes
users     | 8 GB       | 3 GB       | 4 GB         | 1 GB
```

### 2.3. Répartition Taille Table vs Index

Pour comprendre le ratio table/index :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS indexes_size,
    round(
        100.0 * pg_indexes_size(schemaname||'.'||tablename) /
        NULLIF(pg_relation_size(schemaname||'.'||tablename), 0),
        2
    ) AS index_ratio_pct,
    (SELECT count(*)
     FROM pg_indexes
     WHERE schemaname = pt.schemaname
     AND tablename = pt.tablename) AS index_count
FROM
    pg_tables pt
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
    AND pg_relation_size(schemaname||'.'||tablename) > 0
ORDER BY
    index_ratio_pct DESC
LIMIT 20;
```

**Analyse** :

- **index_ratio_pct > 100%** : Les index occupent plus de place que les données
  - Peut être normal pour des petites tables avec beaucoup d'index
  - Pour grosses tables : signe de sur-indexation
- **index_ratio_pct < 10%** avec `index_count` faible : Peut-être sous-indexée

**Exemple de résultat** :

```
tablename | table_size | indexes_size | index_ratio_pct | index_count
----------+------------+--------------+-----------------+------------
products  | 500 MB     | 2 GB         | 400.00          | 12
orders    | 25 GB      | 18 GB        | 72.00           | 8
logs      | 115 GB     | 5 GB         | 4.35            | 2
```

→ La table `products` a un ratio de **400%** : Révisez vos index !

### 2.4. Taille de Tous les Index d'une Table

Pour examiner en détail les index d'une table spécifique :

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_relation_size(indexrelid) AS index_bytes,
    idx_scan AS number_of_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM
    pg_stat_user_indexes
WHERE
    tablename = 'orders'  -- Remplacer par votre table
ORDER BY
    pg_relation_size(indexrelid) DESC;
```

**Analyse** :

- **Gros index avec idx_scan = 0** : Index inutilisé, envisagez la suppression
- **Petit index avec idx_scan élevé** : Index très utile, à conserver
- **Plusieurs gros index** : Vérifiez s'il n'y a pas de redondance

### 2.5. Croissance des Tables dans le Temps

Pour suivre la croissance, créez une table de tracking :

```sql
-- Table pour stocker l'historique de taille (à créer une fois)
CREATE TABLE IF NOT EXISTS monitoring.table_size_history (
    measured_at TIMESTAMP DEFAULT now(),
    schema_name TEXT,
    table_name TEXT,
    total_size_bytes BIGINT,
    table_size_bytes BIGINT,
    indexes_size_bytes BIGINT
);

-- Requête à exécuter périodiquement (quotidien/hebdomadaire)
INSERT INTO monitoring.table_size_history (
    schema_name,
    table_name,
    total_size_bytes,
    table_size_bytes,
    indexes_size_bytes
)
SELECT
    schemaname,
    tablename,
    pg_total_relation_size(schemaname||'.'||tablename),
    pg_relation_size(schemaname||'.'||tablename),
    pg_indexes_size(schemaname||'.'||tablename)
FROM
    pg_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema');

-- Analyser la croissance sur 30 jours
SELECT
    table_name,
    pg_size_pretty(MAX(total_size_bytes)) AS current_size,
    pg_size_pretty(MIN(total_size_bytes)) AS size_30_days_ago,
    pg_size_pretty(MAX(total_size_bytes) - MIN(total_size_bytes)) AS growth,
    round(
        100.0 * (MAX(total_size_bytes) - MIN(total_size_bytes)) /
        NULLIF(MIN(total_size_bytes), 0),
        2
    ) AS growth_pct
FROM
    monitoring.table_size_history
WHERE
    measured_at > now() - interval '30 days'
GROUP BY
    table_name
ORDER BY
    MAX(total_size_bytes) - MIN(total_size_bytes) DESC
LIMIT 20;
```

**Utilité** :

- Prédire quand vous atteindrez les limites de stockage
- Identifier les tables à croissance anormale
- Valider l'efficacité des stratégies d'archivage

### 2.6. Taille par Schéma

Si vous utilisez plusieurs schémas :

```sql
SELECT
    schemaname,
    count(*) AS table_count,
    pg_size_pretty(sum(pg_total_relation_size(schemaname||'.'||tablename))) AS total_size,
    sum(pg_total_relation_size(schemaname||'.'||tablename)) AS total_bytes
FROM
    pg_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY
    schemaname
ORDER BY
    sum(pg_total_relation_size(schemaname||'.'||tablename)) DESC;
```

**Exemple de résultat** :

```
schemaname | table_count | total_size
-----------+-------------+-----------
public     | 45          | 350 GB
analytics  | 12          | 120 GB
staging    | 30          | 25 GB
```

### 2.7. Espace Disque Disponible

Pour voir l'espace disque total disponible sur le serveur :

```sql
-- PostgreSQL 18 : Nouvelles fonctions de monitoring disque
SELECT
    pg_size_pretty(sum(pg_database_size(datname))) AS total_db_size,
    pg_size_pretty(
        (SELECT setting::bigint * current_setting('block_size')::bigint
         FROM pg_settings
         WHERE name = 'shared_buffers')
    ) AS shared_buffers_size
FROM
    pg_database;
```

**Note** : Pour l'espace disque total du système, utilisez une commande shell :

```bash
# Linux
df -h /var/lib/postgresql

# Résultat exemple :
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       500G  350G  150G  70% /var/lib/postgresql
```

---

## 3. Statistiques des Tables

### 3.1. Vue d'Ensemble des Statistiques

PostgreSQL collecte automatiquement des statistiques sur l'utilisation des tables :

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    seq_scan AS sequential_scans,
    seq_tup_read AS seq_rows_read,
    idx_scan AS index_scans,
    idx_tup_fetch AS idx_rows_fetched,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM
    pg_stat_user_tables
ORDER BY
    n_live_tup DESC
LIMIT 20;
```

**Colonnes clés** :

- **n_live_tup** : Nombre de lignes actives (estimation)
- **n_dead_tup** : Nombre de lignes mortes (bloat potentiel)
- **seq_scan** : Nombre de scans séquentiels (peut indiquer besoin d'index)
- **idx_scan** : Nombre de scans d'index (bonne utilisation)
- **n_tup_ins/upd/del** : Activité d'écriture

**Analyse** :

- **dead_pct > 20%** : Besoin de VACUUM
- **seq_scan élevé sur grosse table** : Manque d'index
- **last_autovacuum NULL** : Table jamais nettoyée (problème)

### 3.2. Tables les Plus Actives (Lectures)

Identifier les tables les plus consultées :

```sql
SELECT
    schemaname,
    relname AS table_name,
    seq_scan + idx_scan AS total_scans,
    seq_scan AS sequential_scans,
    idx_scan AS index_scans,
    round(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS index_usage_pct,
    seq_tup_read AS rows_read_seq,
    idx_tup_fetch AS rows_fetched_idx,
    n_live_tup AS estimated_rows,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM
    pg_stat_user_tables
WHERE
    seq_scan + idx_scan > 0
ORDER BY
    seq_scan + idx_scan DESC
LIMIT 20;
```

**Analyse** :

- **index_usage_pct < 50%** sur grosse table : Créez des index
- **seq_tup_read >> n_live_tup** : Scans séquentiels multiples, optimisez
- **Tables très actives** : Candidates au caching applicatif (Redis, Memcached)

### 3.3. Tables les Plus Actives (Écritures)

Identifier les tables avec beaucoup d'écritures :

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_tup_ins + n_tup_upd + n_tup_del AS total_writes,
    n_live_tup AS live_rows,
    round(
        (n_tup_ins + n_tup_upd + n_tup_del)::numeric /
        NULLIF(n_live_tup, 0),
        2
    ) AS write_ratio,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM
    pg_stat_user_tables
WHERE
    n_tup_ins + n_tup_upd + n_tup_del > 0
ORDER BY
    n_tup_ins + n_tup_upd + n_tup_del DESC
LIMIT 20;
```

**Analyse** :

- **write_ratio élevé** : Beaucoup d'écritures par rapport au nombre de lignes
  - Table de logs : Normal
  - Table de données métier : Peut indiquer un problème de conception
- **Grosses tables avec beaucoup d'UPDATE** : Génèrent du bloat, surveillez VACUUM

### 3.4. Taux d'Insertion vs Suppression

Pour comprendre la dynamique des données :

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_tup_ins AS inserts,
    n_tup_del AS deletes,
    n_live_tup AS current_rows,
    CASE
        WHEN n_tup_del = 0 THEN 'Append-only'
        WHEN n_tup_ins = n_tup_del THEN 'Stable'
        WHEN n_tup_ins > n_tup_del THEN 'Growing'
        ELSE 'Shrinking'
    END AS growth_pattern,
    round(
        100.0 * (n_tup_ins - n_tup_del) / NULLIF(n_tup_ins, 0),
        2
    ) AS net_growth_pct
FROM
    pg_stat_user_tables
WHERE
    n_tup_ins > 0
ORDER BY
    n_tup_ins DESC
LIMIT 20;
```

**Analyse** :

- **Append-only** (logs, events) : Pas de suppression, archivage nécessaire
- **Stable** : INSERT = DELETE, table de taille constante (queue, cache)
- **Growing** : Croissance continue, planifiez le partitionnement
- **Shrinking** : Rare, peut indiquer un nettoyage ou une migration

### 3.5. Statistiques d'I/O par Table

Voir combien de fois les données sont lues depuis le cache vs disque :

```sql
SELECT
    schemaname,
    relname AS table_name,
    heap_blks_read AS disk_reads,
    heap_blks_hit AS cache_hits,
    heap_blks_read + heap_blks_hit AS total_reads,
    round(
        100.0 * heap_blks_hit / NULLIF(heap_blks_read + heap_blks_hit, 0),
        2
    ) AS cache_hit_ratio_pct,
    idx_blks_read AS idx_disk_reads,
    idx_blks_hit AS idx_cache_hits,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM
    pg_statio_user_tables
WHERE
    heap_blks_read + heap_blks_hit > 0
ORDER BY
    heap_blks_read DESC
LIMIT 20;
```

**Analyse** :

- **cache_hit_ratio_pct < 90%** : Table peu cachée, considérez :
  - Augmenter `shared_buffers`
  - Partitionner si très grosse
  - Archiver données anciennes
- **disk_reads élevé** : Consomme beaucoup d'I/O

### 3.6. Nouveauté PostgreSQL 18 : Statistiques VACUUM Enrichies

PostgreSQL 18 ajoute des compteurs détaillés pour VACUUM et ANALYZE :

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_vacuum,
    last_autovacuum,
    vacuum_count,           -- Nouveau dans PG 18
    autovacuum_count,       -- Nouveau dans PG 18
    last_analyze,
    last_autoanalyze,
    analyze_count,          -- Nouveau dans PG 18
    autoanalyze_count       -- Nouveau dans PG 18
FROM
    pg_stat_all_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    n_dead_tup DESC
LIMIT 20;
```

**Nouveaux compteurs** :

- **vacuum_count** : Nombre de VACUUM manuels
- **autovacuum_count** : Nombre d'autovacuum exécutés
- **analyze_count** / **autoanalyze_count** : Idem pour ANALYZE

**Utilité** : Vérifier que VACUUM/ANALYZE s'exécutent régulièrement.

---

## 4. Statistiques des Colonnes

### 4.1. Vue d'Ensemble des Statistiques de Colonnes

PostgreSQL collecte des statistiques détaillées sur chaque colonne pour optimiser les requêtes :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    n_distinct,
    null_frac,
    avg_width,
    correlation
FROM
    pg_stats
WHERE
    schemaname = 'public'
    AND tablename = 'users'  -- Remplacer par votre table
ORDER BY
    attname;
```

**Colonnes importantes** :

- **n_distinct** : Nombre de valeurs uniques estimé
  - Valeur positive : Nombre absolu
  - Valeur négative : Proportion (ex: -0.5 = 50% des lignes sont uniques)
- **null_frac** : Proportion de valeurs NULL (0.0 à 1.0)
- **avg_width** : Taille moyenne de la colonne en bytes
- **correlation** : Corrélation entre ordre physique et ordre logique (-1 à 1)

### 4.2. Cardinalité des Colonnes

La cardinalité (nombre de valeurs distinctes) est cruciale pour l'optimisation :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    CASE
        WHEN n_distinct > 0 THEN n_distinct
        WHEN n_distinct < 0 THEN abs(n_distinct) *
            (SELECT n_live_tup FROM pg_stat_user_tables
             WHERE schemaname = ps.schemaname
             AND relname = ps.tablename)
        ELSE 0
    END AS estimated_distinct_values,
    (SELECT n_live_tup FROM pg_stat_user_tables
     WHERE schemaname = ps.schemaname
     AND relname = ps.tablename) AS total_rows,
    round(null_frac * 100, 2) AS null_percentage,
    avg_width AS avg_bytes
FROM
    pg_stats ps
WHERE
    schemaname = 'public'
    AND tablename = 'orders'
ORDER BY
    estimated_distinct_values DESC;
```

**Analyse** :

- **Cardinalité faible** (< 100 valeurs distinctes) :
  - Colonne de type ENUM ou status
  - Bon candidat pour un index partiel ou un index BRIN
- **Cardinalité élevée** (proche du nombre de lignes) :
  - Colonne unique (ID, email)
  - Bon candidat pour un index B-Tree
- **null_percentage élevé** : Considérez un index partiel `WHERE col IS NOT NULL`

**Exemple de résultat** :

```
column_name          | estimated_distinct | total_rows | null_percentage | avg_bytes
---------------------+--------------------+------------+-----------------+----------
order_id             | 5000000            | 5000000    | 0.00            | 8
user_id              | 250000             | 5000000    | 0.00            | 8
status               | 5                  | 5000000    | 0.00            | 10
created_at           | 4500000            | 5000000    | 0.00            | 8
notes                | 3000000            | 5000000    | 45.00           | 120
```

### 4.3. Distribution des Valeurs les Plus Fréquentes

PostgreSQL stocke les valeurs les plus communes (Most Common Values - MCV) :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    most_common_vals,
    most_common_freqs
FROM
    pg_stats
WHERE
    schemaname = 'public'
    AND tablename = 'orders'
    AND most_common_vals IS NOT NULL
ORDER BY
    attname;
```

**Utilisation** :

- **most_common_vals** : Array des valeurs les plus fréquentes
- **most_common_freqs** : Array des fréquences correspondantes (somme = 1.0)

**Exemple** :

```
column_name | most_common_vals              | most_common_freqs
------------+-------------------------------+---------------------------
status      | {pending,shipped,delivered}   | {0.45,0.30,0.20}
```

→ 45% des commandes sont `pending`, 30% `shipped`, 20% `delivered`

**Implication pour l'indexation** :

- Si une valeur représente > 10% des lignes, PostgreSQL peut préférer un scan séquentiel
- Index partiel utile : `CREATE INDEX ... WHERE status != 'pending'`

### 4.4. Corrélation Physique vs Logique

La corrélation indique si les données sont bien ordonnées physiquement :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    correlation,
    CASE
        WHEN abs(correlation) > 0.9 THEN 'Highly correlated'
        WHEN abs(correlation) > 0.5 THEN 'Moderately correlated'
        ELSE 'Low correlation'
    END AS correlation_level
FROM
    pg_stats
WHERE
    schemaname = 'public'
    AND tablename = 'orders'
    AND correlation IS NOT NULL
ORDER BY
    abs(correlation) DESC;
```

**Interprétation** :

- **correlation ≈ 1.0** : Les données sont physiquement ordonnées dans l'ordre croissant
- **correlation ≈ -1.0** : Les données sont physiquement ordonnées dans l'ordre décroissant
- **correlation ≈ 0** : Aucune corrélation, données désordonnées

**Impact** :

- **Haute corrélation** : Scans d'index très efficaces (lecture séquentielle)
- **Basse corrélation** : Scans d'index avec beaucoup de random I/O

**Solution pour améliorer** :

```sql
-- Réorganiser physiquement la table selon un index
CLUSTER orders USING idx_orders_created_at;
```

**Attention** : `CLUSTER` pose un verrou exclusif, utilisez hors production.

### 4.5. Colonnes Volumineuses

Identifier les colonnes qui consomment le plus d'espace :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    avg_width AS avg_bytes,
    pg_size_pretty(
        avg_width *
        (SELECT n_live_tup FROM pg_stat_user_tables
         WHERE schemaname = ps.schemaname
         AND relname = ps.tablename)
    ) AS estimated_column_size,
    (SELECT pg_size_pretty(pg_relation_size(schemaname||'.'||tablename))
     FROM pg_tables WHERE schemaname = ps.schemaname AND tablename = ps.tablename) AS table_size
FROM
    pg_stats ps
WHERE
    schemaname = 'public'
    AND tablename = 'products'
ORDER BY
    avg_width DESC
LIMIT 10;
```

**Analyse** :

- **avg_bytes > 1000** : Colonne volumineuse (TEXT, BYTEA, JSONB)
  - Ces colonnes utilisent TOAST si > 2KB
- **Grosse colonne rarement utilisée** : Envisagez de la déplacer dans une table séparée

### 4.6. Histogrammes de Distribution

Pour les colonnes non-catégorielles, PostgreSQL maintient des histogrammes :

```sql
SELECT
    schemaname,
    tablename,
    attname AS column_name,
    histogram_bounds
FROM
    pg_stats
WHERE
    schemaname = 'public'
    AND tablename = 'orders'
    AND histogram_bounds IS NOT NULL
ORDER BY
    attname;
```

**histogram_bounds** : Array de valeurs délimitant des "buckets" de distribution.

**Utilité** : Le planificateur utilise ces histogrammes pour estimer le nombre de lignes retournées par une clause WHERE.

**Exemple** :

```
column_name | histogram_bounds
------------+------------------
price       | {0.00,9.99,19.99,49.99,99.99,499.99,9999.99}
```

→ Distribution des prix en 7 buckets

---

## 5. Analyse de la Distribution des Données

### 5.1. Distribution des Lignes par Partition (Si Partitionné)

Pour les tables partitionnées, analyser la distribution :

```sql
SELECT
    schemaname,
    tablename,
    n_live_tup AS rows,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    round(
        100.0 * pg_total_relation_size(schemaname||'.'||tablename) /
        sum(pg_total_relation_size(schemaname||'.'||tablename)) OVER (),
        2
    ) AS size_pct
FROM
    pg_stat_user_tables
WHERE
    tablename LIKE 'orders_%'  -- Pattern de vos partitions
ORDER BY
    tablename;
```

**Analyse** :

- **Distribution uniforme** : Bon partitionnement
- **Quelques partitions très grosses** : Revoir la stratégie de partitionnement
- **Beaucoup de petites partitions** : Overhead inutile

### 5.2. Distribution Temporelle des Données

Pour les tables avec colonne timestamp :

```sql
SELECT
    date_trunc('month', created_at) AS month,
    count(*) AS row_count,
    pg_size_pretty(
        count(*) * (SELECT avg_width FROM pg_stats
                    WHERE schemaname = 'public'
                    AND tablename = 'orders'
                    AND attname = 'created_at')
    ) AS estimated_size
FROM
    orders
GROUP BY
    date_trunc('month', created_at)
ORDER BY
    month DESC
LIMIT 24;  -- 2 ans de données
```

**Utilité** :

- Identifier les périodes de forte activité
- Planifier le partitionnement par date
- Décider quelles données archiver

### 5.3. Distribution des Valeurs Catégorielles

Analyser la distribution d'une colonne catégorielle :

```sql
SELECT
    status,
    count(*) AS row_count,
    round(100.0 * count(*) / sum(count(*)) OVER (), 2) AS percentage,
    pg_size_pretty(
        count(*) * (SELECT avg_width FROM pg_stats
                    WHERE schemaname = 'public'
                    AND tablename = 'orders'
                    AND attname = 'status')
    ) AS estimated_size
FROM
    orders
GROUP BY
    status
ORDER BY
    count(*) DESC;
```

**Utilisation pour l'optimisation** :

- Si une valeur représente > 50% : Index partiel sur les autres valeurs
- Si distribution très inégale : Le planificateur peut mal estimer

**Exemple** :

```
status      | row_count | percentage | estimated_size
------------+-----------+------------+---------------
pending     | 2500000   | 50.00      | 23 MB
shipped     | 1500000   | 30.00      | 14 MB
delivered   | 900000    | 18.00      | 8 MB
cancelled   | 100000    | 2.00       | 1 MB
```

### 5.4. Distribution des Tailles de Lignes

Comprendre la variabilité de la taille des lignes :

```sql
WITH row_sizes AS (
    SELECT
        tablename,
        attname,
        avg_width,
        n_live_tup
    FROM
        pg_stats ps
    JOIN
        pg_stat_user_tables pst
        ON ps.schemaname = pst.schemaname
        AND ps.tablename = pst.relname
    WHERE
        ps.schemaname = 'public'
        AND ps.tablename = 'products'
)
SELECT
    tablename,
    sum(avg_width) AS avg_row_width_bytes,
    pg_size_pretty(sum(avg_width)) AS avg_row_width,
    max(n_live_tup) AS total_rows,
    pg_size_pretty(sum(avg_width) * max(n_live_tup)) AS estimated_table_size,
    (SELECT pg_size_pretty(pg_relation_size('public.products'))) AS actual_table_size
FROM
    row_sizes
GROUP BY
    tablename;
```

**Comparaison** :

- Si `estimated_table_size << actual_table_size` : Beaucoup de bloat
- Si `estimated_table_size ≈ actual_table_size` : Bon état

---

## 6. Analyse de la Fragmentation

### 6.1. Qu'est-ce que la Fragmentation ?

La **fragmentation** se produit quand :

1. **Fragmentation logique (bloat)** : Espace mort dû aux UPDATE/DELETE
2. **Fragmentation physique** : Pages partiellement remplies

**Conséquences** :

- Gaspillage d'espace disque
- Scans séquentiels plus lents
- Cache moins efficace

### 6.2. Détecter la Fragmentation (pgstattuple)

Extension `pgstattuple` fournit des mesures précises :

```sql
-- Installer l'extension (une fois)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Analyser une table
SELECT * FROM pgstattuple('orders');
```

**Résultat** :

```
table_len          | 26843545600   -- Taille totale en bytes
tuple_count        | 5000000       -- Nombre de lignes vivantes
tuple_len          | 20000000000   -- Espace occupé par les tuples
tuple_percent      | 74.55         -- % d'espace utile
dead_tuple_count   | 500000        -- Lignes mortes
dead_tuple_len     | 2000000000    -- Espace mort
dead_tuple_percent | 7.46          -- % de bloat
free_space         | 4843545600    -- Espace libre dans les pages
free_percent       | 18.05         -- % d'espace libre
```

**Analyse** :

- **dead_tuple_percent > 20%** : Beaucoup de bloat, VACUUM recommandé
- **free_percent > 20%** : Pages partiellement remplies, VACUUM FULL ou pg_repack

### 6.3. Estimer le Bloat Sans pgstattuple

Estimation rapide sans scan complet :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS bloat_pct,
    CASE
        WHEN n_dead_tup > n_live_tup * 0.2 THEN 'VACUUM recommended'
        WHEN n_dead_tup > n_live_tup * 0.5 THEN 'VACUUM URGENT'
        ELSE 'OK'
    END AS recommendation
FROM
    pg_stat_user_tables
WHERE
    n_live_tup > 0
ORDER BY
    n_dead_tup DESC
LIMIT 20;
```

### 6.4. Fragmentation des Index

Les index aussi souffrent de fragmentation :

```sql
-- Avec pgstattuple
SELECT
    indexrelname,
    *
FROM
    pgstatindex('idx_orders_user_id');
```

**Résultat** :

```
version            | 4
tree_level         | 3              -- Profondeur de l'arbre B-Tree
index_size         | 8960000        -- Taille en bytes
root_block_no      | 412
internal_pages     | 125            -- Pages internes
leaf_pages         | 2000           -- Pages feuilles
empty_pages        | 50             -- Pages vides (fragmentation)
deleted_pages      | 20             -- Pages supprimées
avg_leaf_density   | 67.45          -- Densité moyenne des feuilles (%)
leaf_fragmentation | 15.23          -- % de fragmentation
```

**Analyse** :

- **leaf_fragmentation > 30%** : Index fragmenté, REINDEX recommandé
- **avg_leaf_density < 60%** : Pages peu remplies

**Solution** :

```sql
REINDEX INDEX CONCURRENTLY idx_orders_user_id;
```

### 6.5. Calcul du Bloat Théorique (Formule Avancée)

Estimation du bloat en bytes :

```sql
SELECT
    schemaname,
    tablename,
    pg_relation_size(schemaname||'.'||tablename) AS actual_size_bytes,
    (n_live_tup *
     (SELECT sum(avg_width) FROM pg_stats
      WHERE schemaname = pst.schemaname
      AND tablename = pst.relname)
    ) AS estimated_useful_bytes,
    pg_relation_size(schemaname||'.'||tablename) -
    (n_live_tup *
     (SELECT sum(avg_width) FROM pg_stats
      WHERE schemaname = pst.schemaname
      AND tablename = pst.relname)
    ) AS bloat_bytes,
    pg_size_pretty(
        pg_relation_size(schemaname||'.'||tablename) -
        (n_live_tup *
         (SELECT sum(avg_width) FROM pg_stats
          WHERE schemaname = pst.schemaname
          AND tablename = pst.relname)
        )
    ) AS bloat_size
FROM
    pg_stat_user_tables pst
WHERE
    n_live_tup > 0
ORDER BY
    bloat_bytes DESC
LIMIT 20;
```

**Note** : C'est une estimation approximative, pgstattuple est plus précis.

---

## 7. Statistiques du Planificateur

### 7.1. Qualité des Estimations du Planificateur

Le planificateur utilise les statistiques pour estimer le nombre de lignes :

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';
```

**Comparer** :

- **rows=X** (estimation du planificateur)
- **actual rows=Y** (résultat réel)

**Si |X - Y| / Y > 0.5** : Estimation mauvaise, réexécutez ANALYZE.

### 7.2. Vérifier la Fraîcheur des Statistiques

Statistiques obsolètes = mauvaises décisions du planificateur :

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS current_rows,
    n_mod_since_analyze AS rows_modified_since_analyze,
    round(
        100.0 * n_mod_since_analyze / NULLIF(n_live_tup, 0),
        2
    ) AS staleness_pct,
    last_analyze,
    last_autoanalyze,
    CASE
        WHEN n_mod_since_analyze > n_live_tup * 0.1 THEN 'ANALYZE recommended'
        ELSE 'OK'
    END AS recommendation
FROM
    pg_stat_user_tables
WHERE
    n_live_tup > 0
ORDER BY
    n_mod_since_analyze DESC
LIMIT 20;
```

**Analyse** :

- **staleness_pct > 10%** : Statistiques obsolètes
- **last_autoanalyze très ancien** : Augmentez la fréquence d'autovacuum

**Solution** :

```sql
ANALYZE ma_table;
-- Ou pour toute la base :
ANALYZE;
```

### 7.3. Statistiques Étendues (Extended Statistics)

PostgreSQL 10+ supporte les statistiques multi-colonnes :

```sql
-- Créer des statistiques étendues
CREATE STATISTICS stats_orders_user_status
ON user_id, status
FROM orders;

-- Mettre à jour
ANALYZE orders;

-- Voir les statistiques étendues
SELECT
    stxnamespace::regnamespace AS schema,
    stxname AS statistics_name,
    stxkeys AS column_ids,
    (SELECT string_agg(attname, ', ')
     FROM pg_attribute
     WHERE attrelid = stxrelid
     AND attnum = ANY(stxkeys)) AS columns
FROM
    pg_statistic_ext
ORDER BY
    stxname;
```

**Utilité** : Aide le planificateur quand deux colonnes sont corrélées.

---

## 8. Mise en Pratique : Scénarios d'Analyse

### Scénario 1 : Prévoir les Besoins en Stockage

**Objectif** : Estimer quand vous manquerez d'espace disque.

**Étapes** :

1. **Mesurer la taille actuelle** :

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

2. **Analyser la croissance (30 derniers jours)** :

```sql
-- Utiliser la table de tracking (section 2.5)
SELECT
    pg_size_pretty(MAX(total_size_bytes) - MIN(total_size_bytes)) AS growth_30_days,
    round(
        (MAX(total_size_bytes) - MIN(total_size_bytes))::numeric / 30,
        0
    ) AS avg_daily_growth_bytes
FROM
    monitoring.table_size_history
WHERE
    measured_at > now() - interval '30 days';
```

3. **Extrapoler** :

```sql
-- Calculer les jours restants avant saturation
WITH current_state AS (
    SELECT
        pg_database_size(current_database()) AS current_bytes,
        -- Remplacer par votre capacité disque réelle
        500 * 1024^3 AS disk_capacity_bytes  -- 500 GB
),
growth AS (
    SELECT
        (MAX(total_size_bytes) - MIN(total_size_bytes)) / 30 AS daily_growth
    FROM
        monitoring.table_size_history
    WHERE
        measured_at > now() - interval '30 days'
)
SELECT
    pg_size_pretty(cs.current_bytes) AS current_size,
    pg_size_pretty(cs.disk_capacity_bytes) AS disk_capacity,
    round(
        (cs.disk_capacity_bytes - cs.current_bytes)::numeric / g.daily_growth,
        0
    ) AS days_until_full
FROM
    current_state cs,
    growth g;
```

### Scénario 2 : Identifier les Candidats au Partitionnement

**Objectif** : Trouver les tables à partitionner.

**Critères** :

- Table > 100 GB
- Croissance rapide
- Requêtes filtrant sur une colonne temporelle

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup AS rows,
    (SELECT attname
     FROM pg_stats
     WHERE schemaname = pt.schemaname
     AND tablename = pt.tablename
     AND atttypid = 'timestamp'::regtype
     LIMIT 1) AS timestamp_column,
    CASE
        WHEN pg_total_relation_size(schemaname||'.'||tablename) > 100 * 1024^3
        THEN 'Candidate for partitioning'
        ELSE 'OK'
    END AS recommendation
FROM
    pg_tables pt
JOIN
    pg_stat_user_tables pst
    ON pt.schemaname = pst.schemaname
    AND pt.tablename = pst.relname
WHERE
    pt.schemaname NOT IN ('pg_catalog', 'information_schema')
    AND pg_total_relation_size(pt.schemaname||'.'||pt.tablename) > 50 * 1024^3  -- > 50 GB
ORDER BY
    pg_total_relation_size(pt.schemaname||'.'||pt.tablename) DESC;
```

### Scénario 3 : Audit de Santé Complet

**Checklist complète** pour une base de données :

```sql
-- Vue d'ensemble
WITH db_overview AS (
    SELECT
        pg_size_pretty(pg_database_size(current_database())) AS total_size,
        (SELECT count(*) FROM pg_tables WHERE schemaname = 'public') AS table_count,
        (SELECT count(*) FROM pg_indexes WHERE schemaname = 'public') AS index_count
),
cache_health AS (
    SELECT
        round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio
    FROM pg_stat_database
    WHERE datname = current_database()
),
bloat_summary AS (
    SELECT
        count(*) FILTER (WHERE n_dead_tup > n_live_tup * 0.2) AS tables_needing_vacuum
    FROM pg_stat_user_tables
),
stats_health AS (
    SELECT
        count(*) FILTER (WHERE n_mod_since_analyze > n_live_tup * 0.1) AS tables_needing_analyze
    FROM pg_stat_user_tables
)
SELECT
    'Database Size' AS metric,
    total_size AS value
FROM db_overview
UNION ALL
SELECT 'Tables', table_count::text FROM db_overview
UNION ALL
SELECT 'Indexes', index_count::text FROM db_overview
UNION ALL
SELECT 'Cache Hit Ratio', cache_hit_ratio::text || '%' FROM cache_health
UNION ALL
SELECT 'Tables Needing VACUUM', tables_needing_vacuum::text FROM bloat_summary
UNION ALL
SELECT 'Tables Needing ANALYZE', tables_needing_analyze::text FROM stats_health;
```

### Scénario 4 : Comparer Deux Environnements

**Comparer Production vs Staging** :

```sql
-- À exécuter sur les deux environnements
WITH env_stats AS (
    SELECT
        'PRODUCTION' AS environment,  -- Changer en 'STAGING' sur l'autre env
        pg_database_size(current_database()) AS db_size,
        (SELECT sum(n_live_tup) FROM pg_stat_user_tables) AS total_rows,
        (SELECT count(*) FROM pg_tables WHERE schemaname = 'public') AS table_count
)
SELECT * FROM env_stats;
```

**Comparer les résultats** pour valider que staging est représentatif.

---

## 9. Maintenance des Statistiques

### 9.1. ANALYZE : Mise à Jour des Statistiques

**Quand exécuter ANALYZE ?**

- Après gros INSERT/UPDATE/DELETE (> 10% de la table)
- Après CREATE INDEX
- Si les plans d'exécution semblent sous-optimaux
- Régulièrement (automatique via autovacuum)

**Commandes** :

```sql
-- Analyser une table spécifique
ANALYZE ma_table;

-- Analyser une colonne spécifique
ANALYZE ma_table(ma_colonne);

-- Analyser toute la base
ANALYZE;

-- Analyser avec VERBOSE (voir les détails)
ANALYZE VERBOSE ma_table;
```

**Nouveauté PostgreSQL 18** : ANALYZE plus rapide grâce aux optimisations I/O.

### 9.2. Configuration de l'Autoanalyze

Dans `postgresql.conf` :

```conf
# Activer autovacuum (inclut autoanalyze)
autovacuum = on

# Seuil de déclenchement pour ANALYZE
autovacuum_analyze_threshold = 50       # Minimum 50 lignes modifiées
autovacuum_analyze_scale_factor = 0.1   # + 10% de la table

# Exemple : Table de 1M lignes
# ANALYZE se déclenche après : 50 + (1M * 0.1) = 100,050 modifications
```

**PostgreSQL 18** : Nouveaux paramètres pour ajuster autovacuum dynamiquement.

### 9.3. Forcer ANALYZE sur Tables Critiques

Pour les tables où les statistiques sont critiques :

```sql
-- Analyser avec un échantillonnage plus large
ALTER TABLE ma_table SET (autovacuum_analyze_scale_factor = 0.05);

-- Ou manuellement avec target plus élevé
SET default_statistics_target = 1000;  -- Défaut : 100
ANALYZE ma_table;
RESET default_statistics_target;
```

### 9.4. Monitoring de l'Autoanalyze

Vérifier que autoanalyze s'exécute :

```sql
SELECT
    schemaname,
    relname,
    last_autoanalyze,
    autoanalyze_count,
    n_mod_since_analyze
FROM
    pg_stat_user_tables
WHERE
    schemaname = 'public'
ORDER BY
    n_mod_since_analyze DESC
LIMIT 20;
```

**Si last_autoanalyze est NULL ou très ancien** : Vérifiez la configuration.

---

## 10. Bonnes Pratiques

### 10.1. Fréquence d'Analyse Recommandée

**Quotidien** :
- Taille de toutes les tables (top 50)
- Cache hit ratio global
- Nombre de connexions

**Hebdomadaire** :
- Bloat par table
- Statistiques obsolètes
- Distribution des données (pour tables partitionnées)

**Mensuel** :
- Audit complet (scénario 3)
- Revue des index (utilisation, taille)
- Statistiques de colonnes (cardinalité)

### 10.2. Création de Rapports Automatisés

**Script exemple (à planifier avec cron)** :

```bash
#!/bin/bash
# daily_stats_report.sh

PGDATABASE="ma_base"
OUTPUT_DIR="/var/reports/postgres"
DATE=$(date +%Y%m%d)

# Taille des tables
psql $PGDATABASE -c "
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
" > "$OUTPUT_DIR/table_sizes_$DATE.txt"

# Bloat
psql $PGDATABASE -c "
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
" > "$OUTPUT_DIR/bloat_$DATE.txt"
```

**Planifier avec cron** :

```bash
# Tous les jours à 2h du matin
0 2 * * * /path/to/daily_stats_report.sh
```

### 10.3. Alertes sur Métriques Clés

**Métriques à alerter** :

| Métrique | Seuil Warning | Seuil Critical |
|----------|---------------|----------------|
| Taille DB | > 80% capacité | > 90% capacité |
| Table bloat | > 20% | > 40% |
| Stats obsolètes | > 10% modif. | > 30% modif. |
| Croissance rapide | > 10 GB/jour | > 50 GB/jour |

### 10.4. Documentation des Changements

**Tenir un registre** :

- Date d'analyse
- Métriques avant/après
- Actions prises (VACUUM, REINDEX, partitionnement)
- Impact observé

**Template** :

```
Date: 2025-11-21
Table: orders
Action: VACUUM FULL
Before: 45 GB (20% bloat)
After: 36 GB (2% bloat)
Downtime: 30 minutes
Impact: 20% faster queries
```

### 10.5. Tests Avant Modifications

**Avant tout changement majeur** :

1. **Backup** : Toujours
2. **Test en staging** : Valider l'impact
3. **Mesure baseline** : Statistiques avant
4. **Mesure post-change** : Statistiques après
5. **Rollback plan** : Au cas où

### 10.6. Utilisation de Vues de Monitoring

Créez des vues pour simplifier l'analyse :

```sql
-- Vue : Santé générale de la base
CREATE VIEW monitoring.database_health AS
SELECT
    'Total Size' AS metric,
    pg_size_pretty(pg_database_size(current_database())) AS value
UNION ALL
SELECT
    'Cache Hit Ratio',
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)::text || '%'
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT
    'Tables Needing VACUUM',
    count(*)::text
FROM pg_stat_user_tables
WHERE n_dead_tup > n_live_tup * 0.2;

-- Utilisation simple
SELECT * FROM monitoring.database_health;
```

### 10.7. Intégration avec Outils de Monitoring

**Exporter vers Prometheus** :

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
```

**Dashboard Grafana recommandés** :
- Dashboard ID 9628 : PostgreSQL Database Overview
- Dashboard ID 12485 : PostgreSQL Detailed

### 10.8. Formation de l'Équipe

**Points essentiels à former** :

1. Lecture des statistiques de base (taille, bloat)
2. Interprétation d'EXPLAIN ANALYZE
3. Quand et comment exécuter VACUUM/ANALYZE
4. Utilisation de pg_stat_statements
5. Procédures d'urgence (table saturée, stats obsolètes)

---

## Résumé des Points Clés

### Analyse de Taille

- ✅ **Fonctions natives** : pg_relation_size, pg_total_relation_size, pg_database_size
- ✅ **Top 20 tables** : Surveiller régulièrement (section 2.2)
- ✅ **Ratio table/index** : Détecter sur-indexation (section 2.3)
- ✅ **Tracking croissance** : Table historique pour prédire (section 2.5)

### Statistiques de Tables

- ✅ **pg_stat_user_tables** : Lignes live/dead, scans, I/O
- ✅ **PostgreSQL 18** : Nouveaux compteurs vacuum_count et analyze_count
- ✅ **Tables actives** : Identifier lectures et écritures (sections 3.2 et 3.3)
- ✅ **Croissance pattern** : Append-only vs Stable vs Growing (section 3.4)

### Statistiques de Colonnes

- ✅ **pg_stats** : Cardinalité, NULL %, distribution
- ✅ **Corrélation** : Ordre physique vs logique (section 4.4)
- ✅ **MCV** : Valeurs les plus fréquentes (section 4.3)
- ✅ **Histogrammes** : Distribution pour le planificateur (section 4.6)

### Fragmentation

- ✅ **pgstattuple** : Mesure précise du bloat
- ✅ **Estimation rapide** : Via pg_stat_user_tables
- ✅ **Index** : pgstatindex pour les index B-Tree
- ✅ **Solutions** : VACUUM, VACUUM FULL, REINDEX, pg_repack

### Maintenance

- ✅ **ANALYZE** : Après modifications importantes
- ✅ **Autoanalyze** : Configuration autovacuum_analyze_*
- ✅ **Statistiques étendues** : CREATE STATISTICS pour colonnes corrélées
- ✅ **Monitoring** : Vérifier last_autoanalyze régulièrement

---

## Ressources Complémentaires

### Documentation Officielle PostgreSQL

- [Monitoring Database Activity](https://www.postgresql.org/docs/18/monitoring.html)
- [Statistics Used by the Planner](https://www.postgresql.org/docs/18/planner-stats.html)
- [Routine Database Maintenance](https://www.postgresql.org/docs/18/maintenance.html)
- [System Catalogs](https://www.postgresql.org/docs/18/catalogs.html)

### Extensions Recommandées

- **pgstattuple** : Analyse détaillée du bloat
- **pg_stat_statements** : Tracking des requêtes
- **pg_repack** : Réorganisation sans verrous
- **HypoPG** : Test d'index hypothétiques

### Outils d'Analyse

- **pgAdmin 4** : Interface graphique avec monitoring
- **pgBadger** : Analyse de logs
- **pg_activity** : Top-like pour PostgreSQL
- **pgtop** : Monitoring en temps réel

### Scripts Communautaires

- [check_postgres](https://bucardo.org/check_postgres/) : Scripts Nagios/Icinga
- [pgDash](https://pgdash.io/) : Monitoring et diagnostics
- [Postgres.ai](https://postgres.ai/) : Optimisation automatisée

### Communautés et Support

- Mailing list : pgsql-admin@postgresql.org
- Reddit : r/PostgreSQL
- Discord PostgreSQL (communauté francophone)
- Stack Overflow : Tag `postgresql-administration`

---


⏭️ [Configuration de Référence par Cas d'Usage](/annexes/configuration-reference/README.md)
