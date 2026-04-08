🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Requêtes de Monitoring PostgreSQL
## Cache Hit Ratio et Slow Queries

---

## Table des Matières

1. [Introduction au Monitoring PostgreSQL](#1-introduction-au-monitoring-postgresql)  
2. [Cache Hit Ratio : Comprendre et Optimiser](#2-cache-hit-ratio--comprendre-et-optimiser)  
3. [Slow Queries : Identifier et Corriger](#3-slow-queries--identifier-et-corriger)  
4. [Métriques Complémentaires Essentielles](#4-m%C3%A9triques-compl%C3%A9mentaires-essentielles)  
5. [Mise en Pratique : Diagnostic de Performance](#5-mise-en-pratique--diagnostic-de-performance)  
6. [Automatisation et Alerting](#6-automatisation-et-alerting)  
7. [Bonnes Pratiques de Monitoring](#7-bonnes-pratiques-de-monitoring)

---

## 1. Introduction au Monitoring PostgreSQL

### 1.1. Pourquoi Monitorer Votre Base de Données ?

Le monitoring (surveillance) de PostgreSQL est **essentiel** pour :

1. **Détecter les problèmes avant qu'ils ne deviennent critiques**
   - Une requête lente aujourd'hui = une panne demain
   - Un cache inefficace = des coûts serveur multipliés

2. **Optimiser les performances**
   - Identifier les goulots d'étranglement
   - Valider l'impact des optimisations

3. **Planifier la capacité**
   - Savoir quand ajouter de la RAM, du CPU, du stockage
   - Éviter les surprises lors des pics de charge

4. **Garantir la disponibilité**
   - Respecter les SLA (Service Level Agreement)
   - Maintenir une expérience utilisateur fluide

### 1.2. Les Deux Piliers du Monitoring PostgreSQL

Ce tutoriel se concentre sur les deux métriques les plus importantes :

#### Cache Hit Ratio (Taux de succès du cache)

**Qu'est-ce que c'est ?**
Le pourcentage de fois où PostgreSQL trouve les données dans la **mémoire RAM** plutôt que de devoir les lire sur le **disque**.

**Pourquoi c'est important ?**
- Lire en RAM : **~100 nanosecondes**
- Lire sur disque SSD : **~100 microsecondes** (1000× plus lent)
- Lire sur disque HDD : **~10 millisecondes** (100 000× plus lent)

**Objectif** : Viser un cache hit ratio **> 95%**

#### Slow Queries (Requêtes lentes)

**Qu'est-ce que c'est ?**
Les requêtes SQL qui prennent **trop de temps** à s'exécuter.

**Pourquoi c'est important ?**
- 1 requête lente peut bloquer des centaines d'utilisateurs
- Consomme des ressources (CPU, RAM, I/O)
- Dégrade l'expérience utilisateur

**Objectif** : Identifier et optimiser toute requête > 100-200ms

### 1.3. Les Vues Système PostgreSQL pour le Monitoring

PostgreSQL expose des **vues système** qui collectent automatiquement des statistiques :

| Vue | Usage |
|-----|-------|
| `pg_stat_database` | Statistiques globales par base de données |
| `pg_stat_user_tables` | Statistiques par table (scans, tuples) |
| `pg_stat_user_indexes` | Statistiques par index (scans, tuples) |
| `pg_statio_user_tables` | Statistiques I/O par table |
| `pg_statio_user_indexes` | Statistiques I/O par index |
| `pg_stat_statements` | **Extension** : Toutes les requêtes exécutées |
| `pg_stat_activity` | Activité en temps réel (connexions, requêtes) |

### 1.4. Configuration Préalable : pg_stat_statements

L'extension **pg_stat_statements** est **indispensable** pour monitorer les slow queries. Elle doit être activée manuellement.

#### Installation (une seule fois par base de données)

```sql
-- Se connecter à votre base de données
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

#### Configuration dans postgresql.conf

```conf
# Ajouter dans postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

# Configuration recommandée
pg_stat_statements.max = 10000          # Nombre de requêtes trackées  
pg_stat_statements.track = all          # Tracker toutes les requêtes  
pg_stat_statements.track_utility = on   # Inclure DDL (CREATE, ALTER, etc.)  
```

**Redémarrage requis** après modification de `shared_preload_libraries`.

#### Vérifier que l'extension est active

```sql
SELECT * FROM pg_stat_statements LIMIT 5;
```

Si la requête retourne des résultats, c'est bon ! Sinon, vérifiez la configuration.

---

## 2. Cache Hit Ratio : Comprendre et Optimiser

### 2.1. Qu'est-ce que le Buffer Cache ?

PostgreSQL utilise une zone de mémoire RAM appelée **Shared Buffers** pour mettre en cache les pages de données lues depuis le disque.

#### Fonctionnement en Détail

1. **Première lecture** : Données lues depuis le disque → Copiées dans le buffer cache  
2. **Lectures suivantes** : Si les données sont encore en cache → Lecture instantanée  
3. **Cache plein** : Algorithme LRU (Least Recently Used) évince les données anciennes

**Configuration** : Le paramètre `shared_buffers` dans `postgresql.conf`

```conf
# Recommandation générale : 25% de la RAM
# Serveur avec 16 GB RAM → shared_buffers = 4 GB
shared_buffers = 4GB
```

### 2.2. Cache Hit Ratio : Formule et Interprétation

Le cache hit ratio se calcule ainsi :

```
Cache Hit Ratio = (blks_hit / (blks_hit + blks_read)) × 100
```

Où :
- **blks_hit** : Nombre de blocs lus depuis le cache (RAM)  
- **blks_read** : Nombre de blocs lus depuis le disque

**Interprétation** :

| Ratio | État | Action |
|-------|------|--------|
| > 99% | **Excellent** | Continuez ainsi |
| 95-99% | **Bon** | Acceptable pour la plupart des cas |
| 90-95% | **Moyen** | Envisagez d'augmenter shared_buffers |
| < 90% | **Mauvais** | Action urgente requise |

### 2.3. Requête : Cache Hit Ratio Global (Base de Données)

Cette requête mesure le cache hit ratio pour toute la base de données :

```sql
SELECT
    datname AS database_name,
    blks_hit AS cache_hits,
    blks_read AS disk_reads,
    blks_hit + blks_read AS total_reads,
    round(
        100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0),
        2
    ) AS cache_hit_ratio_pct
FROM
    pg_stat_database
WHERE
    datname IS NOT NULL
    AND (blks_hit + blks_read) > 0
ORDER BY
    cache_hit_ratio_pct ASC;
```

**Ce que vous cherchez** :
- Votre base de données doit avoir un ratio > 95%
- Si < 95%, c'est un signal d'alerte

**Exemple de résultat** :

```
database_name | cache_hits | disk_reads | total_reads | cache_hit_ratio_pct
--------------+------------+------------+-------------+--------------------
production    | 15234567   | 456789     | 15691356    | 97.09
```

### 2.4. Requête : Cache Hit Ratio par Table

Pour identifier quelles tables causent des lectures disque excessives :

```sql
SELECT
    schemaname,
    relname AS table_name,
    heap_blks_hit AS cache_hits,
    heap_blks_read AS disk_reads,
    heap_blks_hit + heap_blks_read AS total_reads,
    round(
        100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0),
        2
    ) AS cache_hit_ratio_pct,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size
FROM
    pg_statio_user_tables
WHERE
    (heap_blks_hit + heap_blks_read) > 100  -- Ignorer les tables peu utilisées
ORDER BY
    cache_hit_ratio_pct ASC
LIMIT 20;
```

**Analyse** :

- **Table avec ratio < 90%** : Candidate à l'optimisation ou au partitionnement  
- **Grosse table avec ratio < 90%** : Critique, action immédiate  
- **Petite table avec ratio < 90%** : Moins grave mais à surveiller

**Exemple de résultat** :

```
table_name | cache_hits | disk_reads | cache_hit_ratio_pct | total_size
-----------+------------+------------+---------------------+-----------
orders     | 1234       | 5678       | 17.85               | 45 GB  
products   | 89012      | 123        | 99.86               | 2 GB  
```

→ La table `orders` a un ratio catastrophique : **17.85%**. Il faut agir.

### 2.5. Requête : Cache Hit Ratio par Index

Les index aussi bénéficient du cache :

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_blks_hit AS cache_hits,
    idx_blks_read AS disk_reads,
    idx_blks_hit + idx_blks_read AS total_reads,
    round(
        100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0),
        2
    ) AS cache_hit_ratio_pct,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM
    pg_statio_user_indexes
WHERE
    (idx_blks_hit + idx_blks_read) > 100
ORDER BY
    cache_hit_ratio_pct ASC
LIMIT 20;
```

**Analyse** :

- Un index avec un mauvais ratio consomme beaucoup d'I/O
- Si l'index est peu utilisé (`idx_scan` faible) : Envisagez de le supprimer
- Si l'index est très utilisé : C'est normal qu'il soit souvent lu depuis le disque

### 2.6. Diagnostiquer un Mauvais Cache Hit Ratio

Si votre cache hit ratio est < 95%, plusieurs causes possibles :

#### Cause 1 : shared_buffers Trop Petit

**Solution** : Augmenter `shared_buffers` dans `postgresql.conf`

```conf
# Avant
shared_buffers = 128MB

# Après (pour serveur avec 16 GB RAM)
shared_buffers = 4GB
```

**Attention** : Nécessite un redémarrage de PostgreSQL.

#### Cause 2 : Workload "Cold" (Démarrage Récent)

Juste après un redémarrage, le cache est vide. C'est normal d'avoir un ratio bas temporairement.

**Solution** : Attendre quelques heures que le cache se "réchauffe".

#### Cause 3 : Données Plus Grosses que la RAM

Si votre base de données fait 500 GB et que vous avez 16 GB de RAM, vous ne pourrez **jamais** tout mettre en cache.

**Solutions** :
1. Ajouter plus de RAM  
2. Partitionner les tables pour réduire la surface de scan  
3. Archiver les données anciennes  
4. Utiliser un réplica en lecture seule pour répartir la charge

#### Cause 4 : Requêtes Inefficaces (Scan Séquentiels)

Des requêtes qui font des scans séquentiels sur de grosses tables vont systématiquement lire depuis le disque.

**Solution** : Créer des index appropriés (voir section 3).

### 2.7. Requête : Taille de la Base vs Shared Buffers

Pour évaluer si votre cache est dimensionné correctement :

```sql
SELECT
    pg_size_pretty(pg_database_size(current_database())) AS database_size,
    current_setting('shared_buffers') AS shared_buffers_setting,
    pg_size_pretty(
        pg_size_bytes(current_setting('shared_buffers'))
    ) AS shared_buffers_size,
    round(
        100.0 * pg_size_bytes(current_setting('shared_buffers')) /
        pg_database_size(current_database()),
        2
    ) AS cache_coverage_pct;
```

**Interprétation** :

- **cache_coverage_pct > 50%** : Excellent, une grande partie de la DB peut tenir en cache  
- **cache_coverage_pct < 10%** : Très peu de données en cache, augmentez shared_buffers si possible

### 2.8. Nouveauté PostgreSQL 18 : Statistiques I/O par Backend

PostgreSQL 18 ajoute des statistiques I/O détaillées par processus backend :

```sql
-- Requête pour voir les I/O par session active
SELECT
    pid,
    usename,
    application_name,
    query,
    backend_type,
    -- Statistiques I/O (PostgreSQL 18)
    CASE
        WHEN backend_type = 'client backend' THEN 'Active Connection'
        ELSE backend_type
    END AS connection_type
FROM
    pg_stat_activity
WHERE
    state = 'active'
    AND pid != pg_backend_pid()  -- Exclure cette requête
ORDER BY
    query_start DESC;
```

**Note** : Les colonnes I/O détaillées sont disponibles dans les vues `pg_stat_io` (PostgreSQL 18).

---

## 3. Slow Queries : Identifier et Corriger

### 3.1. Qu'est-ce qu'une Slow Query ?

Une **slow query (requête lente)** est une requête SQL qui prend plus de temps que souhaité à s'exécuter.

**Seuils typiques** :

| Type d'application | Seuil "lent" | Seuil "critique" |
|-------------------|--------------|------------------|
| API Web temps réel | > 100ms | > 500ms |
| Dashboard analytics | > 1s | > 10s |
| Batch processing | > 10s | > 60s |

**Impact** :

- **Utilisateur** : Timeouts, frustration  
- **Serveur** : Consommation CPU/RAM excessive  
- **Base de données** : Verrous prolongés, congestion

### 3.2. Installation et Configuration de pg_stat_statements

**Rappel** : pg_stat_statements est l'outil indispensable pour tracker les slow queries.

Si ce n'est pas déjà fait, suivez les étapes de la section 1.4.

### 3.3. Requête : Top 20 des Requêtes les Plus Lentes (Temps Total)

Cette requête identifie les requêtes qui ont consommé le **plus de temps cumulé** :

```sql
SELECT
    round(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round(max_exec_time::numeric, 2) AS max_time_ms,
    round(min_exec_time::numeric, 2) AS min_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_total_time,
    query
FROM
    pg_stat_statements
WHERE
    query NOT LIKE '%pg_stat_statements%'  -- Exclure les requêtes de monitoring
ORDER BY
    total_exec_time DESC
LIMIT 20;
```

**Colonnes importantes** :

- **total_time_ms** : Temps cumulé total (si une requête s'exécute 1000× pendant 10ms = 10 000ms)  
- **calls** : Nombre d'exécutions  
- **avg_time_ms** : Temps moyen par exécution  
- **max_time_ms** : Temps maximum enregistré (pic)  
- **pct_total_time** : Pourcentage du temps total de toutes les requêtes

**Analyse** :

- **Requête avec total_time élevé + avg_time faible** : Requête fréquente, optimiser réduira beaucoup la charge  
- **Requête avec avg_time très élevé** : Requête lente, même si rare

**Exemple de résultat** :

```
total_time_ms | calls | avg_time_ms | pct_total_time | query
--------------+-------+-------------+----------------+-------
125678.45     | 50000 | 2.51        | 45.23          | SELECT * FROM orders WHERE user_id = $1
23456.78      | 100   | 234.57      | 8.44           | SELECT COUNT(*) FROM logs WHERE date > $1
```

→ La première requête représente **45% du temps total** : **priorité absolue**.

### 3.4. Requête : Top 20 des Requêtes les Plus Lentes (Temps Moyen)

Pour identifier les requêtes individuellement lentes :

```sql
SELECT
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round(max_exec_time::numeric, 2) AS max_time_ms,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_time_ms,
    query
FROM
    pg_stat_statements
WHERE
    calls > 10  -- Ignorer les requêtes exécutées trop rarement
    AND query NOT LIKE '%pg_stat_statements%'
ORDER BY
    mean_exec_time DESC
LIMIT 20;
```

**Analyse** :

- **avg_time_ms > 1000ms (1s)** : Très lent, optimisation urgente  
- **stddev_time_ms élevé** : Performance instable, peut indiquer des verrous ou de la contention

### 3.5. Requête : Requêtes Avec le Plus de Lectures Disque

Les requêtes qui lisent beaucoup depuis le disque sont souvent lentes :

```sql
SELECT
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    calls,
    shared_blks_hit AS cache_hits,
    shared_blks_read AS disk_reads,
    round(
        100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0),
        2
    ) AS cache_hit_ratio_pct,
    query
FROM
    pg_stat_statements
WHERE
    (shared_blks_hit + shared_blks_read) > 1000  -- Au moins 1000 blocs lus
ORDER BY
    shared_blks_read DESC
LIMIT 20;
```

**Analyse** :

- **disk_reads élevé** : La requête lit beaucoup depuis le disque  
- **cache_hit_ratio_pct < 90%** : Mauvais taux de cache pour cette requête

**Solution** : Ajouter un index ou augmenter shared_buffers.

### 3.6. Requête : Requêtes en Cours d'Exécution (Temps Réel)

Pour voir les requêtes **actuellement en train de s'exécuter** :

```sql
SELECT
    pid,
    usename AS user,
    application_name,
    client_addr,
    state,
    query_start,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM
    pg_stat_activity
WHERE
    state = 'active'
    AND pid != pg_backend_pid()  -- Exclure cette requête
    AND query NOT LIKE '%pg_stat_activity%'
ORDER BY
    query_start ASC;  -- Les plus anciennes d'abord
```

**Colonnes importantes** :

- **duration** : Depuis combien de temps la requête tourne  
- **wait_event_type** / **wait_event** : Ce que la requête attend (Lock, I/O, etc.)  
- **query** : La requête SQL en cours

**Si duration > 5 minutes** : Requête probablement bloquée ou très lente.

### 3.7. Analyser une Slow Query avec EXPLAIN ANALYZE

Une fois qu'une requête lente est identifiée, il faut comprendre **pourquoi** elle est lente.

**Outil** : `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE  
SELECT * FROM orders  
WHERE user_id = 12345  
AND created_at > '2024-01-01';  
```

**Résultat typique** :

```
Seq Scan on orders  (cost=0.00..125678.45 rows=123 width=200)
                    (actual time=0.234..2345.678 rows=123 loops=1)
  Filter: (user_id = 12345 AND created_at > '2024-01-01'::date)
  Rows Removed by Filter: 4567890
Planning Time: 0.123 ms  
Execution Time: 2345.789 ms  
```

**Ce qu'il faut chercher** :

1. **Seq Scan (Scan Séquentiel)** : PostgreSQL lit toute la table  
   - **Solution** : Créer un index

2. **Rows Removed by Filter élevé** : Beaucoup de lignes lues pour rien  
   - **Solution** : Index plus sélectif

3. **Nested Loop avec grosse table** : Jointure inefficace  
   - **Solution** : Revoir la jointure ou ajouter un index

4. **Sort + High Memory** : Tri coûteux  
   - **Solution** : Index sur la colonne de tri

### 3.8. Exemple Concret : Optimiser une Slow Query

**Requête lente identifiée** :

```sql
SELECT * FROM orders  
WHERE status = 'pending'  
ORDER BY created_at DESC  
LIMIT 20;  
```

**EXPLAIN ANALYZE** montre :

```
Sort  (cost=5678.90..5679.15 rows=100 width=200) (actual time=1234.567..1234.890 rows=20 loops=1)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 25kB
  ->  Seq Scan on orders  (cost=0.00..5675.00 rows=100 width=200) (actual time=0.123..1234.000 rows=100 loops=1)
        Filter: (status = 'pending'::text)
        Rows Removed by Filter: 5000000
Planning Time: 0.234 ms  
Execution Time: 1234.987 ms  
```

**Problème** : Scan séquentiel sur 5 millions de lignes.

**Solution** : Créer un index :

```sql
CREATE INDEX idx_orders_status_created_at  
ON orders(status, created_at DESC);  
```

**Après création de l'index** :

```
Index Scan using idx_orders_status_created_at on orders
  (cost=0.43..25.67 rows=20 width=200) (actual time=0.012..0.034 rows=20 loops=1)
  Index Cond: (status = 'pending'::text)
Planning Time: 0.123 ms  
Execution Time: 0.045 ms  
```

**Résultat** : De **1234ms à 0.045ms** → **27 000× plus rapide** ! 🚀

### 3.9. Nouveauté PostgreSQL 18 : Optimisations du Planificateur

PostgreSQL 18 apporte plusieurs optimisations automatiques :

#### 1. Auto-élimination des Self-Joins

**Avant PostgreSQL 18** :

```sql
SELECT o1.*  
FROM orders o1  
JOIN orders o2 ON o1.id = o2.id  
WHERE o1.status = 'pending';  
```

PostgreSQL devait exécuter la jointure même si redondante.

**Avec PostgreSQL 18** : Le planificateur détecte et élimine le self-join automatiquement.

#### 2. Optimisation OR → ANY

**Avant** :

```sql
SELECT * FROM users  
WHERE status = 'active' OR status = 'pending' OR status = 'trial';  
```

**PostgreSQL 18** réécrit automatiquement en :

```sql
SELECT * FROM users  
WHERE status = ANY(ARRAY['active', 'pending', 'trial']);  
```

Plus efficace pour l'utilisation d'index.

#### 3. Skip Scan pour Index Multi-Colonnes

Index sur `(a, b)` peut maintenant être utilisé efficacement pour `WHERE b = X` (sans condition sur `a`).

### 3.10. Réinitialiser les Statistiques pg_stat_statements

Après avoir optimisé vos requêtes, vous pouvez réinitialiser les statistiques :

```sql
-- Réinitialiser toutes les statistiques de requêtes
SELECT pg_stat_statements_reset();

-- Réinitialiser les statistiques d'une base spécifique
SELECT pg_stat_reset();
```

**Quand le faire** :
- Après déploiement d'optimisations majeures
- Pour mesurer l'impact d'un changement
- Régulièrement (ex: mensuel) pour avoir des stats fraîches

**Attention** : Vous perdez l'historique !

---

## 4. Métriques Complémentaires Essentielles

### 4.1. Nombre de Connexions Actives

Trop de connexions = problème de performance.

```sql
SELECT
    count(*) FILTER (WHERE state = 'active') AS active_connections,
    count(*) FILTER (WHERE state = 'idle') AS idle_connections,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction,
    count(*) AS total_connections,
    current_setting('max_connections')::int AS max_connections,
    round(100.0 * count(*) / current_setting('max_connections')::int, 2) AS usage_pct
FROM
    pg_stat_activity;
```

**Seuils d'alerte** :

- **usage_pct > 80%** : Proche de la saturation  
- **idle_in_transaction élevé** : Transactions ouvertes inutilement (fuites de connexions)

**Solution** : Utiliser un connection pooler (PgBouncer).

### 4.2. Taille des Tables et Index

Surveiller la croissance pour prévoir les besoins en stockage :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS indexes_size,
    pg_total_relation_size(schemaname||'.'||tablename) AS bytes_total
FROM
    pg_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    bytes_total DESC
LIMIT 20;
```

**Analyse** :

- Si `indexes_size > table_size` : Vous avez peut-être trop d'index
- Croissance rapide : Prévoir l'archivage ou le partitionnement

### 4.3. Checkpoint Frequency et WAL Generation

Les checkpoints fréquents indiquent une écriture intensive :

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend
FROM
    pg_stat_bgwriter;
```

**Analyse** :

- **checkpoints_req > checkpoints_timed** : Checkpoints forcés, augmentez `max_wal_size`  
- **checkpoint_write_time élevé** : Disque lent ou surcharge I/O

**Configuration recommandée** :

```conf
max_wal_size = 4GB  # Plus haut si écriture intensive  
checkpoint_timeout = 15min  
checkpoint_completion_target = 0.9  
```

### 4.4. Transaction ID (XID) Wraparound Risk

PostgreSQL utilise des identifiants de transaction (XID) sur 32 bits, qui peuvent "boucler". Si cela arrive sans VACUUM, c'est la **panne garantie**.

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    2^31 - 1000000 - age(datfrozenxid) AS xids_remaining,
    round(100.0 * age(datfrozenxid) / (2^31 - 1000000), 2) AS wraparound_risk_pct
FROM
    pg_database
ORDER BY
    age(datfrozenxid) DESC;
```

**Seuils critiques** :

- **xid_age > 1 000 000 000** : Surveiller  
- **xid_age > 1 500 000 000** : Action recommandée (VACUUM FREEZE)  
- **xid_age > 2 000 000 000** : **URGENT**, risque de wraparound

**Solution** :

```sql
VACUUM FREEZE VERBOSE ma_table;
```

### 4.5. Réplication Lag (Si Réplication Activée)

Pour les setups avec réplication (primary/standby) :

```sql
-- Sur le PRIMARY
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 AS replay_lag_mb
FROM
    pg_stat_replication;
```

**Seuils** :

- **replay_lag_mb < 100 MB** : Bon  
- **replay_lag_mb > 1 GB** : Le standby est en retard, enquêter

### 4.6. Nouveauté PostgreSQL 18 : Statistiques VACUUM et ANALYZE

PostgreSQL 18 enrichit les statistiques de maintenance :

```sql
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count,
    n_dead_tup
FROM
    pg_stat_all_tables
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    n_dead_tup DESC
LIMIT 20;
```

**Nouveauté** : Compteurs `vacuum_count` et `analyze_count` pour suivre la fréquence de maintenance.

---

## 5. Mise en Pratique : Diagnostic de Performance

### Scénario 1 : L'Application Est Lente (Diagnostic Général)

**Symptômes** : Tous les utilisateurs se plaignent de lenteur.

**Checklist de diagnostic** :

#### Étape 1 : Vérifier les Connexions

```sql
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```

- Trop de connexions actives ? → PgBouncer
- Beaucoup de `idle in transaction` ? → Fuites de connexions applicatives

#### Étape 2 : Identifier les Requêtes Lentes en Cours

```sql
SELECT pid, usename, query_start, now() - query_start AS duration, query  
FROM pg_stat_activity  
WHERE state = 'active' AND now() - query_start > interval '30 seconds'  
ORDER BY duration DESC;  
```

- Requêtes tournant depuis > 30s ? → Analyser avec EXPLAIN ANALYZE

#### Étape 3 : Vérifier le Cache Hit Ratio

```sql
SELECT round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio  
FROM pg_stat_database  
WHERE datname = current_database();  
```

- Ratio < 95% ? → Augmenter shared_buffers ou optimiser requêtes

#### Étape 4 : Identifier les Top Slow Queries

```sql
SELECT round(mean_exec_time::numeric, 2) AS avg_ms, calls, query  
FROM pg_stat_statements  
WHERE calls > 100  
ORDER BY mean_exec_time DESC  
LIMIT 10;  
```

- Optimiser les requêtes identifiées

### Scénario 2 : Un Endpoint API Précis Est Lent

**Symptômes** : `/api/users/orders` prend 5 secondes, le reste va bien.

**Diagnostic** :

#### Étape 1 : Filtrer par Application Name

```sql
SELECT query, round(mean_exec_time::numeric, 2) AS avg_ms, calls  
FROM pg_stat_statements  
WHERE query ILIKE '%orders%'  -- Mots-clés de votre endpoint  
ORDER BY mean_exec_time DESC;  
```

#### Étape 2 : Analyser la Requête avec EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
[Coller la requête lente ici];
```

#### Étape 3 : Identifier le Goulot

- **Seq Scan** → Créer index  
- **Nested Loop** coûteux → Revoir jointure  
- **Sort** coûteux → Index sur colonne de tri

### Scénario 3 : Disque Saturé (I/O Wait Élevé)

**Symptômes** : Serveur lent, `iostat` montre 100% disk utilization.

**Diagnostic** :

#### Étape 1 : Trouver les Requêtes Gourmandes en I/O

```sql
SELECT query, shared_blks_read, shared_blks_hit  
FROM pg_stat_statements  
ORDER BY shared_blks_read DESC  
LIMIT 10;  
```

#### Étape 2 : Vérifier le Cache Hit Ratio par Table

```sql
SELECT relname, heap_blks_read, heap_blks_hit,
       round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS ratio
FROM pg_statio_user_tables  
WHERE (heap_blks_hit + heap_blks_read) > 0  
ORDER BY heap_blks_read DESC  
LIMIT 10;  
```

**Solutions** :

1. Augmenter `shared_buffers`  
2. Passer à un SSD si HDD  
3. Partitionner les grosses tables  
4. Archiver les données anciennes

### Scénario 4 : Pic de Charge Soudain

**Symptômes** : Ça marchait bien, puis soudainement très lent à 14h.

**Diagnostic** :

#### Étape 1 : Comparer les Statistiques Avant/Après

Si vous avez des snapshots réguliers de `pg_stat_statements` (via un outil de monitoring), comparez.

#### Étape 2 : Identifier les Nouvelles Requêtes

```sql
SELECT query, calls, round(mean_exec_time::numeric, 2) AS avg_ms  
FROM pg_stat_statements  
WHERE calls > 1000  -- Requêtes fréquentes  
ORDER BY calls DESC  
LIMIT 20;  
```

- Une nouvelle requête avec beaucoup d'appels ? → Optimiser en priorité

#### Étape 3 : Vérifier les Verrous

```sql
SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';
```

- Deadlock ou verrous prolongés ? → Voir le tutoriel sur les locks

---

## 6. Automatisation et Alerting

### 6.1. Pourquoi Automatiser le Monitoring ?

Vous ne pouvez pas rester 24/7 devant votre terminal à exécuter ces requêtes. Il faut :

1. **Collecter automatiquement** les métriques  
2. **Stocker l'historique** pour analyser les tendances  
3. **Alerter** quand un seuil est dépassé

### 6.2. Stack de Monitoring Recommandée

#### Option 1 : Prometheus + Grafana (Open Source)

**Architecture** :

```
PostgreSQL → postgres_exporter → Prometheus → Grafana
                                      ↓
                                  Alertmanager (alertes)
```

**Avantages** :
- Gratuit et open source
- Très populaire (nombreux dashboards communautaires)
- Hautement configurable

**Installation rapide** :

```bash
# 1. Installer postgres_exporter
docker run -d \
  --name postgres_exporter \
  -e DATA_SOURCE_NAME="postgresql://user:password@localhost:5432/dbname?sslmode=disable" \
  -p 9187:9187 \
  prometheuscommunity/postgres-exporter

# 2. Configurer Prometheus pour scraper les métriques
# 3. Importer un dashboard Grafana (ex: dashboard ID 9628)
```

#### Option 2 : pgBadger (Analyse de Logs)

**pgBadger** analyse les logs PostgreSQL et génère des rapports HTML.

```bash
# Générer un rapport
pgbadger /var/log/postgresql/postgresql.log -o report.html
```

**Avantages** :
- Rapports détaillés (slow queries, connexions, erreurs)
- Pas besoin d'agent supplémentaire

**Inconvénient** : Pas de monitoring temps réel.

#### Option 3 : Solutions Cloud Natives

- **AWS RDS** : CloudWatch Metrics automatiques  
- **Azure Database for PostgreSQL** : Azure Monitor  
- **GCP Cloud SQL** : Cloud Monitoring

### 6.3. Configurer les Alertes

**Métriques à alerter** :

| Métrique | Seuil Warning | Seuil Critical | Action |
|----------|---------------|----------------|--------|
| Cache Hit Ratio | < 95% | < 90% | Augmenter shared_buffers |
| Connexions actives | > 80% max | > 95% max | Ajouter PgBouncer |
| Requête lente | > 1s | > 5s | Optimiser SQL |
| Réplication lag | > 100 MB | > 1 GB | Vérifier réseau/load |
| XID age | > 1B | > 1.5B | VACUUM FREEZE |
| Disk usage | > 80% | > 90% | Archiver/Nettoyer |

**Exemple de règle Prometheus** :

```yaml
groups:
  - name: postgresql_alerts
    rules:
      - alert: PostgreSQLCacheHitRatioLow
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio is below 95%"
          description: "Database {{ $labels.datname }} has cache hit ratio of {{ $value }}"
```

### 6.4. Créer des Vues pour Simplifier le Monitoring

Plutôt que de retaper les requêtes complexes, créez des vues :

```sql
-- Vue pour le cache hit ratio
CREATE VIEW monitoring.cache_hit_ratio AS  
SELECT  
    datname,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS ratio
FROM pg_stat_database;

-- Vue pour les slow queries
CREATE VIEW monitoring.slow_queries AS  
SELECT  
    round(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    query
FROM pg_stat_statements  
WHERE mean_exec_time > 100  
ORDER BY mean_exec_time DESC;  
```

**Utilisation** :

```sql
-- Simple à requêter ensuite
SELECT * FROM monitoring.cache_hit_ratio;  
SELECT * FROM monitoring.slow_queries;  
```

---

## 7. Bonnes Pratiques de Monitoring

### 7.1. Établir une Baseline (Ligne de Base)

Avant d'optimiser, mesurez d'abord l'état actuel :

1. **Prendre des snapshots réguliers** de `pg_stat_statements`  
2. **Documenter les métriques normales** :
   - Cache hit ratio typique
   - Temps de réponse moyen
   - Charge CPU/RAM/I/O

3. **Comparer après changements** pour valider l'amélioration

### 7.2. Monitoring Régulier vs Incident Response

**Monitoring Régulier** (quotidien/hebdomadaire) :
- Vérifier les tendances (croissance des données)
- Identifier les dégradations progressives
- Optimisations proactives

**Incident Response** (alerte déclenchée) :
- Diagnostic rapide avec requêtes temps réel
- Identifier la requête/table/verrou problématique
- Action corrective immédiate

### 7.3. Logging : Configuration Optimale

Dans `postgresql.conf` :

```conf
# Activer le logging des requêtes lentes
log_min_duration_statement = 200  # Log toute requête > 200ms

# Format de log structuré
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Durée des verrous
log_lock_waits = on  
deadlock_timeout = 1s  

# Checkpoints
log_checkpoints = on

# Connexions/Déconnexions (peut être verbeux)
log_connections = off  
log_disconnections = off  

# Auto-explain pour les requêtes lentes (extension)
shared_preload_libraries = 'pg_stat_statements,auto_explain'  
auto_explain.log_min_duration = 1000  # EXPLAIN si > 1s  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
```

### 7.4. Rotation et Archivage des Logs

Les logs PostgreSQL peuvent rapidement devenir volumineux.

**Configuration système (logrotate)** :

```conf
/var/log/postgresql/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 postgres postgres
}
```

### 7.5. Tests de Charge et Benchmarking

Avant un déploiement majeur :

1. **Utiliser pgbench** pour simuler de la charge :

```bash
# Initialiser la base de test
pgbench -i -s 50 test_db

# Exécuter un benchmark (10 clients, 1000 transactions chacun)
pgbench -c 10 -t 1000 test_db
```

2. **Comparer les métriques avant/après** :
   - Transactions par seconde (TPS)
   - Latence moyenne
   - Cache hit ratio

### 7.6. Documentation et Runbooks

**Créez des runbooks** pour les scénarios courants :

**Exemple : Runbook "Application Lente"**

```
1. Vérifier cache hit ratio (requête section 2.3)
   - Si < 90% : Augmenter shared_buffers
2. Identifier slow queries (requête section 3.3)
   - Analyser avec EXPLAIN ANALYZE
   - Créer index si nécessaire
3. Vérifier connexions (section 4.1)
   - Si > 80% : Activer PgBouncer
4. Vérifier verrous (tutoriel locks)
   - Tuer transaction bloquante si besoin
```

### 7.7. Revue de Performance Périodique

**Mensuel** :

- Audit des slow queries (top 50)
- Audit des index inutilisés
- Audit du bloat
- Revue des métriques de croissance

**Trimestriel** :

- Revue de l'architecture (partitionnement, réplication)
- Tests de DR (Disaster Recovery)
- Formation de l'équipe sur nouveaux outils

---

## Résumé des Points Clés

### Cache Hit Ratio

- ✅ **Objectif** : > 95%  
- ✅ **Requête principale** : Section 2.3 (cache hit ratio global)  
- ✅ **Action si < 95%** : Augmenter shared_buffers, optimiser requêtes  
- ✅ **PostgreSQL 18** : Nouvelles statistiques I/O par backend

### Slow Queries

- ✅ **Outil indispensable** : pg_stat_statements  
- ✅ **Requête principale** : Section 3.3 (top 20 slow queries)  
- ✅ **Diagnostic** : EXPLAIN ANALYZE  
- ✅ **Solution** : Index, réécriture SQL, partitionnement  
- ✅ **PostgreSQL 18** : Optimisations automatiques (skip scan, OR → ANY)

### Métriques Complémentaires

- ✅ Connexions actives (section 4.1)  
- ✅ Taille des tables (section 4.2)  
- ✅ Checkpoints et WAL (section 4.3)  
- ✅ XID wraparound risk (section 4.4)  
- ✅ PostgreSQL 18 : Statistiques VACUUM/ANALYZE enrichies

### Automatisation

- ✅ Prometheus + Grafana pour monitoring temps réel  
- ✅ Alertes sur seuils critiques  
- ✅ Vues personnalisées pour simplifier  
- ✅ Runbooks pour incident response

---

## Ressources Complémentaires

### Documentation Officielle PostgreSQL

- [Monitoring Database Activity](https://www.postgresql.org/docs/18/monitoring.html)  
- [pg_stat_statements](https://www.postgresql.org/docs/18/pgstatstatements.html)  
- [EXPLAIN Documentation](https://www.postgresql.org/docs/18/sql-explain.html)

### Outils de Monitoring

- **pg_stat_statements** : Extension officielle  
- **Prometheus + postgres_exporter** : Monitoring temps réel  
- **Grafana** : Dashboards visuels (Dashboard ID 9628 recommandé)  
- **pgBadger** : Analyse de logs  
- **pgAdmin 4** : Interface graphique avec monitoring intégré  
- **Datadog, New Relic** : Solutions commerciales

### Articles et Guides

- [Explain Analyze Visualizer](https://explain.dalibo.com/) : Visualiser les plans d'exécution  
- [PostgreSQL Explain](https://www.postgresql.org/docs/current/using-explain.html)
- Blog Percona PostgreSQL : Articles d'experts
- 2ndQuadrant Blog : Astuces avancées

### Communautés

- Mailing list : pgsql-performance@postgresql.org
- Reddit : r/PostgreSQL
- Discord PostgreSQL (communauté francophone)
- Stack Overflow : Tag `postgresql-performance`

---


⏭️ [Requêtes d'analyse (statistiques, tables sizes)](/annexes/requetes-sql-reference/03-requetes-analyse.md)
