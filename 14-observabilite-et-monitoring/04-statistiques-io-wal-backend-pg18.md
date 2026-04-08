🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4. Nouveauté PG 18 : Statistiques I/O et WAL par backend

## Introduction

PostgreSQL 18 introduit une fonctionnalité révolutionnaire pour le diagnostic des performances : la possibilité de suivre **les opérations I/O (entrées/sorties) et la génération de WAL (Write-Ahead Log) au niveau de chaque processus backend** individuel.

Cette nouveauté transforme radicalement votre capacité à identifier et résoudre les problèmes de performance. Mais avant de plonger dans les détails, commençons par comprendre les concepts fondamentaux.

---

## Concepts Fondamentaux : Pour les Débutants

### Qu'est-ce qu'un "backend" ?

En termes simples, un **backend** (ou processus backend) est un processus du système d'exploitation dédié à gérer **une seule connexion cliente** à PostgreSQL.

**Analogie** : Imaginez une banque. Chaque guichetier est un "backend" qui s'occupe d'un client à la fois. Si vous vous connectez à PostgreSQL, un processus backend est créé spécialement pour vous et reste actif pendant toute la durée de votre connexion.

**Points clés** :
- 1 connexion = 1 backend
- Chaque backend a son propre Process ID (PID) dans le système d'exploitation
- Les backends exécutent vos requêtes SQL
- Ils gèrent la mémoire, l'I/O, et interagissent avec le disque

**Visualisation** :
```
Client 1 ─────► Backend (PID: 12345) ─┐  
Client 2 ─────► Backend (PID: 12346) ─┤  
Client 3 ─────► Backend (PID: 12347) ─┼──► PostgreSQL  
Client 4 ─────► Backend (PID: 12348) ─┤      (Data + WAL)  
Client 5 ─────► Backend (PID: 12349) ─┘  
```

### Qu'est-ce que l'I/O (Input/Output) ?

**I/O** signifie "Entrées/Sorties" et représente toutes les opérations de **lecture et écriture sur le disque**.

**Pourquoi c'est important** : Le disque est **beaucoup plus lent** que la RAM (mémoire). Une requête qui lit beaucoup de données sur le disque sera lente.

**Types d'I/O dans PostgreSQL** :
1. **Lectures (Read I/O)** : Charger des données depuis le disque vers la mémoire  
2. **Écritures (Write I/O)** : Sauvegarder des données de la mémoire vers le disque  
3. **Cache hits** : Données trouvées en mémoire (pas d'I/O disque = rapide !)  
4. **Cache misses** : Données non trouvées en mémoire (I/O disque requis = lent)

**Analogie** :
- RAM = Bureau (accès rapide)
- Disque = Armoire d'archives (accès lent)
- I/O = Aller-retour entre votre bureau et l'armoire

### Qu'est-ce que le WAL (Write-Ahead Log) ?

Le **WAL** (aussi appelé journal de transactions) est un mécanisme de sécurité qui garantit que vos données ne seront jamais perdues, même en cas de crash.

**Principe** : Avant de modifier une donnée sur le disque, PostgreSQL écrit d'abord **ce qu'il va faire** dans le WAL. Si le serveur crashe, PostgreSQL peut "rejouer" le WAL pour récupérer les changements.

**Analogie** : Le WAL est comme le journal de bord d'un navire. Avant de faire une action importante, le capitaine l'écrit dans son journal. Si quelque chose tourne mal, il peut consulter le journal pour savoir ce qui s'est passé et reprendre là où il s'était arrêté.

**Ce qui génère du WAL** :
- `INSERT`, `UPDATE`, `DELETE` : Modifications de données  
- `CREATE INDEX`, `ALTER TABLE` : Modifications de structure  
- `COMMIT` : Validation de transactions
- Checkpoints : Synchronisation périodique

**Pourquoi surveiller le WAL** :
- Trop de WAL = beaucoup d'écritures = charge élevée
- Peut saturer le disque ou le réseau (réplication)
- Indicateur de l'intensité des modifications

---

## Le Problème Avant PostgreSQL 18

### Diagnostic limité et frustrant

Avant PostgreSQL 18, vous pouviez voir des **statistiques globales** d'I/O et de WAL, mais pas au niveau de chaque connexion/backend individuel.

**Scénario typique** :
```
Vous : "Le serveur est lent !"  
Monitoring : "Il y a 200 GO d'I/O en cours"  
Vous : "OK... mais QUELLE requête/connexion cause ça ?"  
Monitoring : "🤷 Aucune idée"  
```

### Ce que vous pouviez voir

**Statistiques globales** (dans `pg_stat_database`, `pg_stat_bgwriter`) :
```sql
SELECT
    blks_read,      -- Total de blocs lus depuis le disque (TOUTES connexions)
    blks_hit        -- Total de blocs trouvés en cache (TOUTES connexions)
FROM pg_stat_database  
WHERE datname = 'ma_base';  

  blks_read  |   blks_hit
-------------+--------------
  45234789   | 923847234
```

**Problème** : Vous voyez l'activité totale, mais impossible d'identifier le coupable !

### Ce que vous NE pouviez PAS voir

- ❌ Quelle connexion (backend) génère le plus d'I/O ?  
- ❌ Quel utilisateur ou application est responsable ?  
- ❌ Quelle requête spécifique cause une génération excessive de WAL ?  
- ❌ Est-ce un backend unique qui pose problème ou plusieurs ?

**Résultat** : Des heures de recherche pour trouver la source d'un problème de performance.

---

## La Solution PostgreSQL 18 : Statistiques par Backend

### Vue d'ensemble de la nouveauté

PostgreSQL 18 introduit de **nouvelles colonnes dans pg_stat_activity** qui exposent :
- Les statistiques d'I/O pour chaque backend
- La génération de WAL pour chaque backend
- Le tout en temps réel !

### Les nouvelles vues et colonnes

#### Extension de pg_stat_activity

`pg_stat_activity` est enrichie avec de nouvelles colonnes d'I/O et WAL :

**Colonnes I/O (nouvelles dans PG 18)** :

| Colonne | Description |
|---------|-------------|
| `io_blks_read` | Nombre de blocs lus depuis le disque par ce backend |
| `io_blks_written` | Nombre de blocs écrits sur le disque par ce backend |
| `io_blks_hit` | Nombre de blocs trouvés dans le cache (shared buffers) |
| `io_read_time` | Temps total passé à lire depuis le disque (en millisecondes) |
| `io_write_time` | Temps total passé à écrire sur le disque (en millisecondes) |

**Colonnes WAL (nouvelles dans PG 18)** :

| Colonne | Description |
|---------|-------------|
| `wal_records` | Nombre d'enregistrements WAL générés |
| `wal_bytes` | Nombre d'octets WAL générés |
| `wal_fpi` | Nombre de Full Page Images WAL générés |

**Note** : Ces statistiques sont **cumulatives** depuis le début de la connexion du backend.

#### Filtrage par type de backend dans pg_stat_io

Les statistiques I/O par backend sont disponibles directement dans la vue `pg_stat_io` existante, en filtrant sur la colonne `backend_type`. Il n'y a pas de vue séparée — toutes les informations sont consolidées dans `pg_stat_io`.

---

## Utiliser les Nouvelles Statistiques : Exemples Pratiques

### Exemple 1 : Identifier le backend qui génère le plus d'I/O

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    io_blks_read,
    io_blks_written,
    io_blks_hit,
    ROUND(100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0), 2) AS cache_hit_ratio,
    query
FROM pg_stat_activity  
WHERE state = 'active'  
  AND io_blks_read + io_blks_written > 0
ORDER BY (io_blks_read + io_blks_written) DESC  
LIMIT 10;  
```

**Interprétation** :
- `io_blks_read` élevé : Ce backend lit beaucoup depuis le disque (potentiellement lent)  
- `io_blks_written` élevé : Ce backend écrit beaucoup (INSERT/UPDATE/DELETE massif)  
- `cache_hit_ratio` faible : Les données ne sont pas en cache, beaucoup d'I/O disque
- Vous voyez aussi la `query` exacte qui cause ce comportement !

**Résultat exemple** :
```
  pid  | usename |  application_name  |  state  | io_blks_read | io_blks_written | io_blks_hit | cache_hit_ratio |              query
-------+---------+--------------------+---------+--------------+-----------------+-------------+-----------------+--------------------------------
 12345 | appuser | MyApp              | active  |      845320  |          12450  |    12340    |           1.44  | SELECT * FROM huge_table WHERE...
 12348 | etl     | DataImport         | active  |        1230  |         945000  |      450    |          26.79  | INSERT INTO logs VALUES...
```

### Exemple 2 : Trouver les backends qui génèrent le plus de WAL

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    wal_records,
    pg_size_pretty(wal_bytes) AS wal_size,
    wal_fpi,
    query_start,
    NOW() - query_start AS query_duration,
    LEFT(query, 100) AS query_preview
FROM pg_stat_activity  
WHERE wal_bytes > 0  
ORDER BY wal_bytes DESC  
LIMIT 10;  
```

**Interprétation** :
- `wal_bytes` élevé : Ce backend modifie beaucoup de données  
- `wal_fpi` élevé : Beaucoup de Full Page Images (après un checkpoint)  
- `query_duration` long : La requête tourne depuis longtemps
- Corrélation entre `wal_bytes` et durée : Requête lente ET gourmande

**Cas d'usage** :
- Identifier les ETL (Extract-Transform-Load) qui surchargent le disque
- Détecter les imports massifs mal optimisés
- Trouver les transactions longues qui bloquent le VACUUM

### Exemple 3 : Comparer les performances I/O de plusieurs backends

```sql
SELECT
    usename,
    application_name,
    COUNT(*) AS nb_connections,
    SUM(io_blks_read) AS total_reads,
    SUM(io_blks_written) AS total_writes,
    SUM(io_blks_hit) AS total_cache_hits,
    ROUND(AVG(100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0)), 2) AS avg_cache_hit_ratio,
    pg_size_pretty(SUM(wal_bytes)) AS total_wal
FROM pg_stat_activity  
WHERE state != 'idle'  
GROUP BY usename, application_name  
ORDER BY total_reads + total_writes DESC;  
```

**Interprétation** :
- Vue agrégée par utilisateur ou application
- Identifie quelle application consomme le plus de ressources I/O
- Compare le `cache_hit_ratio` entre applications
- Montre la contribution au WAL par application

**Résultat exemple** :
```
 usename |  application_name  | nb_connections | total_reads | total_writes | total_cache_hits | avg_cache_hit_ratio | total_wal
---------+--------------------+----------------+-------------+--------------+------------------+--------------------+-----------
 etl     | NightlyImport      |              5 |     4532100 |      2340100 |           234500 |              4.92  | 234 GB
 webapp  | MyWebApp           |             45 |      123400 |        23400 |         23400000 |             99.47  | 1245 MB
 analyst | Tableau            |              3 |     1234500 |          450 |          4500000 |             78.45  | 45 MB
```

**Insight** : L'ETL `NightlyImport` a un cache hit ratio catastrophique (4.92%) et génère énormément de WAL !

### Exemple 4 : Surveiller l'I/O en temps réel

```sql
-- Snapshot 1 : Capturer l'état initial
CREATE TEMP TABLE io_snapshot_1 AS  
SELECT  
    pid,
    io_blks_read,
    io_blks_written,
    wal_bytes,
    query
FROM pg_stat_activity  
WHERE state = 'active';  

-- Attendre quelques secondes
SELECT pg_sleep(5);

-- Snapshot 2 : Capturer l'état après 5 secondes
CREATE TEMP TABLE io_snapshot_2 AS  
SELECT  
    pid,
    io_blks_read,
    io_blks_written,
    wal_bytes,
    query
FROM pg_stat_activity  
WHERE state = 'active';  

-- Calculer le delta (l'activité pendant ces 5 secondes)
SELECT
    s2.pid,
    s2.query,
    (s2.io_blks_read - COALESCE(s1.io_blks_read, 0)) AS reads_per_5sec,
    (s2.io_blks_written - COALESCE(s1.io_blks_written, 0)) AS writes_per_5sec,
    pg_size_pretty((s2.wal_bytes - COALESCE(s1.wal_bytes, 0))) AS wal_per_5sec
FROM io_snapshot_2 s2  
LEFT JOIN io_snapshot_1 s1 USING (pid)  
WHERE (s2.io_blks_read - COALESCE(s1.io_blks_read, 0)) > 100  
   OR (s2.io_blks_written - COALESCE(s1.io_blks_written, 0)) > 100
ORDER BY reads_per_5sec + writes_per_5sec DESC;
```

**Utilité** :
- Mesure l'activité I/O **instantanée** (sur 5 secondes)
- Identifie les backends qui consomment le plus en ce moment précis
- Permet de suivre l'évolution dans le temps

### Exemple 5 : Corrélation avec le temps CPU et les locks

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    io_blks_read + io_blks_written AS total_io_blocks,
    pg_size_pretty(wal_bytes) AS wal_generated,
    NOW() - query_start AS duration,
    NOW() - state_change AS time_in_state,
    wait_event_type,
    wait_event,
    LEFT(query, 80) AS query_preview
FROM pg_stat_activity  
WHERE state = 'active'  
  AND (io_blks_read + io_blks_written > 1000 OR wal_bytes > 1048576)  -- > 1 MB WAL
ORDER BY total_io_blocks DESC;
```

**Interprétation** :
- `wait_event_type = 'IO'` : Le backend attend des I/O (lent !)  
- `wait_event = 'DataFileRead'` : Lecture de fichiers de données  
- `duration` long + `total_io_blocks` élevé : Requête lente à cause des I/O
- Corrélation entre WAL et attentes de locks

---

## Scénarios de Diagnostic Avancés

### Scénario 1 : Le serveur est lent, qui est le coupable ?

**Symptôme** : L'ensemble du serveur ralentit. Les utilisateurs se plaignent.

**Diagnostic avec PostgreSQL 18** :

```sql
-- Étape 1 : Vue d'ensemble des backends actifs
SELECT
    pid,
    usename,
    application_name,
    NOW() - query_start AS duration,
    io_blks_read,
    io_blks_written,
    pg_size_pretty(wal_bytes) AS wal,
    state,
    wait_event_type,
    LEFT(query, 60) AS query
FROM pg_stat_activity  
WHERE state != 'idle'  
ORDER BY io_blks_read + io_blks_written DESC  
LIMIT 20;  
```

**Analyse** : Supposons que vous trouvez :
```
  pid  | usename |  duration  | io_blks_read | io_blks_written |   wal   |  state  | wait_event_type |           query
-------+---------+------------+--------------+-----------------+---------+---------+-----------------+--------------------------
 14532 | appuser | 00:45:23   |    23450000  |         450000  | 345 GB  | active  | IO              | DELETE FROM audit_logs WHERE...
```

**Conclusion** : Le backend 14532 effectue une suppression massive qui :
- Lit 23 millions de blocs depuis le disque
- Génère 345 GB de WAL
- Est en attente I/O (goulot d'étranglement disque)
- Tourne depuis 45 minutes

**Actions** :
1. Évaluer si cette requête est légitime ou peut être interrompue  
2. Si légitime : optimiser (ajouter un index, découper en batches)  
3. Si bloquante : tuer le backend : `SELECT pg_terminate_backend(14532);`

### Scénario 2 : Saturation du WAL et réplication qui prend du retard

**Symptôme** : Les réplicas (standby servers) sont en retard. Le WAL s'accumule.

**Diagnostic avec PostgreSQL 18** :

```sql
-- Identifier les backends qui génèrent le plus de WAL
SELECT
    pid,
    usename,
    application_name,
    pg_size_pretty(wal_bytes) AS wal_generated,
    wal_records,
    NOW() - backend_start AS connection_age,
    NOW() - query_start AS query_age,
    state,
    LEFT(query, 100) AS query
FROM pg_stat_activity  
WHERE wal_bytes > 0  
ORDER BY wal_bytes DESC  
LIMIT 10;  
```

**Analyse** : Supposons que vous trouvez :
```
  pid  | usename |  application_name  | wal_generated | wal_records | connection_age | query_age  |  state  |           query
-------+---------+--------------------+---------------+-------------+----------------+------------+---------+--------------------------
 15234 | etl     | DataMigration      | 1.2 TB        |  234500000  | 08:30:00       | 08:29:45   | active  | INSERT INTO target_table SELECT...
```

**Conclusion** : Une migration de données génère 1,2 TB de WAL en continu !

**Actions** :
1. **Optimiser** : Utiliser `COPY` au lieu d'`INSERT` (10× plus rapide, moins de WAL)  
2. **Throttler** : Ajouter des pauses entre les batches  
3. **Paralléliser** : Répartir sur plusieurs transactions pour permettre au WAL d'être répliqué  
4. **Temporaire** : Augmenter `max_wal_size` pendant la migration

### Scénario 3 : Ratio de cache anormalement bas

**Symptôme** : Les performances se dégradent progressivement. Le monitoring montre un cache hit ratio global qui chute.

**Diagnostic avec PostgreSQL 18** :

```sql
SELECT
    usename,
    application_name,
    COUNT(*) AS connections,
    SUM(io_blks_read) AS total_disk_reads,
    SUM(io_blks_hit) AS total_cache_hits,
    ROUND(100.0 * SUM(io_blks_hit) / NULLIF(SUM(io_blks_hit) + SUM(io_blks_read), 0), 2) AS cache_hit_ratio,
    ROUND(AVG(io_blks_read)::numeric, 0) AS avg_disk_reads_per_conn
FROM pg_stat_activity  
WHERE state != 'idle'  
  AND backend_type = 'client backend'
GROUP BY usename, application_name  
HAVING SUM(io_blks_read) > 0  
ORDER BY cache_hit_ratio ASC;  
```

**Analyse** : Supposons que vous trouvez :
```
 usename |  application_name  | connections | total_disk_reads | total_cache_hits | cache_hit_ratio | avg_disk_reads_per_conn
---------+--------------------+-------------+------------------+------------------+-----------------+------------------------
 analyst | BusinessIntel      |           8 |         45600000 |           234500 |            0.51 |              5700000
 webapp  | MyApp              |          45 |          1234000 |        123400000 |           99.01 |                27422
```

**Conclusion** : L'application `BusinessIntel` a un cache hit ratio catastrophique (0,51%) !

**Causes possibles** :
- Tables trop grandes pour tenir en cache
- Requêtes qui scannent toute la table
- Pas d'index appropriés

**Actions** :
1. Analyser les requêtes de `BusinessIntel` (via `pg_stat_statements`)  
2. Vérifier les plans d'exécution (`EXPLAIN ANALYZE`)  
3. Ajouter des index si nécessaire  
4. Envisager d'augmenter `shared_buffers` si le serveur a de la RAM disponible

### Scénario 4 : Temps d'I/O excessif

**Symptôme** : Les requêtes sont lentes, mais le CPU est faible. Problème d'I/O ?

**Diagnostic avec PostgreSQL 18** :

```sql
SELECT
    pid,
    usename,
    application_name,
    ROUND(io_read_time::numeric, 2) AS read_time_ms,
    ROUND(io_write_time::numeric, 2) AS write_time_ms,
    ROUND((io_read_time + io_write_time)::numeric, 2) AS total_io_time_ms,
    io_blks_read,
    io_blks_written,
    ROUND((io_read_time / NULLIF(io_blks_read, 0))::numeric, 2) AS avg_read_latency_ms,
    LEFT(query, 80) AS query
FROM pg_stat_activity  
WHERE state = 'active'  
  AND (io_read_time > 0 OR io_write_time > 0)
ORDER BY total_io_time_ms DESC  
LIMIT 10;  
```

**Interprétation** :
- `total_io_time_ms` élevé : Beaucoup de temps passé à attendre les I/O  
- `avg_read_latency_ms` élevé : Disque lent ou saturé
- Comparaison entre différents backends : Certains sont-ils plus affectés ?

**Analyse** : Si `avg_read_latency_ms > 10 ms`, vous avez un problème de disque !

**Actions** :
1. Vérifier la santé du disque (SMART, I/O wait système)  
2. Vérifier si d'autres processus saturent le disque  
3. Envisager un upgrade disque (SSD, NVMe)  
4. Optimiser les requêtes pour réduire les I/O

---

## Intégration avec les Outils de Monitoring

### Grafana + Prometheus

Exposez ces nouvelles métriques pour les visualiser dans Grafana :

```sql
-- Métriques à exporter pour Prometheus (postgres_exporter)

-- 1. I/O par application
SELECT
    application_name,
    SUM(io_blks_read) AS total_blks_read,
    SUM(io_blks_written) AS total_blks_written,
    SUM(io_blks_hit) AS total_blks_hit
FROM pg_stat_activity  
WHERE backend_type = 'client backend'  
GROUP BY application_name;  

-- 2. WAL par application
SELECT
    application_name,
    SUM(wal_bytes) AS total_wal_bytes,
    SUM(wal_records) AS total_wal_records
FROM pg_stat_activity  
WHERE backend_type = 'client backend'  
GROUP BY application_name;  

-- 3. Top 10 des backends par I/O
SELECT
    pid,
    usename,
    (io_blks_read + io_blks_written) AS total_io
FROM pg_stat_activity  
WHERE state = 'active'  
ORDER BY total_io DESC  
LIMIT 10;  
```

**Dashboards Grafana recommandés** :
- Graphique de l'I/O par backend au fil du temps
- Heatmap de la génération WAL par heure
- Alerte si un backend dépasse 1 GB de WAL en 5 minutes

### Scripts de Monitoring Personnalisés

Voici un script shell pour surveiller l'I/O en continu :

```bash
#!/bin/bash
# monitor_backend_io.sh

THRESHOLD_IO_BLOCKS=100000  
THRESHOLD_WAL_GB=5  

while true; do
    psql -U postgres -d ma_base -t -A -c "
    SELECT
        pid,
        usename,
        io_blks_read + io_blks_written AS total_io,
        ROUND(wal_bytes / 1073741824.0, 2) AS wal_gb
    FROM pg_stat_activity
    WHERE state = 'active'
      AND (io_blks_read + io_blks_written > $THRESHOLD_IO_BLOCKS
           OR wal_bytes > $THRESHOLD_WAL_GB * 1073741824)
    " | while IFS='|' read pid user io wal; do
        if [ ! -z "$pid" ]; then
            echo "[ALERT] Backend $pid (user: $user) - IO: $io blocks, WAL: ${wal}GB"
            # Envoyer une notification (email, Slack, etc.)
        fi
    done

    sleep 30
done
```

### Intégration avec pg_stat_statements

Combinez les statistiques par backend avec `pg_stat_statements` pour une analyse complète :

```sql
-- Trouver les requêtes qui génèrent le plus de WAL
SELECT
    pss.queryid,
    LEFT(pss.query, 100) AS query_preview,
    pss.calls,
    pg_size_pretty(pss.total_exec_time::bigint * 1000) AS total_time,
    -- Corrélation avec les backends actifs
    (SELECT SUM(wal_bytes)
     FROM pg_stat_activity
     WHERE query LIKE '%' || SUBSTRING(pss.query, 1, 30) || '%') AS estimated_wal
FROM pg_stat_statements pss  
WHERE pss.calls > 100  
ORDER BY estimated_wal DESC NULLS LAST  
LIMIT 10;  
```

---

## Cas d'Usage Pratiques

### Cas 1 : Audit de consommation de ressources par application

```sql
-- Rapport hebdomadaire : Quelle application consomme le plus de ressources ?
CREATE MATERIALIZED VIEW weekly_app_resource_usage AS  
SELECT  
    application_name,
    usename,
    COUNT(DISTINCT pid) AS avg_concurrent_connections,
    pg_size_pretty(SUM(io_blks_read) * 8192) AS total_disk_reads,
    pg_size_pretty(SUM(io_blks_written) * 8192) AS total_disk_writes,
    pg_size_pretty(SUM(wal_bytes)) AS total_wal_generated,
    ROUND(AVG(100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0)), 2) AS avg_cache_hit_ratio
FROM pg_stat_activity  
WHERE backend_type = 'client backend'  
  AND backend_start > NOW() - INTERVAL '7 days'
GROUP BY application_name, usename  
ORDER BY SUM(wal_bytes) DESC;  

-- Rafraîchir toutes les semaines
-- SELECT refresh_materialized_view('weekly_app_resource_usage');
```

**Utilité** :
- Facturation interne par application
- Identification des applications à optimiser
- Planification de capacité

### Cas 2 : Détection d'anomalies en temps réel

```sql
-- Détecter les backends avec une activité I/O ou WAL anormale
WITH backend_stats AS (
    SELECT
        pid,
        usename,
        application_name,
        io_blks_read + io_blks_written AS total_io,
        wal_bytes,
        query
    FROM pg_stat_activity
    WHERE state = 'active'
),
stats_summary AS (
    SELECT
        AVG(total_io) AS avg_io,
        STDDEV(total_io) AS stddev_io,
        AVG(wal_bytes) AS avg_wal,
        STDDEV(wal_bytes) AS stddev_wal
    FROM backend_stats
)
SELECT
    bs.pid,
    bs.usename,
    bs.application_name,
    bs.total_io,
    ROUND((bs.total_io - ss.avg_io) / NULLIF(ss.stddev_io, 0), 2) AS io_zscore,
    pg_size_pretty(bs.wal_bytes) AS wal,
    ROUND((bs.wal_bytes - ss.avg_wal) / NULLIF(ss.stddev_wal, 0), 2) AS wal_zscore,
    LEFT(bs.query, 80) AS query
FROM backend_stats bs, stats_summary ss  
WHERE (bs.total_io - ss.avg_io) / NULLIF(ss.stddev_io, 0) > 3  -- 3 écarts-types  
   OR (bs.wal_bytes - ss.avg_wal) / NULLIF(ss.stddev_wal, 0) > 3
ORDER BY io_zscore DESC;
```

**Explication** : Utilise le **Z-score** (écart à la moyenne) pour détecter les backends avec une activité statistiquement anormale.

### Cas 3 : Optimisation de l'autovacuum

```sql
-- Identifier si l'autovacuum génère beaucoup d'I/O
SELECT
    pid,
    backend_type,
    io_blks_read,
    io_blks_written,
    io_read_time,
    io_write_time,
    NOW() - backend_start AS duration,
    LEFT(query, 80) AS query
FROM pg_stat_activity  
WHERE backend_type = 'autovacuum worker'  
  AND state = 'active'
ORDER BY io_blks_read + io_blks_written DESC;
```

**Utilité** :
- Vérifier si l'autovacuum consomme trop d'I/O (impacte les requêtes utilisateurs)
- Ajuster `autovacuum_vacuum_cost_delay` si nécessaire
- Détecter les autovacuums qui tournent trop longtemps

---

## Bonnes Pratiques et Recommandations

### 1. Définir des seuils d'alerte

Configurez des alertes basées sur des seuils adaptés à votre contexte :

**Seuils suggérés (à ajuster selon votre charge)** :

| Métrique | Seuil Warning | Seuil Critical | Action |
|----------|---------------|----------------|--------|
| `io_blks_read + io_blks_written` | > 500 000 | > 2 000 000 | Analyser la requête |
| `wal_bytes` | > 1 GB | > 10 GB | Vérifier la transaction |
| `cache_hit_ratio` | < 95% | < 90% | Optimiser les index |
| `io_read_time + io_write_time` | > 60 000 ms | > 300 000 ms | Vérifier le disque |

### 2. Surveiller en continu

Créez une vue dédiée pour le monitoring :

```sql
CREATE OR REPLACE VIEW v_backend_io_monitoring AS  
SELECT  
    pid,
    usename,
    application_name,
    state,
    NOW() - backend_start AS connection_age,
    NOW() - query_start AS query_age,
    io_blks_read,
    io_blks_written,
    io_blks_hit,
    ROUND(100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0), 2) AS cache_hit_ratio,
    pg_size_pretty(wal_bytes) AS wal_size,
    ROUND(io_read_time::numeric, 2) AS read_time_ms,
    ROUND(io_write_time::numeric, 2) AS write_time_ms,
    wait_event_type,
    wait_event,
    LEFT(query, 100) AS query_preview
FROM pg_stat_activity  
WHERE backend_type = 'client backend'  
  AND state != 'idle';

-- Utilisation
SELECT * FROM v_backend_io_monitoring  
WHERE cache_hit_ratio < 90 OR wal_size > '1 GB'  
ORDER BY io_blks_read + io_blks_written DESC;  
```

### 3. Corréler avec d'autres métriques

Ne regardez jamais les statistiques I/O/WAL de manière isolée. Combinez avec :

```sql
-- Vue complète : I/O + Locks + CPU
SELECT
    a.pid,
    a.usename,
    a.application_name,
    a.state,
    a.io_blks_read + a.io_blks_written AS total_io,
    pg_size_pretty(a.wal_bytes) AS wal,
    a.wait_event_type,
    a.wait_event,
    -- Locks bloquants
    (SELECT COUNT(*)
     FROM pg_locks l
     WHERE l.pid = a.pid AND NOT l.granted) AS blocked_locks,
    -- Locks tenus
    (SELECT COUNT(*)
     FROM pg_locks l
     WHERE l.pid = a.pid AND l.granted) AS held_locks,
    LEFT(a.query, 80) AS query
FROM pg_stat_activity a  
WHERE a.state = 'active'  
ORDER BY total_io DESC;  
```

### 4. Archiver les statistiques historiques

Les statistiques dans `pg_stat_activity` sont en temps réel et éphémères. Pour l'analyse historique :

```sql
-- Table d'historique
CREATE TABLE backend_io_history (
    captured_at timestamp DEFAULT NOW(),
    pid integer,
    usename text,
    application_name text,
    io_blks_read bigint,
    io_blks_written bigint,
    wal_bytes bigint,
    query text
);

-- Job périodique (via pg_cron ou cron système)
INSERT INTO backend_io_history (pid, usename, application_name, io_blks_read, io_blks_written, wal_bytes, query)  
SELECT pid, usename, application_name, io_blks_read, io_blks_written, wal_bytes, query  
FROM pg_stat_activity  
WHERE state = 'active' AND (io_blks_read + io_blks_written > 1000 OR wal_bytes > 1048576);  
```

### 5. Documenter les patterns normaux

Chaque application a ses propres patterns d'I/O et WAL. Documentez ce qui est "normal" :

- ETL nocturne : 100 GB WAL / nuit = normal
- Application web : < 1 MB WAL / requête = normal
- Rapports analytiques : 1M blocs lus = normal

Cela vous permettra de détecter plus facilement les anomalies.

---

## Limites et Considérations

### Ce que ces statistiques NE montrent PAS

1. **Détails des fichiers** : Vous ne savez pas quels fichiers/tables sont lus/écrits  
2. **Décomposition par requête** : Si un backend exécute plusieurs requêtes, les stats sont cumulatives  
3. **I/O système** : Uniquement l'I/O PostgreSQL, pas l'I/O OS global  
4. **Historique** : Statistiques depuis le début de la connexion, pas de breakdown temporel

### Impact sur les performances

L'activation de ces statistiques a un **impact minimal** :
- Overhead : < 2% dans la plupart des cas
- Déjà activé par défaut dans PostgreSQL 18
- Peut être désactivé si vraiment nécessaire via `track_io_timing`

### Compatibilité

- **Nouvelle fonctionnalité PostgreSQL 18** : N'existe pas dans les versions antérieures
- Migration depuis une version plus ancienne : Les colonnes apparaîtront automatiquement
- Scripts existants : Non impactés (nouvelles colonnes ajoutées)

---

## Requêtes Prêtes à l'Emploi

### Dashboard de santé complet

```sql
-- Vue d'ensemble : Backends actifs avec I/O et WAL
SELECT
    ROW_NUMBER() OVER (ORDER BY io_blks_read + io_blks_written DESC) AS rank,
    pid,
    usename,
    application_name,
    state,
    ROUND(EXTRACT(EPOCH FROM (NOW() - query_start)) / 60, 1) AS minutes_running,
    io_blks_read AS disk_reads,
    io_blks_written AS disk_writes,
    io_blks_hit AS cache_hits,
    ROUND(100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0), 1) AS cache_hit_pct,
    pg_size_pretty(wal_bytes) AS wal_generated,
    wait_event_type || ': ' || COALESCE(wait_event, 'none') AS wait_status,
    LEFT(query, 60) AS query_preview
FROM pg_stat_activity  
WHERE state != 'idle'  
  AND backend_type = 'client backend'
ORDER BY io_blks_read + io_blks_written DESC  
LIMIT 20;  
```

### Top 5 des gourmands en ressources

```sql
-- Les 5 backends qui consomment le plus de chaque ressource
(SELECT 'TOP I/O READS' AS category, pid, usename, application_name,
        io_blks_read AS value, LEFT(query, 50) AS query
 FROM pg_stat_activity WHERE state = 'active' ORDER BY io_blks_read DESC LIMIT 5)
UNION ALL
(SELECT 'TOP I/O WRITES', pid, usename, application_name,
        io_blks_written, LEFT(query, 50)
 FROM pg_stat_activity WHERE state = 'active' ORDER BY io_blks_written DESC LIMIT 5)
UNION ALL
(SELECT 'TOP WAL GEN', pid, usename, application_name,
        wal_bytes, LEFT(query, 50)
 FROM pg_stat_activity WHERE state = 'active' ORDER BY wal_bytes DESC LIMIT 5);
```

### Alertes automatiques

```sql
-- Backends nécessitant une attention immédiate
SELECT
    'ALERT' AS priority,
    pid,
    usename,
    application_name,
    CASE
        WHEN io_blks_read > 1000000 THEN 'Excessive disk reads (>1M blocks)'
        WHEN io_blks_written > 1000000 THEN 'Excessive disk writes (>1M blocks)'
        WHEN wal_bytes > 10737418240 THEN 'Excessive WAL generation (>10GB)'
        WHEN (100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0)) < 80 THEN 'Low cache hit ratio (<80%)'
    END AS issue,
    pg_size_pretty(wal_bytes) AS wal,
    LEFT(query, 100) AS query
FROM pg_stat_activity  
WHERE state = 'active'  
  AND (io_blks_read > 1000000
       OR io_blks_written > 1000000
       OR wal_bytes > 10737418240
       OR (100.0 * io_blks_hit / NULLIF(io_blks_hit + io_blks_read, 0)) < 80);
```

---

## Conclusion

Les statistiques I/O et WAL par backend dans PostgreSQL 18 représentent une **révolution** pour le diagnostic et l'optimisation des performances.

### Avantages clés

✅ **Visibilité granulaire** : Voir exactement quel backend consomme des ressources

✅ **Diagnostic rapide** : Identifier instantanément la source d'un problème de performance

✅ **Responsabilisation** : Savoir quelle application ou utilisateur est responsable

✅ **Optimisation ciblée** : Concentrer les efforts d'optimisation sur les vrais coupables

✅ **Monitoring proactif** : Détecter les problèmes avant qu'ils n'impactent les utilisateurs

### Transformation du workflow

**Avant PostgreSQL 18** :
1. "Le serveur est lent" 😰  
2. Analyses laborieuses pendant des heures  
3. Redémarrage en dernier recours

**Avec PostgreSQL 18** :
1. "Le serveur est lent" 🔍  
2. Requête sur `pg_stat_activity` (30 secondes)  
3. Identification du backend coupable  
4. Action ciblée (optimisation ou interruption)  
5. Problème résolu ✅

### Points clés à retenir

1. **I/O** : Mesure les accès disque (lent) vs cache (rapide)  
2. **WAL** : Mesure l'intensité des modifications de données  
3. **Backend** : Un processus = une connexion = des statistiques dédiées  
4. Les colonnes `io_blks_*` et `wal_*` dans `pg_stat_activity` sont vos nouveaux meilleurs amis  
5. Combinez toujours avec d'autres vues (`pg_stat_statements`, `pg_locks`) pour un diagnostic complet

### Prochaines étapes

1. **Migrer vers PostgreSQL 18** pour bénéficier de cette fonctionnalité  
2. **Créer des vues personnalisées** adaptées à votre contexte  
3. **Intégrer dans votre monitoring** (Grafana, scripts, alertes)  
4. **Former votre équipe** à l'utilisation de ces nouvelles métriques  
5. **Documenter les seuils** propres à vos applications

### Ressources

- [Documentation PostgreSQL 18 - Monitoring](https://www.postgresql.org/docs/18/monitoring-stats.html)  
- [Release Notes PostgreSQL 18 - I/O Statistics](https://www.postgresql.org/docs/18/release-18.html)
- Blog : "Deep Dive into PostgreSQL 18 Backend I/O Statistics"

---

**💡 Astuce finale** : Créez un alias psql pour interroger rapidement ces statistiques :

```sql
-- Dans votre fichier .psqlrc
\set io_top 'SELECT pid, usename, application_name, io_blks_read + io_blks_written AS total_io, pg_size_pretty(wal_bytes) AS wal, LEFT(query, 80) FROM pg_stat_activity WHERE state = \'active\' ORDER BY total_io DESC LIMIT 10;'

-- Utilisation : \:io_top
```

**🎯 Objectif** : Faire de PostgreSQL 18 le SGBD le plus observable et diagnosticable du marché !

⏭️ [Analyse des logs : Configuration et interprétation (log_line_prefix, auto_explain)](/14-observabilite-et-monitoring/05-analyse-des-logs.md)
