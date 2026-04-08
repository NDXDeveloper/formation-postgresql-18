🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Requêtes d'Administration PostgreSQL
## Locks, Bloat et Utilisation des Index

---

## Table des Matières

1. [Introduction à l'Administration par Requêtes](#1-introduction-%C3%A0-ladministration-par-requ%C3%AAtes)  
2. [Gestion des Locks (Verrous)](#2-gestion-des-locks-verrous)  
3. [Détection et Analyse du Bloat](#3-d%C3%A9tection-et-analyse-du-bloat)  
4. [Surveillance de l'Utilisation des Index](#4-surveillance-de-lutilisation-des-index)  
5. [Mise en Pratique : Scénarios Courants](#5-mise-en-pratique--sc%C3%A9narios-courants)  
6. [Bonnes Pratiques et Recommandations](#6-bonnes-pratiques-et-recommandations)

---

## 1. Introduction à l'Administration par Requêtes

### 1.1. Pourquoi ces Requêtes sont Essentielles

En tant qu'administrateur ou développeur travaillant avec PostgreSQL, vous devez être capable de **diagnostiquer les problèmes de performance** et de **maintenir la santé de votre base de données**. PostgreSQL expose une multitude d'informations sur son état interne via des **vues système** (system views), et vous pouvez interroger ces vues avec des requêtes SQL standard.

Les trois domaines les plus critiques pour l'administration quotidienne sont :

1. **Les Locks (Verrous)** : Identifier les blocages entre transactions qui ralentissent ou figent votre application  
2. **Le Bloat (Gonflement)** : Détecter l'espace disque gaspillé dans vos tables et index  
3. **L'Utilisation des Index** : Vérifier que vos index sont effectivement utilisés et identifier ceux qui sont inutiles

### 1.2. Les Vues Système de PostgreSQL

PostgreSQL maintient automatiquement des **vues système** qui contiennent des métadonnées et des statistiques sur :

- Les connexions actives (`pg_stat_activity`)
- Les verrous en cours (`pg_locks`)
- Les statistiques de tables (`pg_stat_user_tables`)
- Les statistiques d'index (`pg_stat_user_indexes`)
- Le catalogue système (`pg_class`, `pg_stat_all_tables`)

Ces vues sont **mises à jour en temps réel** (ou quasi temps réel) par PostgreSQL et constituent votre fenêtre d'observation sur le fonctionnement interne du serveur.

---

## 2. Gestion des Locks (Verrous)

### 2.1. Qu'est-ce qu'un Lock ?

Un **lock (verrou)** est un mécanisme qui empêche deux transactions d'accéder simultanément à une ressource de manière incompatible. Par exemple :

- Si la transaction A est en train de modifier une ligne, la transaction B qui veut aussi modifier cette même ligne devra **attendre** que A se termine
- C'est ce qui garantit la **cohérence des données** (propriété ACID)

#### Types de Locks Courants

PostgreSQL utilise différents types de verrous :

| Type de Lock | Description | Exemple |
|--------------|-------------|---------|
| **AccessShareLock** | Le plus permissif, posé par SELECT | `SELECT * FROM users` |
| **RowShareLock** | Posé par SELECT FOR UPDATE | `SELECT * FROM users FOR UPDATE` |
| **RowExclusiveLock** | Posé par INSERT, UPDATE, DELETE | `UPDATE users SET ...` |
| **ShareUpdateExclusiveLock** | Posé par VACUUM, CREATE INDEX CONCURRENTLY | `VACUUM users` |
| **ShareLock** | Posé par CREATE INDEX (non concurrent) | `CREATE INDEX idx_users ...` |
| **ExclusiveLock** | Bloque tout sauf AccessShareLock | `LOCK TABLE users` |
| **AccessExclusiveLock** | Le plus restrictif, bloque tout | `ALTER TABLE users ...` |

### 2.2. Problèmes Causés par les Locks

Les locks deviennent problématiques quand :

1. **Deadlock (Interblocage)** : Deux transactions s'attendent mutuellement  
2. **Lock Wait (Attente)** : Une transaction attend trop longtemps qu'un lock soit libéré  
3. **Blocking Cascade** : Une transaction bloque plusieurs autres qui elles-mêmes en bloquent d'autres

Ces situations peuvent **ralentir drastiquement** votre application ou la **figer complètement**.

### 2.3. Requête : Identifier les Locks Actifs

Cette requête vous montre **tous les locks actuellement détenus** dans votre base de données :

```sql
SELECT
    pl.locktype,
    pl.database,
    pl.relation::regclass AS relation_name,
    pl.page,
    pl.tuple,
    pl.virtualxid,
    pl.transactionid,
    pl.mode,
    pl.granted,
    pl.pid AS process_id
FROM
    pg_locks pl
ORDER BY
    pl.pid, pl.granted DESC;
```

**Explication des colonnes importantes** :

- **locktype** : Type de ressource verrouillée (`relation`, `transactionid`, `tuple`, etc.)  
- **relation_name** : Nom de la table concernée (si applicable)  
- **mode** : Type de verrou (ex: `AccessShareLock`, `ExclusiveLock`)  
- **granted** : `true` si le verrou est obtenu, `false` si en attente  
- **pid** : ID du processus qui détient ou attend le verrou

### 2.4. Requête : Identifier les Transactions Bloquantes

Cette requête plus avancée identifie **qui bloque qui** :

```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application,
    blocking_activity.application_name AS blocking_application
FROM
    pg_catalog.pg_locks blocked_locks
JOIN
    pg_catalog.pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN
    pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN
    pg_catalog.pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE
    NOT blocked_locks.granted;
```

**Ce que cette requête vous montre** :

- **blocked_pid** : Le processus qui est bloqué  
- **blocking_pid** : Le processus qui bloque  
- **blocked_statement** : La requête qui est bloquée  
- **blocking_statement** : La requête qui bloque

### 2.5. Requête : Deadlocks en Cours

PostgreSQL détecte automatiquement les deadlocks et les résout en annulant une des transactions. Pour voir les **deadlocks récents**, consultez les logs PostgreSQL ou utilisez cette vue :

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    wait_event_type,
    wait_event,
    state_change
FROM
    pg_stat_activity
WHERE
    wait_event_type = 'Lock'
    AND state = 'active'
ORDER BY
    state_change;
```

**Interprétation** :
- Les processus listés attendent un verrou
- Si plusieurs processus s'attendent mutuellement, vous avez un deadlock potentiel

### 2.6. Solution : Tuer une Transaction Bloquante

Si vous identifiez une transaction qui bloque toutes les autres et qui doit être interrompue :

```sql
-- Terminer proprement (recommandé)
SELECT pg_cancel_backend(12345);  -- Remplacer 12345 par le PID

-- Tuer de force (en dernier recours)
SELECT pg_terminate_backend(12345);
```

**Différence importante** :
- `pg_cancel_backend()` : Demande poliment à la transaction de s'arrêter (SIGINT)  
- `pg_terminate_backend()` : Force l'arrêt immédiat (SIGTERM)

---

## 3. Détection et Analyse du Bloat

### 3.1. Qu'est-ce que le Bloat ?

Le **bloat (gonflement)** est l'accumulation d'**espace mort** dans vos tables et index. Cela se produit à cause du fonctionnement de PostgreSQL avec **MVCC (Multi-Version Concurrency Control)**.

#### Pourquoi le Bloat se Produit-il ?

1. Quand vous faites un `UPDATE` ou `DELETE`, PostgreSQL ne supprime pas immédiatement les anciennes versions des lignes  
2. Ces anciennes versions (appelées **dead tuples**) restent dans la table pour permettre aux transactions concurrentes de les voir  
3. Le processus **VACUUM** nettoie ces dead tuples... mais si VACUUM ne tourne pas assez souvent, elles s'accumulent  
4. Résultat : Vos tables et index occupent **beaucoup plus d'espace** qu'ils ne devraient

#### Conséquences du Bloat

- **Ralentissement des requêtes** : PostgreSQL doit parcourir plus de données  
- **Gaspillage d'espace disque**  
- **Cache moins efficace** : Les buffers sont remplis de données mortes  
- **Index moins performants**

### 3.2. Requête : Détecter le Bloat dans les Tables

Cette requête estime le pourcentage de bloat dans vos tables :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    round(100 * pg_relation_size(schemaname||'.'||tablename) /
          NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS table_pct,
    n_live_tup,
    n_dead_tup,
    round(100 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tuple_pct,
    last_vacuum,
    last_autovacuum
FROM
    pg_stat_user_tables
WHERE
    n_live_tup > 0
ORDER BY
    n_dead_tup DESC
LIMIT 20;
```

**Colonnes importantes** :

- **total_size** : Taille totale (table + index + TOAST)  
- **table_size** : Taille de la table seule  
- **n_live_tup** : Nombre de lignes vivantes  
- **n_dead_tup** : Nombre de lignes mortes (bloat)  
- **dead_tuple_pct** : Pourcentage de bloat  
- **last_vacuum** / **last_autovacuum** : Dernière exécution de VACUUM

**Seuils d'alerte** :
- `dead_tuple_pct > 20%` : Bloat modéré, surveiller  
- `dead_tuple_pct > 40%` : Bloat important, action recommandée  
- `dead_tuple_pct > 60%` : Bloat critique, action urgente

### 3.3. Requête : Estimer le Bloat Précisément (Extension pgstattuple)

Pour une estimation plus précise, utilisez l'extension `pgstattuple` :

```sql
-- Installer l'extension (une seule fois)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Analyser une table spécifique
SELECT * FROM pgstattuple('ma_table');
```

Cette fonction retourne :

- **table_len** : Taille totale de la table en bytes  
- **tuple_count** : Nombre de lignes vivantes  
- **dead_tuple_count** : Nombre de lignes mortes  
- **free_space** : Espace libre récupérable  
- **dead_tuple_percent** : Pourcentage exact de bloat

**Attention** : `pgstattuple()` fait un scan complet de la table, donc **coûteux** sur de grosses tables. À utiliser hors production ou en maintenance.

### 3.4. Requête : Détecter le Bloat dans les Index

Les index aussi peuvent souffrir de bloat :

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE
        WHEN idx_scan = 0 THEN 'NEVER USED'
        ELSE 'OK'
    END AS usage_status
FROM
    pg_stat_user_indexes
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    pg_relation_size(indexrelid) DESC
LIMIT 20;
```

**Indicateurs de bloat d'index** :

- Un index qui ne cesse de grossir alors que la table est stable
- Un ratio `idx_tup_read / idx_tup_fetch` très élevé (index inefficace)

### 3.5. Solution : Nettoyer le Bloat

Il existe plusieurs stratégies :

#### Option 1 : VACUUM (Standard)

```sql
-- VACUUM standard (récupère l'espace mais ne le rend pas à l'OS)
VACUUM VERBOSE ma_table;

-- VACUUM ANALYZE (+ mise à jour des statistiques)
VACUUM ANALYZE ma_table;
```

**Avantage** : Rapide, n'impose pas de verrou exclusif  
**Inconvénient** : Ne réduit pas physiquement la taille du fichier  

#### Option 2 : VACUUM FULL (Drastique)

```sql
-- Réécrit complètement la table et rend l'espace à l'OS
VACUUM FULL ma_table;
```

**Avantage** : Récupère vraiment tout l'espace  
**Inconvénient** : Pose un verrou exclusif (AccessExclusiveLock), bloque toute opération sur la table  

#### Option 3 : REINDEX (Pour les Index)

```sql
-- Reconstruire un index spécifique
REINDEX INDEX mon_index;

-- Reconstruire tous les index d'une table
REINDEX TABLE ma_table;

-- Version concurrente (sans bloquer les lectures/écritures)
REINDEX INDEX CONCURRENTLY mon_index;
```

**PostgreSQL 18** : REINDEX CONCURRENTLY est plus performant grâce aux optimisations I/O.

#### Option 4 : pg_repack (Extension Recommandée)

L'extension `pg_repack` permet de réorganiser une table **sans verrou exclusif** :

```sql
-- En ligne de commande
pg_repack -d ma_base -t ma_table
```

**Avantage majeur** : Pas d'interruption de service

---

## 4. Surveillance de l'Utilisation des Index

### 4.1. Pourquoi Surveiller les Index ?

Les index sont **essentiels pour la performance**, mais :

1. **Coût de maintenance** : Chaque index ralentit les INSERT/UPDATE/DELETE  
2. **Consommation d'espace disque**  
3. **Cache pollution** : Index inutiles occupent de la mémoire

Il est donc crucial d'identifier :
- Les **index inutilisés** (à supprimer)
- Les **index manquants** (à créer)
- Les **index redondants** (doublons)

### 4.2. Requête : Index Inutilisés

Cette requête identifie les index qui **ne sont jamais utilisés** :

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS number_of_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size
FROM
    pg_stat_user_indexes
WHERE
    schemaname NOT IN ('pg_catalog', 'information_schema')
    AND idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey'  -- Exclure les primary keys
ORDER BY
    pg_relation_size(indexrelid) DESC;
```

**Interprétation** :

- **idx_scan = 0** : L'index n'a JAMAIS été utilisé depuis le dernier démarrage ou reset des stats
- Si un index apparaît ici et que votre serveur tourne depuis longtemps : **candidat à la suppression**

**Précautions** :
- Ne supprimez pas les primary keys ou les contraintes UNIQUE
- Vérifiez que les statistiques ne viennent pas d'être réinitialisées (`pg_stat_reset()`)

### 4.3. Requête : Taux d'Utilisation des Index par Table

Pour voir quelle proportion de requêtes utilise les index vs scan séquentiel :

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    seq_scan AS sequential_scans,
    idx_scan AS index_scans,
    round(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS index_usage_pct,
    n_live_tup AS live_rows
FROM
    pg_stat_user_tables
WHERE
    (seq_scan + idx_scan) > 0
ORDER BY
    seq_scan DESC,
    pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

**Analyse** :

- **index_usage_pct > 90%** : Excellent, vos index sont bien utilisés  
- **index_usage_pct < 50%** : Beaucoup de scans séquentiels, envisagez d'ajouter des index  
- **seq_scan élevé sur grosse table** : Problème de performance potentiel

**Nuance** : Sur de petites tables (<1000 lignes), PostgreSQL préfère souvent le scan séquentiel car c'est plus rapide. C'est normal.

### 4.4. Requête : Index Redondants

Des index redondants sont des index qui font "doublon". Par exemple :

- Index 1 : `(a, b, c)`
- Index 2 : `(a, b)` ← **Redondant**, car l'index 1 peut le remplacer

```sql
SELECT
    pg_size_pretty(sum(pg_relation_size(idx))::bigint) AS total_size,
    array_agg(indexrelname) AS indexes,
    tablename
FROM (
    SELECT
        indexrelid::regclass AS idx,
        indrelid::regclass AS tablename,
        indexrelname,
        string_to_array(indkey::text, ' ')::int[] AS cols
    FROM
        pg_index
    JOIN
        pg_class ON pg_class.oid = pg_index.indexrelid
    WHERE
        indisvalid
) sub
GROUP BY
    tablename, cols
HAVING
    count(*) > 1
ORDER BY
    sum(pg_relation_size(idx)) DESC;
```

**Cette requête détecte** :
- Les index ayant exactement les mêmes colonnes dans le même ordre
- Ces index sont des **doublons parfaits** et l'un peut être supprimé

### 4.5. Requête : Cache Hit Ratio des Index

Mesure l'efficacité du cache pour les index :

```sql
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_blks_read AS disk_reads,
    idx_blks_hit AS cache_hits,
    round(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS cache_hit_ratio
FROM
    pg_statio_user_indexes
WHERE
    (idx_blks_hit + idx_blks_read) > 0
ORDER BY
    cache_hit_ratio ASC
LIMIT 20;
```

**Objectif** :
- **cache_hit_ratio > 95%** : Excellent  
- **cache_hit_ratio < 80%** : L'index est souvent lu depuis le disque, envisagez d'augmenter `shared_buffers`

### 4.6. Requête : Taille de Tous les Index d'une Table

Pour connaître l'empreinte disque de vos index :

```sql
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM
    pg_stat_user_indexes
WHERE
    tablename = 'ma_table'  -- Remplacer par votre table
ORDER BY
    pg_relation_size(indexrelid) DESC;
```

**Analyse** :
- Si un gros index a `idx_scan = 0` : **Supprimez-le**
- Si un petit index a `idx_scan` très élevé : **Gardez-le absolument**

### 4.7. Requête : Suggestions d'Index Manquants (Requêtes Lentes)

Cette approche nécessite l'extension `pg_stat_statements` (fortement recommandée) :

```sql
-- Installer l'extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Trouver les requêtes les plus lentes
SELECT
    round(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_time_ms,
    query
FROM
    pg_stat_statements
WHERE
    mean_exec_time > 100  -- Plus de 100ms en moyenne
ORDER BY
    total_exec_time DESC
LIMIT 20;
```

**Processus d'analyse** :

1. Identifiez les requêtes lentes avec `pg_stat_statements`  
2. Pour chaque requête, faites un `EXPLAIN ANALYZE` pour voir le plan d'exécution  
3. Si vous voyez des `Seq Scan` sur de grosses tables : **créez un index**

Exemple :

```sql
EXPLAIN ANALYZE  
SELECT * FROM users WHERE email = 'john@example.com';  
```

Si le plan montre :
```
Seq Scan on users  (cost=0.00..1234.56 rows=1 width=200) (actual time=45.123..45.125 rows=1 loops=1)
  Filter: (email = 'john@example.com'::text)
```

Alors créez :
```sql
CREATE INDEX idx_users_email ON users(email);
```

### 4.8. Nouveauté PostgreSQL 18 : Skip Scan Optimization

PostgreSQL 18 introduit l'optimisation **Skip Scan** pour les index multi-colonnes.

**Avant PostgreSQL 18** :
- Index sur `(a, b)` ne pouvait pas être utilisé efficacement pour `WHERE b = 5` (sans condition sur `a`)
- PostgreSQL faisait un scan séquentiel

**Avec PostgreSQL 18** :
- PostgreSQL peut "sauter" les valeurs de `a` et scanner uniquement sur `b`
- Plus besoin de créer des index redondants dans certains cas

**Impact** : Moins d'index nécessaires, meilleure performance automatique.

---

## 5. Mise en Pratique : Scénarios Courants

### Scénario 1 : Application Figée, Tous les Utilisateurs Bloqués

**Symptôme** : Votre application ne répond plus, timeouts partout.

**Diagnostic** :

```sql
-- Étape 1 : Voir les connexions actives
SELECT pid, usename, state, query, wait_event_type, wait_event  
FROM pg_stat_activity  
WHERE state != 'idle';  

-- Étape 2 : Identifier qui bloque qui
-- (Utiliser la requête de la section 2.4)

-- Étape 3 : Décision
-- Si une transaction bloque tout depuis longtemps :
SELECT pg_terminate_backend(123456);  -- PID de la transaction bloquante
```

### Scénario 2 : Table qui Grossit sans Cesse

**Symptôme** : Votre table `orders` fait 500 GB alors qu'elle ne devrait faire que 100 GB.

**Diagnostic** :

```sql
-- Vérifier le bloat
SELECT
    tablename,
    n_live_tup,
    n_dead_tup,
    round(100 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE tablename = 'orders';  
```

**Si dead_pct > 40%** :

```sql
-- Solution 1 : VACUUM standard (rapide mais ne réduit pas la taille)
VACUUM ANALYZE orders;

-- Solution 2 : VACUUM FULL (hors prod, bloque la table)
VACUUM FULL orders;

-- Solution 3 : pg_repack (recommandé, pas de blocage)
-- En ligne de commande :
-- pg_repack -d ma_base -t orders
```

### Scénario 3 : Requêtes Soudainement Lentes

**Symptôme** : Une requête qui prenait 50ms prend maintenant 5 secondes.

**Diagnostic** :

```sql
-- Étape 1 : Vérifier les index inutilisés
-- (Section 4.2)

-- Étape 2 : Analyser le plan d'exécution
EXPLAIN ANALYZE  
SELECT * FROM products WHERE category = 'Electronics';  

-- Étape 3 : Si vous voyez "Seq Scan" sur une grosse table
-- Créer un index :
CREATE INDEX idx_products_category ON products(category);

-- Étape 4 : Forcer la mise à jour des statistiques
ANALYZE products;
```

### Scénario 4 : Trop d'Index, Performances INSERT Dégradées

**Symptôme** : Les INSERT sont de plus en plus lents.

**Diagnostic** :

```sql
-- Lister tous les index d'une table avec leur usage
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan
FROM pg_stat_user_indexes  
WHERE tablename = 'ma_table'  
ORDER BY idx_scan ASC;  
```

**Solution** : Supprimer les index avec `idx_scan = 0` :

```sql
DROP INDEX idx_unused_column;
```

**Attention** : Surveillez les performances après suppression pendant quelques jours.

---

## 6. Bonnes Pratiques et Recommandations

### 6.1. Surveillance Proactive

**Mettez en place un monitoring régulier** :

1. **Locks** : Alerter si une transaction bloque depuis > 5 minutes  
2. **Bloat** : Audit hebdomadaire du bloat, VACUUM si nécessaire  
3. **Index** : Audit mensuel de l'utilisation des index

**Outils recommandés** :
- **pg_stat_statements** : Extension indispensable  
- **pgBadger** : Analyse de logs  
- **Prometheus + postgres_exporter** : Monitoring temps réel  
- **Grafana** : Dashboards visuels

### 6.2. Configuration de l'Autovacuum

**PostgreSQL 18** : Autovacuum amélioré avec ajustements dynamiques.

Paramètres clés à tuner dans `postgresql.conf` :

```conf
# Activer autovacuum (par défaut activé)
autovacuum = on

# Nombre de workers (augmenter si beaucoup de tables actives)
autovacuum_max_workers = 3  # Défaut, augmenter à 5-10 si nécessaire

# PostgreSQL 18 : Nouveau paramètre
autovacuum_vacuum_max_threshold = 50000000

# Seuil de déclenchement (% de lignes modifiées)
autovacuum_vacuum_scale_factor = 0.1  # 10% de la table

# Coût du vacuum (limiter l'impact I/O)
autovacuum_vacuum_cost_delay = 2ms  # Pause entre I/O  
autovacuum_vacuum_cost_limit = 200  # Budget I/O  
```

### 6.3. Stratégie d'Indexation

**Principes** :

1. **Indexez les colonnes de filtrage (WHERE)** : Toujours  
2. **Indexez les colonnes de jointure (JOIN)** : FK notamment  
3. **Indexez les colonnes de tri (ORDER BY)** : Si tri fréquent  
4. **N'indexez pas les petites tables** : < 10 000 lignes, inutile  
5. **Utilisez des index partiels** : Pour filtrer sur des valeurs spécifiques

**Exemple d'index partiel** :

```sql
-- Au lieu de indexer toute la colonne status
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';
-- Plus petit, plus rapide pour les requêtes sur status='pending'
```

### 6.4. Maintenance Planifiée

**Hebdomadaire** :
- Audit du bloat (section 3.2)
- Vérification des index inutilisés (section 4.2)

**Mensuel** :
- VACUUM ANALYZE sur les tables critiques
- REINDEX sur les index gonflés
- Revue des requêtes lentes (`pg_stat_statements`)

**Annuel** :
- Audit complet de l'architecture (normalisation, partitionnement)
- Revue des permissions et sécurité

### 6.5. Documentation et Traçabilité

**Tenez un registre** :

- Date de création de chaque index et justification
- Historique des VACUUM FULL / REINDEX
- Incidents de locks et leur résolution

Cela vous aidera à **identifier les patterns** et **prévenir les problèmes récurrents**.

### 6.6. Tests de Charge et Benchmarking

Avant toute modification majeure (ajout d'index, VACUUM FULL, changement de configuration) :

1. **Testez en environnement de préproduction**  
2. **Mesurez l'impact** : Temps de réponse, débit, utilisation CPU/RAM  
3. **Préparez un rollback** : Plan B si ça se passe mal

---

## Résumé des Points Clés

### Locks

- ✅ Utilisez `pg_stat_activity` et `pg_locks` pour diagnostiquer les blocages  
- ✅ Identifiez rapidement la transaction bloquante avec la requête de la section 2.4  
- ✅ Utilisez `pg_cancel_backend()` ou `pg_terminate_backend()` en dernier recours

### Bloat

- ✅ Surveillez le pourcentage de dead tuples avec `pg_stat_user_tables`  
- ✅ Configurez correctement l'autovacuum (PostgreSQL 18 : nouveaux paramètres)  
- ✅ Utilisez VACUUM ou pg_repack pour nettoyer le bloat  
- ✅ Évitez VACUUM FULL en production (bloque la table)

### Index

- ✅ Identifiez et supprimez les index inutilisés (`idx_scan = 0`)  
- ✅ Surveillez le ratio index vs scan séquentiel  
- ✅ Créez des index pour les requêtes lentes (vérifiez avec EXPLAIN)  
- ✅ PostgreSQL 18 : Skip Scan réduit le besoin d'index redondants

---

## Ressources Complémentaires

### Documentation Officielle PostgreSQL

- [Monitoring Database Activity](https://www.postgresql.org/docs/18/monitoring.html)  
- [Routine Database Maintenance](https://www.postgresql.org/docs/18/maintenance.html)  
- [System Administration Functions](https://www.postgresql.org/docs/18/functions-admin.html)

### Outils Recommandés

- **pg_stat_statements** : Tracking des requêtes  
- **pgBadger** : Analyse de logs  
- **pg_repack** : Réorganisation sans verrous  
- **HypoPG** : Tester des index hypothétiques

### Communautés et Support

- Mailing list PostgreSQL : pgsql-general
- Reddit : r/PostgreSQL
- Discord PostgreSQL (communauté francophone et internationale)
- Stack Overflow : Tag `postgresql`

---


⏭️ [Requêtes de monitoring (cache hit ratio, slow queries)](/annexes/requetes-sql-reference/02-requetes-monitoring.md)
