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
```

```text
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

PostgreSQL 18 introduit **deux fonctions dédiées** qui exposent, pour **un backend donné** (identifié par son `pid`) :
- ses statistiques d'I/O ;
- sa génération de WAL.

Il enrichit aussi la vue **`pg_stat_io`** (qui existait depuis PG 16) avec :
- de nouvelles rangées pour l'activité **WAL** (en plus des opérations sur les relations) ;
- des colonnes en **octets** (`read_bytes`, `write_bytes`, `extend_bytes`) en complément des compteurs d'opérations ;
- la suppression de la colonne `op_bytes` (redondante puisqu'elle valait toujours `BLCKSZ`).

> ⚠️ **À noter** : PostgreSQL 18 **n'ajoute PAS** de colonnes `io_blks_read`, `io_blks_written`, `wal_records`, `wal_bytes`… à `pg_stat_activity`. Si vous voyez de telles colonnes dans un tutoriel, c'est une erreur — il s'agit en réalité des nouvelles **fonctions** décrites ci-dessous, et de la vue `pg_stat_io`.

### Les nouvelles fonctions par backend (PG 18)

| Fonction | Retourne |
|----------|----------|
| `pg_stat_get_backend_io(pid)` | Mêmes colonnes que `pg_stat_io`, mais **filtrées sur le backend** indiqué par `pid` |
| `pg_stat_get_backend_wal(pid)` | Statistiques WAL pour le backend indiqué (compteurs et octets) |
| `pg_stat_reset_backend_stats(pid)` | Remet à zéro les stats I/O et WAL d'un backend précis |

Ces fonctions retournent en réalité un **set** (`SETOF`) que l'on consomme avec `SELECT * FROM pg_stat_get_backend_io(<pid>);`.

### Les colonnes de `pg_stat_io` en PG 18

| Colonne | Description |
|---------|-------------|
| `backend_type` | Type de backend (`client backend`, `autovacuum worker`, `walwriter`…) |
| `object` | Cible : `relation`, `temp relation` ou **`wal`** (nouveau en PG 18) |
| `context` | Contexte : `normal`, `init`, `vacuum`, `bulkread`, `bulkwrite` |
| `reads` / `read_bytes` / `read_time` | Lectures : nombre, octets (**PG 18**), temps en ms |
| `writes` / `write_bytes` / `write_time` | Écritures : nombre, octets (**PG 18**), temps en ms |
| `extends` / `extend_bytes` / `extend_time` | Extensions de relation : nombre, octets (**PG 18**), temps en ms |
| `writebacks` / `writeback_time` | Demandes de writeback OS |
| `hits` | Blocs trouvés dans le cache (shared buffers) |
| `evictions` / `reuses` | Évictions et réutilisations de buffers |
| `fsyncs` / `fsync_time` | Appels `fsync()` et leur durée |
| `stats_reset` | Dernier reset des stats |

### Paramètres de timing (à connaître)

Pour que les colonnes de **temps** soient peuplées dans `pg_stat_io`, deux paramètres doivent être activés :

| Paramètre | Effet | Cible |
|-----------|-------|-------|
| `track_io_timing` | Active le timing des I/O sur les **relations** (read_time, write_time, fsync_time…) | Lignes `object` = `relation` ou `temp relation` |
| `track_wal_io_timing` (paramètre **PG 14+**) | Active le timing des I/O **WAL** | Lignes `object` = `wal` (ces lignes WAL de `pg_stat_io` sont, elles, nouvelles en PG 18) |

```ini
# postgresql.conf
track_io_timing      = on   # défaut : off  
track_wal_io_timing  = on   # défaut : off  (paramètre PG 14+ ; il peuple la ligne WAL de pg_stat_io en PG 18)  
```

> ⚠️ Ces paramètres interrogent l'horloge système à chaque opération I/O — sur certaines plateformes le surcoût peut être notable. Pour le valider, comparez `EXPLAIN (ANALYZE)` avec/sans `track_io_timing` sur une charge représentative. Sur Linux/x86 récent, le surcoût est en pratique sous 1 %.

---

## Utiliser les Nouvelles Statistiques : Exemples Pratiques

### Exemple 1 : Inspecter l'I/O d'un backend précis (PG 18)

Pour un PID spécifique, on appelle directement `pg_stat_get_backend_io(pid)` :

```sql
-- Voir le détail I/O du backend 12345 (par object/context)
SELECT
    object,
    context,
    reads,
    pg_size_pretty(read_bytes) AS read_size,
    ROUND(read_time::numeric, 1) AS read_ms,
    writes,
    pg_size_pretty(write_bytes) AS write_size,
    hits
FROM pg_stat_get_backend_io(12345);
```

Pour comparer **tous les backends actifs** entre eux, on croise `pg_stat_activity` avec la fonction via `LATERAL` :

```sql
SELECT
    a.pid,
    a.usename,
    a.application_name,
    SUM(io.reads)        AS total_reads,
    pg_size_pretty(SUM(io.read_bytes))  AS read_bytes,
    SUM(io.writes)       AS total_writes,
    pg_size_pretty(SUM(io.write_bytes)) AS write_bytes,
    SUM(io.hits)         AS total_hits,
    LEFT(a.query, 80)    AS query_preview
FROM pg_stat_activity a  
CROSS JOIN LATERAL pg_stat_get_backend_io(a.pid) io  
WHERE a.state = 'active'  
GROUP BY a.pid, a.usename, a.application_name, a.query  
HAVING SUM(io.reads + io.writes) > 0  
ORDER BY SUM(io.reads + io.writes) DESC  
LIMIT 10;  
```

**Interprétation** :
- `read_bytes` / `write_bytes` (PG 18) donnent directement le volume en octets, sans avoir à multiplier par la taille de bloc
- `hits` élevé = données en cache, très bon
- `reads` élevé sans `hits` correspondant = lectures disque massives

### Exemple 2 : Trouver les backends qui génèrent le plus de WAL (PG 18)

`pg_stat_get_backend_wal(pid)` retourne les compteurs WAL pour un backend précis :

```sql
-- WAL généré par chaque backend client actif
SELECT
    a.pid,
    a.usename,
    a.application_name,
    w.wal_records,
    pg_size_pretty(w.wal_bytes) AS wal_size,
    w.wal_fpi,
    NOW() - a.query_start AS query_duration,
    LEFT(a.query, 100) AS query_preview
FROM pg_stat_activity a  
CROSS JOIN LATERAL pg_stat_get_backend_wal(a.pid) w  
WHERE a.backend_type = 'client backend'  
  AND w.wal_bytes > 0
ORDER BY w.wal_bytes DESC  
LIMIT 10;  
```

> 💡 Les colonnes `wal_records`, `wal_bytes`, `wal_fpi` ne sont **pas** dans `pg_stat_activity` directement : elles sont retournées par la fonction `pg_stat_get_backend_wal(pid)` (PG 18+).

**Interprétation** :
- `wal_bytes` élevé : Ce backend modifie beaucoup de données
- `wal_fpi` élevé : Beaucoup de Full Page Images (typiquement après un checkpoint)
- Corrélation `wal_bytes` × durée de requête : transaction longue et lourde en écriture

**Cas d'usage** :
- Identifier les ETL (Extract-Transform-Load) qui surchargent le disque
- Détecter les imports massifs mal optimisés
- Trouver les transactions longues qui empêchent le VACUUM

### Exemple 3 : Comparer les performances I/O de plusieurs backends

```sql
WITH stats AS (
    SELECT
        a.usename,
        a.application_name,
        a.pid,
        (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid)) AS reads,
        (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid)) AS writes,
        (SELECT SUM(hits)       FROM pg_stat_get_backend_io(a.pid)) AS hits,
        (SELECT SUM(read_bytes) FROM pg_stat_get_backend_io(a.pid)) AS read_bytes,
        (SELECT wal_bytes       FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
    FROM pg_stat_activity a
    WHERE a.state IS NOT NULL AND a.state <> 'idle'
)
SELECT
    usename,
    application_name,
    COUNT(*) AS nb_connections,
    SUM(reads)  AS total_reads,
    SUM(writes) AS total_writes,
    SUM(hits)   AS total_cache_hits,
    ROUND(100.0 * SUM(hits) / NULLIF(SUM(hits) + SUM(reads), 0), 2)
        AS avg_cache_hit_ratio,
    pg_size_pretty(SUM(wal_bytes)) AS total_wal
FROM stats  
GROUP BY usename, application_name  
ORDER BY SUM(reads + writes) DESC;  
```

**Interprétation** :
- Vue agrégée par utilisateur ou application
- Identifie quelle application consomme le plus de ressources I/O
- Compare le **cache hit ratio** par application
- Montre la contribution au WAL par application

> 💡 Notez que tout passe par les fonctions `pg_stat_get_backend_io(pid)` et `pg_stat_get_backend_wal(pid)` — il n'y a **pas** de colonnes `io_blks_read` ou `wal_bytes` directement dans `pg_stat_activity` en PG 18.

### Exemple 4 : Surveiller l'I/O en temps réel

```sql
-- Snapshot 1 : Capturer l'état initial
CREATE TEMP TABLE io_snapshot_1 AS  
SELECT  
    a.pid,
    a.query,
    (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid))  AS reads,
    (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid))  AS writes,
    (SELECT wal_bytes       FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
FROM pg_stat_activity a  
WHERE a.state = 'active';  

-- Attendre quelques secondes
SELECT pg_sleep(5);

-- Snapshot 2 : Capturer l'état après 5 secondes
CREATE TEMP TABLE io_snapshot_2 AS  
SELECT  
    a.pid,
    a.query,
    (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid))  AS reads,
    (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid))  AS writes,
    (SELECT wal_bytes       FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
FROM pg_stat_activity a  
WHERE a.state = 'active';  

-- Calculer le delta (l'activité pendant ces 5 secondes)
SELECT
    s2.pid,
    s2.query,
    (s2.reads     - COALESCE(s1.reads, 0))     AS reads_per_5sec,
    (s2.writes    - COALESCE(s1.writes, 0))    AS writes_per_5sec,
    pg_size_pretty((s2.wal_bytes - COALESCE(s1.wal_bytes, 0))) AS wal_per_5sec
FROM io_snapshot_2 s2  
LEFT JOIN io_snapshot_1 s1 USING (pid)  
WHERE (s2.reads  - COALESCE(s1.reads, 0))  > 100  
   OR (s2.writes - COALESCE(s1.writes, 0)) > 100
-- on répète les expressions : les alias reads_per_5sec/writes_per_5sec ne sont pas
-- utilisables dans une EXPRESSION du ORDER BY (seulement en référence directe)
ORDER BY (s2.reads - COALESCE(s1.reads, 0)) + (s2.writes - COALESCE(s1.writes, 0)) DESC;
```

**Utilité** :
- Mesure l'activité I/O **instantanée** (sur 5 secondes)
- Identifie les backends qui consomment le plus en ce moment précis
- Permet de suivre l'évolution dans le temps

### Exemple 5 : Corrélation avec le temps CPU et les locks

```sql
SELECT
    a.pid,
    a.usename,
    a.application_name,
    a.state,
    (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))  AS total_io_blocks,
    pg_size_pretty(
        (SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid))
    ) AS wal_generated,
    NOW() - a.query_start  AS duration,
    NOW() - a.state_change AS time_in_state,
    a.wait_event_type,
    a.wait_event,
    LEFT(a.query, 80) AS query_preview
FROM pg_stat_activity a  
WHERE a.state = 'active'  
  AND a.backend_type = 'client backend'
ORDER BY total_io_blocks DESC NULLS LAST  
LIMIT 20;  
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
    a.pid,
    a.usename,
    a.application_name,
    NOW() - a.query_start AS duration,
    (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid))  AS reads,
    (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid))  AS writes,
    pg_size_pretty(
        (SELECT wal_bytes  FROM pg_stat_get_backend_wal(a.pid))
    ) AS wal,
    a.state,
    a.wait_event_type,
    LEFT(a.query, 60) AS query
FROM pg_stat_activity a  
WHERE a.state IS NOT NULL AND a.state <> 'idle'  
ORDER BY (  
    (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))
) DESC NULLS LAST
LIMIT 20;
```

**Analyse** : Supposons que vous trouvez :
```
  pid  | usename |  duration  |   reads   |  writes  |   wal   |  state  | wait_event_type |           query
-------+---------+------------+-----------+----------+---------+---------+-----------------+--------------------------
 14532 | appuser | 00:45:23   | 23450000  |  450000  | 345 GB  | active  | IO              | DELETE FROM audit_logs WHERE...
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
-- Identifier les backends qui génèrent le plus de WAL (PG 18)
SELECT
    a.pid,
    a.usename,
    a.application_name,
    pg_size_pretty(w.wal_bytes) AS wal_generated,
    w.wal_records,
    NOW() - a.backend_start AS connection_age,
    NOW() - a.query_start   AS query_age,
    a.state,
    LEFT(a.query, 100) AS query
FROM pg_stat_activity a  
CROSS JOIN LATERAL pg_stat_get_backend_wal(a.pid) w  
WHERE w.wal_bytes > 0  
ORDER BY w.wal_bytes DESC  
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
WITH per_backend AS (
    SELECT
        a.usename,
        a.application_name,
        a.pid,
        (SELECT SUM(reads) FROM pg_stat_get_backend_io(a.pid)) AS reads,
        (SELECT SUM(hits)  FROM pg_stat_get_backend_io(a.pid)) AS hits
    FROM pg_stat_activity a
    WHERE a.state IS NOT NULL AND a.state <> 'idle'
      AND a.backend_type = 'client backend'
)
SELECT
    usename,
    application_name,
    COUNT(*) AS connections,
    SUM(reads) AS total_disk_reads,
    SUM(hits)  AS total_cache_hits,
    ROUND(100.0 * SUM(hits) / NULLIF(SUM(hits) + SUM(reads), 0), 2) AS cache_hit_ratio,
    ROUND(AVG(reads)::numeric, 0) AS avg_disk_reads_per_conn
FROM per_backend  
GROUP BY usename, application_name  
HAVING SUM(reads) > 0  
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
-- track_io_timing doit être activé pour avoir des temps non nuls
WITH per_backend AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        a.query,
        (SELECT SUM(read_time)  FROM pg_stat_get_backend_io(a.pid)) AS read_time,
        (SELECT SUM(write_time) FROM pg_stat_get_backend_io(a.pid)) AS write_time,
        (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid)) AS reads,
        (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid)) AS writes
    FROM pg_stat_activity a
    WHERE a.state = 'active'
)
SELECT
    pid,
    usename,
    application_name,
    ROUND(read_time::numeric, 2)  AS read_time_ms,
    ROUND(write_time::numeric, 2) AS write_time_ms,
    ROUND((read_time + write_time)::numeric, 2) AS total_io_time_ms,
    reads,
    writes,
    ROUND((read_time / NULLIF(reads, 0))::numeric, 2) AS avg_read_latency_ms,
    LEFT(query, 80) AS query
FROM per_backend  
WHERE (read_time + write_time) > 0  
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
-- Toutes utilisent les FONCTIONS PG 18 par backend

-- 1. I/O par application
SELECT
    a.application_name,
    SUM((SELECT SUM(reads)  FROM pg_stat_get_backend_io(a.pid))) AS total_reads,
    SUM((SELECT SUM(writes) FROM pg_stat_get_backend_io(a.pid))) AS total_writes,
    SUM((SELECT SUM(hits)   FROM pg_stat_get_backend_io(a.pid))) AS total_hits
FROM pg_stat_activity a  
WHERE a.backend_type = 'client backend'  
GROUP BY a.application_name;  

-- 2. WAL par application
SELECT
    a.application_name,
    SUM(w.wal_bytes)   AS total_wal_bytes,
    SUM(w.wal_records) AS total_wal_records
FROM pg_stat_activity a  
CROSS JOIN LATERAL pg_stat_get_backend_wal(a.pid) w  
WHERE a.backend_type = 'client backend'  
GROUP BY a.application_name;  

-- 3. Top 10 des backends par I/O
SELECT
    a.pid,
    a.usename,
    (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid)) AS total_io
FROM pg_stat_activity a  
WHERE a.state = 'active'  
ORDER BY total_io DESC NULLS LAST  
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
        a.pid,
        a.usename,
        (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))   AS total_io,
        ROUND(
            (SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid)) / 1073741824.0,
            2
        ) AS wal_gb
    FROM pg_stat_activity a
    WHERE a.state = 'active'
      AND (
        (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))
            > $THRESHOLD_IO_BLOCKS
        OR
        (SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid))
            > $THRESHOLD_WAL_GB * 1073741824
      )
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
-- Trouver les requêtes qui génèrent le plus de WAL.
-- pg_stat_statements expose DIRECTEMENT le WAL par requête normalisée
-- (colonnes wal_records / wal_fpi / wal_bytes, depuis PG 13). Inutile — et
-- incorrect — d'aller chercher un « wal_bytes » dans pg_stat_activity : cette
-- vue n'a pas cette colonne (le WAL par backend passe par pg_stat_get_backend_wal()).
SELECT
    pss.queryid,
    LEFT(pss.query, 100) AS query_preview,
    pss.calls,
    ROUND((pss.total_exec_time / 1000)::numeric, 1) AS total_sec,
    pss.wal_records,
    pss.wal_fpi,
    pg_size_pretty(pss.wal_bytes) AS wal_generated
FROM pg_stat_statements pss  
WHERE pss.calls > 100  
ORDER BY pss.wal_bytes DESC  
LIMIT 10;  
```

---

## Cas d'Usage Pratiques

### Cas 1 : Audit de consommation de ressources par application

```sql
-- ⚠️ Avertissement : pg_stat_activity ne montre que les sessions ACTUELLES.
-- Cette requête ne couvre donc pas une "période passée" — pour cela, il faut
-- snapshotter périodiquement les valeurs et stocker les deltas (cf. Exemple 4
-- plus haut). L'exemple ci-dessous mesure la consommation des sessions ouvertes
-- depuis moins de 7 jours.
SELECT
    a.application_name,
    a.usename,
    COUNT(DISTINCT a.pid) AS connexions_ouvertes,
    pg_size_pretty(
        SUM((SELECT SUM(read_bytes)  FROM pg_stat_get_backend_io(a.pid)))
    ) AS total_disk_reads,
    pg_size_pretty(
        SUM((SELECT SUM(write_bytes) FROM pg_stat_get_backend_io(a.pid)))
    ) AS total_disk_writes,
    pg_size_pretty(
        SUM((SELECT wal_bytes        FROM pg_stat_get_backend_wal(a.pid)))
    ) AS total_wal_generated
FROM pg_stat_activity a  
WHERE a.backend_type = 'client backend'  
  AND a.backend_start > NOW() - INTERVAL '7 days'
GROUP BY a.application_name, a.usename  
ORDER BY SUM((SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid))) DESC NULLS LAST;  
```

**Utilité** :
- Facturation interne par application
- Identification des applications à optimiser
- Planification de capacité

### Cas 2 : Détection d'anomalies en temps réel

```sql
-- Détecter les backends avec une activité I/O ou WAL anormale (Z-score > 3)
WITH backend_stats AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))  AS total_io,
        (SELECT wal_bytes           FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes,
        a.query
    FROM pg_stat_activity a
    WHERE a.state = 'active'
),
stats_summary AS (
    SELECT
        AVG(total_io)::numeric  AS avg_io,
        STDDEV(total_io)        AS stddev_io,
        AVG(wal_bytes)::numeric AS avg_wal,
        STDDEV(wal_bytes)       AS stddev_wal
    FROM backend_stats
)
SELECT
    bs.pid,
    bs.usename,
    bs.application_name,
    bs.total_io,
    ROUND((bs.total_io  - ss.avg_io)  / NULLIF(ss.stddev_io, 0), 2)  AS io_zscore,
    pg_size_pretty(bs.wal_bytes) AS wal,
    ROUND((bs.wal_bytes - ss.avg_wal) / NULLIF(ss.stddev_wal, 0), 2) AS wal_zscore,
    LEFT(bs.query, 80) AS query
FROM backend_stats bs, stats_summary ss  
WHERE (bs.total_io  - ss.avg_io)  / NULLIF(ss.stddev_io, 0)  > 3  -- 3 écarts-types  
   OR (bs.wal_bytes - ss.avg_wal) / NULLIF(ss.stddev_wal, 0) > 3
ORDER BY io_zscore DESC NULLS LAST;
```

**Explication** : Utilise le **Z-score** (écart à la moyenne) pour détecter les backends avec une activité statistiquement anormale.

### Cas 3 : Optimisation de l'autovacuum

```sql
-- Identifier si l'autovacuum génère beaucoup d'I/O
SELECT
    a.pid,
    a.backend_type,
    (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid)) AS reads,
    (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid)) AS writes,
    (SELECT SUM(read_time)  FROM pg_stat_get_backend_io(a.pid)) AS read_time_ms,
    (SELECT SUM(write_time) FROM pg_stat_get_backend_io(a.pid)) AS write_time_ms,
    NOW() - a.backend_start AS duration,
    LEFT(a.query, 80) AS query
FROM pg_stat_activity a  
WHERE a.backend_type = 'autovacuum worker'  
  AND a.state = 'active'
ORDER BY (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid))
    DESC NULLS LAST;
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

| Métrique (via `pg_stat_get_backend_io/wal`) | Seuil Warning | Seuil Critical | Action |
|----------|---------------|----------------|--------|
| `reads + writes` | > 500 000 | > 2 000 000 | Analyser la requête |
| `wal_bytes` | > 1 GB | > 10 GB | Vérifier la transaction |
| Ratio `hits / (hits + reads)` | < 95% | < 90% | Optimiser les index |
| `read_time + write_time` (ms) | > 60 000 | > 300 000 | Vérifier le disque |

### 2. Surveiller en continu

Créez une vue dédiée pour le monitoring :

```sql
CREATE OR REPLACE VIEW v_backend_io_monitoring AS  
SELECT  
    a.pid,
    a.usename,
    a.application_name,
    a.state,
    NOW() - a.backend_start AS connection_age,
    NOW() - a.query_start   AS query_age,
    (SELECT SUM(reads)      FROM pg_stat_get_backend_io(a.pid))  AS reads,
    (SELECT SUM(writes)     FROM pg_stat_get_backend_io(a.pid))  AS writes,
    (SELECT SUM(hits)       FROM pg_stat_get_backend_io(a.pid))  AS hits,
    ROUND(
        100.0 *
        (SELECT SUM(hits)  FROM pg_stat_get_backend_io(a.pid))
        / NULLIF(
            (SELECT SUM(hits + reads) FROM pg_stat_get_backend_io(a.pid)),
            0
        )
    , 2) AS cache_hit_ratio,
    pg_size_pretty(
        (SELECT wal_bytes  FROM pg_stat_get_backend_wal(a.pid))
    ) AS wal_size,
    ROUND((SELECT SUM(read_time)  FROM pg_stat_get_backend_io(a.pid))::numeric, 2)
        AS read_time_ms,
    ROUND((SELECT SUM(write_time) FROM pg_stat_get_backend_io(a.pid))::numeric, 2)
        AS write_time_ms,
    a.wait_event_type,
    a.wait_event,
    LEFT(a.query, 100) AS query_preview
FROM pg_stat_activity a  
WHERE a.backend_type = 'client backend'  
  AND a.state != 'idle';

-- Utilisation
SELECT * FROM v_backend_io_monitoring  
WHERE cache_hit_ratio < 90 OR wal_size > '1 GB'  
ORDER BY reads + writes DESC NULLS LAST;  
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
    (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid)) AS total_io,
    pg_size_pretty(
        (SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid))
    ) AS wal,
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
ORDER BY total_io DESC NULLS LAST;  
```

### 4. Archiver les statistiques historiques

Les statistiques dans `pg_stat_activity` sont en temps réel et éphémères. Pour l'analyse historique :

```sql
-- Table d'historique
CREATE TABLE backend_io_history (
    captured_at      timestamp DEFAULT NOW(),
    pid              integer,
    usename          text,
    application_name text,
    reads            bigint,
    writes           bigint,
    wal_bytes        bigint,
    query            text
);

-- Job périodique (via pg_cron ou cron système)
INSERT INTO backend_io_history (pid, usename, application_name, reads, writes, wal_bytes, query)  
SELECT  
    a.pid,
    a.usename,
    a.application_name,
    (SELECT SUM(reads)  FROM pg_stat_get_backend_io(a.pid)),
    (SELECT SUM(writes) FROM pg_stat_get_backend_io(a.pid)),
    (SELECT wal_bytes   FROM pg_stat_get_backend_wal(a.pid)),
    a.query
FROM pg_stat_activity a  
WHERE a.state = 'active'  
  AND (
    (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid)) > 1000
    OR
    (SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid)) > 1048576
  );
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

La collecte des compteurs (reads, writes, hits, wal_bytes…) a un **impact minimal** et est activée par défaut.

> ℹ️ Distinction importante : `track_io_timing` ne désactive **pas** les compteurs eux-mêmes — il contrôle uniquement la mesure des **temps** (`read_time`, `write_time`, `fsync_time`). Désactiver `track_io_timing` n'a pas d'impact sur `reads`, `writes`, `hits` qui restent collectés.

### Compatibilité

- **Fonctionnalité PostgreSQL 18** : il s'agit de **trois nouvelles fonctions** (`pg_stat_get_backend_io(pid)`, `pg_stat_get_backend_wal(pid)`, `pg_stat_reset_backend_stats(pid)`), **non disponibles** dans les versions antérieures. La vue `pg_stat_io` (déjà présente en PG 16+) est aussi enrichie de colonnes en octets et de nouvelles lignes pour le WAL.
- Migration depuis une version antérieure : les fonctions et les nouvelles colonnes de `pg_stat_io` apparaissent automatiquement après `pg_upgrade` vers PG 18.
- Scripts existants : non impactés. Toute requête utilisant l'ancienne API continue de fonctionner ; il faut écrire de nouvelles requêtes pour exploiter les fonctions PG 18.

---

## Requêtes Prêtes à l'Emploi

### Dashboard de santé complet

```sql
-- Vue d'ensemble : Backends actifs avec I/O et WAL (PG 18)
WITH per_backend AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        a.state,
        a.query,
        a.query_start,
        a.wait_event_type,
        a.wait_event,
        (SELECT SUM(reads)  FROM pg_stat_get_backend_io(a.pid))  AS disk_reads,
        (SELECT SUM(writes) FROM pg_stat_get_backend_io(a.pid))  AS disk_writes,
        (SELECT SUM(hits)   FROM pg_stat_get_backend_io(a.pid))  AS cache_hits,
        (SELECT wal_bytes   FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
    FROM pg_stat_activity a
    WHERE a.state IS NOT NULL AND a.state <> 'idle'
      AND a.backend_type = 'client backend'
)
SELECT
    ROW_NUMBER() OVER (ORDER BY disk_reads + disk_writes DESC NULLS LAST) AS rank,
    pid,
    usename,
    application_name,
    state,
    ROUND(EXTRACT(EPOCH FROM (NOW() - query_start)) / 60, 1) AS minutes_running,
    disk_reads,
    disk_writes,
    cache_hits,
    ROUND(100.0 * cache_hits / NULLIF(cache_hits + disk_reads, 0), 1) AS cache_hit_pct,
    pg_size_pretty(wal_bytes) AS wal_generated,
    wait_event_type || ': ' || COALESCE(wait_event, 'none') AS wait_status,
    LEFT(query, 60) AS query_preview
FROM per_backend  
ORDER BY disk_reads + disk_writes DESC NULLS LAST  
LIMIT 20;  
```

### Top 5 des gourmands en ressources

```sql
-- Les 5 backends qui consomment le plus de chaque ressource (PG 18)
WITH active_with_stats AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        a.query,
        (SELECT SUM(reads)  FROM pg_stat_get_backend_io(a.pid))  AS reads,
        (SELECT SUM(writes) FROM pg_stat_get_backend_io(a.pid))  AS writes,
        (SELECT wal_bytes   FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
    FROM pg_stat_activity a
    WHERE a.state = 'active'
)
(SELECT 'TOP I/O READS' AS category, pid, usename, application_name,
        reads AS value, LEFT(query, 50) AS query
 FROM active_with_stats ORDER BY reads DESC NULLS LAST LIMIT 5)
UNION ALL
(SELECT 'TOP I/O WRITES', pid, usename, application_name,
        writes, LEFT(query, 50)
 FROM active_with_stats ORDER BY writes DESC NULLS LAST LIMIT 5)
UNION ALL
(SELECT 'TOP WAL GEN', pid, usename, application_name,
        wal_bytes, LEFT(query, 50)
 FROM active_with_stats ORDER BY wal_bytes DESC NULLS LAST LIMIT 5);
```

### Alertes automatiques

```sql
-- Backends nécessitant une attention immédiate (PG 18)
WITH per_backend AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        a.query,
        (SELECT SUM(reads)  FROM pg_stat_get_backend_io(a.pid))  AS reads,
        (SELECT SUM(writes) FROM pg_stat_get_backend_io(a.pid))  AS writes,
        (SELECT SUM(hits)   FROM pg_stat_get_backend_io(a.pid))  AS hits,
        (SELECT wal_bytes   FROM pg_stat_get_backend_wal(a.pid)) AS wal_bytes
    FROM pg_stat_activity a
    WHERE a.state = 'active'
)
SELECT
    'ALERT' AS priority,
    pid,
    usename,
    application_name,
    CASE
        WHEN reads     > 1000000      THEN 'Excessive disk reads (>1M blocks)'
        WHEN writes    > 1000000      THEN 'Excessive disk writes (>1M blocks)'
        WHEN wal_bytes > 10737418240  THEN 'Excessive WAL generation (>10GB)'
        WHEN (100.0 * hits / NULLIF(hits + reads, 0)) < 80
                                       THEN 'Low cache hit ratio (<80%)'
    END AS issue,
    pg_size_pretty(wal_bytes) AS wal,
    LEFT(query, 100) AS query
FROM per_backend  
WHERE reads     > 1000000  
   OR writes    > 1000000
   OR wal_bytes > 10737418240
   OR (100.0 * hits / NULLIF(hits + reads, 0)) < 80;
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
4. PG 18 expose ces statistiques **par backend** via deux **fonctions** dédiées : `pg_stat_get_backend_io(pid)` et `pg_stat_get_backend_wal(pid)`. Pour les réinitialiser : `pg_stat_reset_backend_stats(pid)`.
5. `pg_stat_io` gagne aussi en PG 18 des colonnes en octets (`read_bytes`, `write_bytes`, `extend_bytes`) et de nouvelles lignes pour l'I/O WAL.
6. Combinez toujours avec d'autres vues (`pg_stat_statements`, `pg_locks`) pour un diagnostic complet.

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
-- Dans votre fichier .psqlrc (PG 18)
\set io_top 'SELECT a.pid, a.usename, a.application_name, (SELECT SUM(reads + writes) FROM pg_stat_get_backend_io(a.pid)) AS total_io, pg_size_pretty((SELECT wal_bytes FROM pg_stat_get_backend_wal(a.pid))) AS wal, LEFT(a.query, 80) FROM pg_stat_activity a WHERE a.state = \'active\' ORDER BY total_io DESC NULLS LAST LIMIT 10;'

-- Utilisation : \:io_top
```

**🎯 Objectif** : Faire de PostgreSQL 18 le SGBD le plus observable et diagnosticable du marché !

⏭️ [Analyse des logs : Configuration et interprétation (log_line_prefix, auto_explain)](/14-observabilite-et-monitoring/05-analyse-des-logs.md)
