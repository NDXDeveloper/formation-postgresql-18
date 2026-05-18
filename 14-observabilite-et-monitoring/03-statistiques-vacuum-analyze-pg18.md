🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3. Nouveauté PG 18 : Statistiques de VACUUM et ANALYZE dans pg_stat_all_tables

## Introduction

PostgreSQL 18, sorti en septembre 2025, apporte une amélioration significative dans la visibilité des opérations de maintenance de votre base de données. Cette nouveauté concerne deux opérations essentielles : **VACUUM** et **ANALYZE**.

Avant de plonger dans les nouveautés, prenons le temps de comprendre ce que sont ces opérations et pourquoi elles sont cruciales pour la santé de votre base de données.

### Rappel pour débutants : Qu'est-ce que VACUUM et ANALYZE ?

#### VACUUM : Le "nettoyeur" de PostgreSQL

Imaginez votre base de données comme un grand livre où vous écrivez et modifiez constamment des informations. Lorsque vous **supprimez** ou **modifiez** une ligne dans PostgreSQL, la donnée n'est pas immédiatement effacée du disque (c'est lié au système MVCC que nous avons vu précédemment). Les anciennes versions restent "invisibles" mais occupent toujours de l'espace.

**VACUUM** est l'opération qui :
- Récupère cet espace "mort" (les anciennes versions de lignes)
- Permet de réutiliser cet espace pour de nouvelles données
- Prévient un problème critique appelé "transaction ID wraparound"
- Maintient la santé générale de votre base de données

**Analogie simple** : VACUUM est comme sortir les poubelles. Si vous ne le faites jamais, votre maison (votre base de données) va se remplir de déchets.

#### ANALYZE : Le "statisticien" de PostgreSQL

Lorsque PostgreSQL doit exécuter une requête, il doit d'abord créer un **plan d'exécution** : décider dans quel ordre accéder aux tables, quels index utiliser, etc. Pour prendre ces décisions intelligemment, PostgreSQL a besoin de **statistiques** sur vos données :
- Combien de lignes contient chaque table ?
- Quelle est la distribution des valeurs dans chaque colonne ?
- Combien de valeurs distinctes existe-t-il ?

**ANALYZE** est l'opération qui :
- Échantillonne vos données
- Calcule ces statistiques
- Permet au planificateur de requêtes de faire des choix optimaux

**Analogie simple** : ANALYZE est comme faire un inventaire. Sans inventaire à jour, vous ne savez pas vraiment ce que vous avez en stock et vous risquez de prendre de mauvaises décisions.

#### Autovacuum : Le processus automatique

Heureusement, PostgreSQL dispose d'un processus automatique appelé **autovacuum** qui exécute régulièrement VACUUM et ANALYZE sans intervention humaine. C'est un peu comme avoir un robot ménager qui nettoie automatiquement votre maison.

Cependant, il est crucial de **surveiller** que ces opérations se déroulent correctement. C'est exactement là qu'interviennent les nouvelles statistiques de PostgreSQL 18.

---

## Pourquoi ces nouvelles statistiques sont importantes ?

### Le problème avant PostgreSQL 18

Avant PostgreSQL 18, les informations sur VACUUM et ANALYZE étaient limitées :
- Vous saviez **quand** la dernière opération s'était produite
- Mais vous ne saviez pas **comment** elle s'était déroulée
- Impossible de savoir si VACUUM avait été efficace ou s'il avait rencontré des problèmes
- Difficile de comprendre pourquoi une table n'était jamais analysée

**Exemple de frustration** : Vous remarquez qu'une table n'a pas été vacuum depuis longtemps. Pourquoi ? Est-ce parce que l'autovacuum est désactivé ? Parce que la table est trop grosse ? Parce qu'il y a eu des erreurs ? Impossible à savoir facilement.

### La solution PostgreSQL 18

PostgreSQL 18 enrichit la vue `pg_stat_all_tables` avec de **nouvelles colonnes** qui donnent une visibilité complète sur :
- Le nombre de fois où VACUUM et ANALYZE ont été exécutés
- Combien de fois ils ont été **lancés** vs combien de fois ils ont **terminé**
- Les erreurs et interruptions éventuelles
- Des métriques détaillées sur chaque opération

Cela transforme votre capacité à diagnostiquer et résoudre les problèmes de maintenance !

---

## La vue pg_stat_all_tables : Vue d'ensemble

### Qu'est-ce que pg_stat_all_tables ?

`pg_stat_all_tables` est une **vue système** qui fournit des statistiques sur toutes les tables de votre base de données (utilisateur et système). Il existe aussi :
- `pg_stat_user_tables` : Uniquement les tables utilisateur (recommandé pour le monitoring)  
- `pg_stat_sys_tables` : Uniquement les tables système

### Accéder à la vue

C'est très simple :

```sql
SELECT * FROM pg_stat_user_tables;
```

Cette commande retourne une ligne par table avec de nombreuses colonnes statistiques.

### Colonnes principales (avant et après PG 18)

Voici les colonnes les plus importantes :

#### Colonnes existantes (pré-PG 18)

| Colonne | Description |
|---------|-------------|
| `schemaname` | Le schéma de la table |
| `relname` | Le nom de la table |
| `seq_scan` | Nombre de scans séquentiels |
| `idx_scan` | Nombre de scans d'index |
| `n_tup_ins` | Nombre de lignes insérées |
| `n_tup_upd` | Nombre de lignes mises à jour |
| `n_tup_del` | Nombre de lignes supprimées |
| `n_live_tup` | Nombre estimé de lignes vivantes |
| `n_dead_tup` | Nombre estimé de lignes mortes (à nettoyer) |
| `last_vacuum` | Timestamp du dernier VACUUM manuel |
| `last_autovacuum` | Timestamp du dernier autovacuum |
| `last_analyze` | Timestamp du dernier ANALYZE manuel |
| `last_autoanalyze` | Timestamp du dernier autoanalyze |

#### Colonnes pré-existantes liées au VACUUM/ANALYZE

Ces compteurs **existent depuis PG 9.1** (ce ne sont PAS des nouveautés PG 18) :

| Colonne | Description |
|---------|-------------|
| `vacuum_count` | Nombre total de VACUUM manuels exécutés |
| `autovacuum_count` | Nombre total d'autovacuums exécutés |
| `analyze_count` | Nombre total d'ANALYZE manuels exécutés |
| `autoanalyze_count` | Nombre total d'autoanalyses exécutés |

#### Vraies nouvelles colonnes en PostgreSQL 18

PG 18 ajoute **quatre colonnes** mesurant le **temps cumulé** passé en VACUUM et ANALYZE (et non leur nombre, déjà connu) :

| Colonne | Type | Description |
|---------|------|-------------|
| `total_vacuum_time` | `double precision` | Temps total cumulé en VACUUM manuel (millisecondes) |
| `total_autovacuum_time` | `double precision` | Temps total cumulé en VACUUM automatique (ms) |
| `total_analyze_time` | `double precision` | Temps total cumulé en ANALYZE manuel (ms) |
| `total_autoanalyze_time` | `double precision` | Temps total cumulé en ANALYZE automatique (ms) |

> 💡 **L'apport pédagogique de PG 18** : avant, on connaissait la *fréquence* de la maintenance (compteurs `*_count`). Maintenant, on connaît son *coût en temps* (compteurs `total_*_time`). Cela permet de repérer une table qui « coûte cher » à autovacuum même si elle n'est nettoyée qu'occasionnellement.

> ⚠️ **À noter** : il n'existe **pas** de colonnes `vacuum_started`, `vacuum_completed`, `vacuum_cancelled`, `analyze_started`, `analyze_completed`, `analyze_cancelled` en PG 18. Si vous voyez ces noms dans un tutoriel, c'est une erreur.

> ℹ️ **Comment ce temps est-il compté ?** Les compteurs `total_*_time` incluent le **temps de sleep** lié au throttling autovacuum (`autovacuum_vacuum_cost_delay`). Une table avec `avg_ms_par_passe` élevé peut donc être lente parce que la passe **traite beaucoup**, ou parce que `cost_delay` la fait dormir longtemps. Pour distinguer les deux, voir le nouveau paramètre `track_cost_delay_timing` ci-dessous.

### Nouveauté PG 18 : `track_cost_delay_timing`

PG 18 introduit un nouveau paramètre booléen **`track_cost_delay_timing`** (défaut : `off`) qui expose le **temps passé en sleep** dû au cost-based throttling de VACUUM/ANALYZE :

```ini
# postgresql.conf (PG 18+)
track_cost_delay_timing = on   # défaut : off
```

Une fois activé, le temps de sleep apparaît :
- dans `pg_stat_progress_vacuum` et `pg_stat_progress_analyze` (vues de progression en temps réel)
- dans la sortie de `VACUUM VERBOSE` et `ANALYZE VERBOSE`
- dans les logs d'autovacuum quand `log_autovacuum_min_duration` est configuré

> ⚠️ **Coût** : ce paramètre interroge l'horloge système à chaque sleep — sur certaines plateformes le surcoût peut être notable. À activer ponctuellement pour diagnostiquer un autovacuum lent, plutôt que tout le temps.

---

## Utiliser les nouvelles statistiques : Exemples pratiques

### Exemple 1 : Identifier les tables jamais vacuum

```sql
SELECT
    schemaname,
    relname,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables  
WHERE last_autovacuum IS NULL  
  AND n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

**Interprétation** :
- Si `last_autovacuum` est NULL et que `n_dead_tup` est élevé, la table accumule des lignes mortes
- C'est un signe que l'autovacuum ne s'exécute peut-être pas correctement
- Avec PG 18, pour confirmer qu'un worker n'est pas bloqué en cours d'exécution, croisez avec `pg_stat_activity` filtré sur `backend_type = 'autovacuum worker'`

### Exemple 2 : Identifier les VACUUM les plus coûteux (PG 18)

Avec PG 18, le **temps cumulé** par table permet de repérer les tables qui pèsent le plus sur le processus autovacuum :

```sql
SELECT
    schemaname,
    relname,
    autovacuum_count,
    ROUND((total_autovacuum_time / 1000)::numeric, 1) AS total_sec_autovacuum,
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 2)
        AS avg_ms_par_passe,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE autovacuum_count > 0  
ORDER BY total_autovacuum_time DESC NULLS LAST  
LIMIT 20;  
```

**Interprétation** :
- `total_autovacuum_time` cumule **le temps passé** dans tous les autovacuums sur cette table
- `avg_ms_par_passe` indique le coût moyen d'une passe : utile pour repérer les tables où chaque autovacuum prend beaucoup de temps (transactions longues, table très grande, etc.)
- Pour détecter un autovacuum « bloqué » en cours, regardez plutôt `pg_stat_activity` (voir Scénario 3 ci-dessous)

### Exemple 3 : Tables avec beaucoup de lignes mortes malgré des VACUUM

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    autovacuum_count,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE n_dead_tup > 10000  
  AND autovacuum_count > 0
ORDER BY dead_ratio DESC;
```

**Interprétation** :
- Ces tables sont vacuum régulièrement (`autovacuum_count > 0`)
- Mais elles ont toujours beaucoup de lignes mortes (`n_dead_tup` élevé)
- Cela peut indiquer :
  - Des transactions longues qui empêchent le nettoyage
  - Un autovacuum pas assez agressif
  - Une charge d'écriture très élevée

### Exemple 4 : Fréquence d'ANALYZE par table

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    analyze_count,
    autoanalyze_count,
    last_autoanalyze,
    AGE(NOW(), last_autoanalyze) AS time_since_last_analyze
FROM pg_stat_user_tables  
WHERE n_live_tup > 100000  
ORDER BY time_since_last_analyze DESC NULLS FIRST;  
```

**Interprétation** :
- Les grandes tables (`n_live_tup` élevé) devraient être régulièrement analyzed
- Si `time_since_last_analyze` est très ancien, les statistiques sont obsolètes
- Le planificateur risque de faire de mauvais choix d'exécution

### Exemple 5 : Comparer VACUUM manuel vs automatique

```sql
SELECT
    schemaname,
    relname,
    vacuum_count AS manual_vacuums,
    autovacuum_count AS auto_vacuums,
    CASE
        WHEN vacuum_count > autovacuum_count THEN 'Mostly manual'
        WHEN autovacuum_count > vacuum_count THEN 'Mostly automatic'
        ELSE 'Mixed'
    END AS vacuum_strategy
FROM pg_stat_user_tables  
WHERE vacuum_count + autovacuum_count > 0  
ORDER BY (vacuum_count + autovacuum_count) DESC;  
```

**Interprétation** :
- Montre si vous dépendez plus des VACUUM manuels ou de l'autovacuum
- Idéalement, l'autovacuum devrait gérer la plupart des tables
- Trop de VACUUM manuels peut indiquer que l'autovacuum est mal configuré

---

## Scénarios de diagnostic avancés

### Scénario 1 : Table qui gonfle sans cesse (bloat)

**Symptôme** : Une table occupe de plus en plus d'espace disque malgré un volume de données stable.

**Diagnostic avec PG 18** :

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size,
    n_live_tup,
    n_dead_tup,
    autovacuum_count,
    ROUND((total_autovacuum_time / 1000)::numeric, 1) AS sec_total_autovacuum,
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 1)
        AS ms_moyenne_par_passe,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE relname = 'ma_table_problematique';  
```

**Analyse** :
- Si `autovacuum_count` est bas mais `n_dead_tup` est élevé : l'autovacuum ne se déclenche pas assez souvent (seuils `autovacuum_vacuum_scale_factor` / `autovacuum_vacuum_threshold` trop hauts pour cette table)
- Si `autovacuum_count` est élevé **et** `n_dead_tup` reste élevé : chaque passe ne récupère pas les lignes mortes — typiquement une **transaction longue** (snapshot ouvert) qui empêche PostgreSQL de considérer les versions comme « vraiment mortes »
- Si `ms_moyenne_par_passe` est très élevée : l'autovacuum mouline longtemps sans réussir à finir efficacement

**Actions possibles** :
1. Identifier et terminer les transactions longues  
2. Augmenter la fréquence d'autovacuum  
3. Exécuter un VACUUM FULL (attention : bloque la table)

### Scénario 2 : Requêtes lentes après des modifications massives

**Symptôme** : Après un gros import ou des modifications massives, les requêtes deviennent lentes.

**Diagnostic avec PG 18** :

```sql
SELECT
    schemaname,
    relname,
    n_tup_ins + n_tup_upd + n_tup_del AS total_modifications,
    analyze_count,
    autoanalyze_count,
    last_autoanalyze,
    AGE(NOW(), last_autoanalyze) AS time_since_analyze
FROM pg_stat_user_tables  
WHERE n_tup_ins + n_tup_upd + n_tup_del > 100000  
ORDER BY time_since_analyze DESC NULLS FIRST;  
```

**Analyse** :
- Si `time_since_analyze` est ancien : Les statistiques sont obsolètes
- Le planificateur utilise de mauvaises estimations
- Les plans de requête sont sous-optimaux

**Actions** :
```sql
ANALYZE ma_table;  -- Force une mise à jour des statistiques
```

### Scénario 3 : Autovacuum semble bloqué

**Symptôme** : `last_autovacuum` ne se met pas à jour depuis longtemps.

**Diagnostic** :

```sql
-- Tables candidates à autovacuum, mais sans passe récente
SELECT
    schemaname,
    relname,
    last_autovacuum,
    autovacuum_count,
    ROUND((total_autovacuum_time / 1000)::numeric, 1) AS total_sec_autovacuum,
    n_dead_tup
FROM pg_stat_user_tables  
WHERE last_autovacuum < NOW() - INTERVAL '1 day'  
  AND n_dead_tup > 10000;

-- Voir les processus autovacuum réellement en cours
-- (backend_type = 'autovacuum worker' depuis PG 10)
SELECT
    pid,
    backend_type,
    state,
    wait_event_type,
    wait_event,
    query,
    NOW() - query_start AS duree
FROM pg_stat_activity  
WHERE backend_type = 'autovacuum worker';  

-- Voir les verrous qui pourraient bloquer un worker autovacuum
SELECT
    a.pid,
    a.backend_type,
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    a.wait_event
FROM pg_locks l  
JOIN pg_stat_activity a ON a.pid = l.pid  
WHERE a.backend_type = 'autovacuum worker'  
   OR NOT l.granted;
```

**Analyse** :
- `backend_type = 'autovacuum worker'` filtre **précisément** les processus autovacuum (depuis PG 10) — préférable à `query LIKE '%autovacuum%'` qui matche aussi des requêtes utilisateurs sur la table `pg_stat_progress_vacuum`
- Si un worker tourne depuis longtemps avec `wait_event` non nul, il est bloqué (sur un lock le plus souvent)

#### Vue de progression `pg_stat_progress_vacuum`

Cette vue (disponible depuis PG 9.6) montre l'**état d'avancement** d'un VACUUM en cours, ligne par ligne (1 ligne par worker actif).

```sql
SELECT
    p.pid,
    a.usename,
    p.datname,
    p.relid::regclass    AS table_name,
    p.phase,             -- 'scanning heap', 'vacuuming indexes', 'vacuuming heap', ...
    pg_size_pretty(p.heap_blks_total * 8192)  AS heap_size,
    pg_size_pretty(p.heap_blks_scanned * 8192) AS scanned,
    ROUND(100.0 * p.heap_blks_scanned / NULLIF(p.heap_blks_total, 0), 2) AS pct_scanned,
    p.index_vacuum_count,                     -- nombre de cycles d'index complétés
    p.indexes_total,                          -- (PG 17+) total d'index sur la table
    p.indexes_processed,                      -- (PG 17+) index traités dans le cycle courant
    NOW() - a.query_start AS duree
FROM pg_stat_progress_vacuum p  
JOIN pg_stat_activity a ON a.pid = p.pid  
ORDER BY p.pid;  
```

**Interprétation** :
- `phase` permet de savoir *où en est* le VACUUM : `'scanning heap'` (lecture des pages), `'vacuuming indexes'` (nettoyage des index, souvent le plus long), `'vacuuming heap'` (compactage), `'truncating heap'` (libération d'espace en queue de table), etc.
- `pct_scanned` donne une estimation visuelle de l'avancement.
- `indexes_processed` / `indexes_total` (PG 17+) : indique précisément combien d'index ont été traités sur les N à parcourir — particulièrement utile pour estimer le temps restant en phase `'vacuuming indexes'`.
- Si le VACUUM stagne longtemps dans `'vacuuming indexes'`, c'est typiquement une table très indexée — vérifier qu'il n'y a pas d'index inutiles à supprimer.

> 💡 **Nouveauté PG 18 — `delay_time`** : si `track_cost_delay_timing = on`, `pg_stat_progress_vacuum` (et `pg_stat_progress_analyze`) expose une colonne `delay_time` indiquant le temps cumulé passé en sleep dû au cost-based throttling. Permet de répondre à la question « pourquoi mon autovacuum est lent ? » : il *traite* lentement, ou il *dort* beaucoup ? Note : pour les parallel workers, `delay_time` n'est rafraîchi qu'une fois par seconde et peut donc être légèrement obsolète.

> ℹ️ Les autres vues de progression utiles, dans l'ordre des versions :  
> - `pg_stat_progress_cluster` (PG 12+) — pour `CLUSTER` et `VACUUM FULL`  
> - `pg_stat_progress_create_index` (PG 12+) — pour `CREATE INDEX [CONCURRENTLY]` et `REINDEX`  
> - `pg_stat_progress_analyze` (PG 13+) — pour `ANALYZE` (suivi des phases : `acquiring sample rows`, `computing statistics`…)  
> - `pg_stat_progress_basebackup` (PG 13+) — pour le suivi côté serveur d'un `pg_basebackup` distant  
> - `pg_stat_progress_copy` (PG 14+) — pour le suivi d'un `COPY` (lignes traitées, bytes traités)

---

## Surveillance et alerting : Requêtes pratiques

### Requête 1 : Dashboard de santé des tables

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    autovacuum_count,
    autoanalyze_count,
    last_autovacuum,
    last_autoanalyze,
    CASE
        WHEN last_autovacuum < NOW() - INTERVAL '7 days'
         AND n_dead_tup > 10000 THEN '🔴 Critical'
        WHEN last_autovacuum < NOW() - INTERVAL '3 days'
         AND n_dead_tup > 5000 THEN '🟠 Warning'
        ELSE '🟢 OK'
    END AS vacuum_status,
    CASE
        WHEN last_autoanalyze < NOW() - INTERVAL '7 days'
         AND n_live_tup > 100000 THEN '🔴 Critical'
        WHEN last_autoanalyze < NOW() - INTERVAL '3 days'
         AND n_live_tup > 50000 THEN '🟠 Warning'
        ELSE '🟢 OK'
    END AS analyze_status
FROM pg_stat_user_tables  
WHERE n_live_tup > 1000  
ORDER BY dead_ratio DESC NULLS LAST;  
```

### Requête 2 : Alerter sur l'autovacuum trop coûteux ou absent

```sql
-- Tables qui consomment beaucoup de temps autovacuum
-- (PG 18 : grâce à total_autovacuum_time)
SELECT
    schemaname,
    relname,
    autovacuum_count,
    ROUND((total_autovacuum_time / 1000)::numeric, 1) AS total_sec,
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 1)
        AS avg_ms_par_passe
FROM pg_stat_user_tables  
WHERE autovacuum_count > 0  
ORDER BY total_autovacuum_time DESC NULLS LAST  
LIMIT 20;  

-- Tables non vacuumées depuis longtemps malgré beaucoup de lignes mortes
SELECT
    schemaname,
    relname,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum,
    AGE(NOW(), last_autovacuum) AS age_autovacuum
FROM pg_stat_user_tables  
WHERE n_dead_tup > 10000  
  AND (last_autovacuum IS NULL
       OR last_autovacuum < NOW() - INTERVAL '24 hours')
ORDER BY n_dead_tup DESC;
```

**Utilisation** : Configurez ces requêtes dans votre outil de monitoring (Grafana, Prometheus) pour générer des alertes automatiques.

### Requête 3 : Tables nécessitant une attention immédiate

```sql
WITH table_health AS (
    SELECT
        schemaname,
        relname,
        n_live_tup,
        n_dead_tup,
        ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
        autovacuum_count,
        last_autovacuum,
        EXTRACT(EPOCH FROM (NOW() - last_autovacuum))/3600 AS hours_since_vacuum
    FROM pg_stat_user_tables
    WHERE n_live_tup > 0
)
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    dead_ratio,
    ROUND(hours_since_vacuum, 1) AS hours_since_vacuum,
    autovacuum_count,
    CASE
        WHEN dead_ratio > 20 AND hours_since_vacuum > 24 THEN 'Manual VACUUM recommended'
        WHEN dead_ratio > 40 THEN 'Manual VACUUM URGENT'
        WHEN hours_since_vacuum > 168 THEN 'Check autovacuum config'
        ELSE 'OK'
    END AS recommendation
FROM table_health  
WHERE dead_ratio > 10 OR hours_since_vacuum > 48  
ORDER BY dead_ratio DESC;  
```

---

## Intégration avec les outils de monitoring

### Grafana + Prometheus

Voici un exemple de métriques à exporter pour Grafana :

```sql
-- Métrique 1 : Ratio de lignes mortes par table
SELECT
    schemaname || '.' || relname AS table_name,
    COALESCE(n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0), 0) AS dead_tuple_ratio
FROM pg_stat_user_tables;

-- Métrique 2 (PG 18) : Coût moyen d'une passe d'autovacuum
SELECT
    schemaname || '.' || relname AS table_name,
    CASE
        WHEN autovacuum_count > 0
        THEN total_autovacuum_time / autovacuum_count
        ELSE NULL
    END AS avg_autovacuum_ms
FROM pg_stat_user_tables;

-- Métrique 3 : Temps depuis dernier ANALYZE (en secondes)
SELECT
    schemaname || '.' || relname AS table_name,
    EXTRACT(EPOCH FROM (NOW() - last_autoanalyze)) AS seconds_since_analyze
FROM pg_stat_user_tables;
```

### pgBadger

pgBadger peut analyser vos logs pour corréler les événements VACUUM/ANALYZE avec les performances. Avec PostgreSQL 18, vous pouvez maintenant :
- Croiser les statistiques de pg_stat_all_tables avec les logs
- Identifier les patterns (ex: dégradation après échec d'ANALYZE)

### Scripts de monitoring personnalisés

Voici un exemple de script shell pour vérifier la santé quotidienne :

```bash
#!/bin/bash
# check_vacuum_health.sh

psql -U postgres -d ma_base -t -c "  
SELECT  
    relname,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count,
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 1) AS avg_ms_passe
FROM pg_stat_user_tables  
WHERE n_dead_tup > 10000  
  AND (last_autovacuum IS NULL
       OR last_autovacuum < NOW() - INTERVAL '2 days')
" | while read line; do
    if [ ! -z "$line" ]; then
        echo "ALERT: Table with vacuum issues: $line"
        # Envoyer une notification (email, Slack, etc.)
    fi
done
```

---

## Comparaison : Avant vs Après PostgreSQL 18

### Scénario : Une table a des problèmes de performance

#### Avant PostgreSQL 18

**Ce que vous voyiez** :
```sql
SELECT relname, last_autovacuum, n_dead_tup  
FROM pg_stat_user_tables  
WHERE relname = 'orders';  

  relname  |     last_autovacuum      | n_dead_tup
-----------+--------------------------+------------
 orders    | 2025-11-15 03:22:41.123  |     458932
```

**Vos questions sans réponse** :
- ❓ Pourquoi `n_dead_tup` est si élevé alors qu'un autovacuum a tourné ?  
- ❓ L'autovacuum a-t-il vraiment terminé ou a-t-il été interrompu ?  
- ❓ Combien de fois l'autovacuum a-t-il tenté de nettoyer cette table ?

#### Après PostgreSQL 18

**Ce que vous voyez maintenant** :
```sql
SELECT
    relname,
    last_autovacuum,
    n_dead_tup,
    autovacuum_count,
    ROUND((total_autovacuum_time / 1000)::numeric, 1)               AS sec_total,
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 1)
        AS ms_moyenne_par_passe
FROM pg_stat_user_tables  
WHERE relname = 'orders';  

  relname  |     last_autovacuum      | n_dead_tup | autovacuum_count | sec_total | ms_moyenne_par_passe
-----------+--------------------------+------------+------------------+-----------+----------------------
 orders    | 2025-11-15 03:22:41.123  |     458932 |               15 |     487.3 |              32480.0
```

**Vos réponses** :
- ✅ L'autovacuum a tourné 15 fois et a consommé **487 s au total** (environ 32 s par passe en moyenne)  
- ✅ Si `n_dead_tup` reste élevé malgré ces 15 passes, le problème n'est pas une absence d'autovacuum mais probablement une **transaction longue** qui empêche le nettoyage effectif (`xmin horizon` ancien)  
- ✅ Action : croisez avec `pg_stat_activity` pour repérer les sessions à `xact_start` très ancien

**Gain** : on connaît désormais le **coût en temps** de la maintenance par table, pas seulement sa fréquence — utile pour planifier les fenêtres et identifier les tables « gourmandes ».

---

## Bonnes pratiques et recommandations

### 1. Surveiller régulièrement les métriques clés

**Quotidiennement** :
- Tables avec `dead_ratio > 20%` malgré un `last_autovacuum` récent (signe de transaction longue)
- Tables avec `time_since_analyze > 7 jours` et `n_live_tup > 100000`
- Tables où `total_autovacuum_time` croît anormalement vite (passes plus longues que d'habitude)

**Hebdomadairement** :
- Tendances d'évolution du nombre de lignes mortes
- Fréquence moyenne d'autovacuum par table
- Tables avec grosse croissance de `n_tup_ins`

### 2. Configurer des alertes

Utilisez votre outil de monitoring pour déclencher des alertes sur :

```sql
-- Alerte Critique : autovacuum « coûteux » et inefficace
-- (passe moyenne > 30 secondes ET lignes mortes encore élevées)
autovacuum_count > 0
  AND total_autovacuum_time / autovacuum_count > 30000  -- ms
  AND n_dead_tup > 50000

-- Alerte Warning : Table pas vacuum depuis longtemps
last_autovacuum < NOW() - INTERVAL '3 days' AND n_dead_tup > 10000

-- Alerte Info : ANALYZE obsolète
last_autoanalyze < NOW() - INTERVAL '7 days' AND n_live_tup > 100000
```

### 3. Corréler avec d'autres métriques

Les statistiques de VACUUM/ANALYZE sont plus utiles quand combinées avec :
- `pg_stat_activity` : Transactions longues bloquant le VACUUM  
- `pg_locks` : Verrous empêchant les opérations de maintenance  
- `pg_stat_statements` : Requêtes lentes après des statistiques obsolètes
- Métriques système : I/O, CPU pendant les VACUUM

### 4. Documentation des anomalies

Quand vous identifiez une anomalie (ex: `total_autovacuum_time` qui explose), documentez :
- La requête de diagnostic utilisée
- Les valeurs observées
- Les actions correctives prises
- Le résultat

Cela créera une base de connaissance pour l'équipe.

### 5. Ajuster la configuration autovacuum si nécessaire

Si vous voyez systématiquement des problèmes, ajustez :

```sql
-- Pour une table spécifique, rendre l'autovacuum plus agressif
ALTER TABLE ma_table SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- Par défaut: 0.2
    autovacuum_analyze_scale_factor = 0.05  -- Par défaut: 0.1
);

-- Augmenter le nombre de workers autovacuum (postgresql.conf)
autovacuum_max_workers = 4;  -- Par défaut: 3
```

### 6. Accélérer un VACUUM manuel : `vacuum_buffer_usage_limit` (PG 16+)

Quand on lance un `VACUUM` manuel sur une grosse table en fenêtre de maintenance, on veut souvent que ça finisse **vite**. Le frein historique : PostgreSQL utilise par défaut un anneau de buffers très petit (256 kB) pour VACUUM, ce qui le ralentit énormément sur les grosses tables.

PG 16 introduit **`vacuum_buffer_usage_limit`** qui permet d'augmenter cet anneau :

```ini
# postgresql.conf — globalement
vacuum_buffer_usage_limit = 256MB    # défaut : 2 MB
# vacuum_buffer_usage_limit = 0      # sans limite (consomme librement shared_buffers)
```

Ou ponctuellement, sans modifier le serveur :

```sql
-- À la volée sur une commande précise (PG 16+)
VACUUM (BUFFER_USAGE_LIMIT '512MB') ma_grosse_table;  
ANALYZE (BUFFER_USAGE_LIMIT '256MB') ma_grosse_table;  
```

**Effet** : un VACUUM qui prenait 4 h en peut prendre 1 h ou moins, au prix d'un peu de cache éviction.

> ⚠️ Plafond automatique à `shared_buffers / 8` (pour ne pas écraser le cache d'autres requêtes). En production, **2 MB est trop bas** pour les grosses tables — passer à 64-256 MB est un gain immédiat sans risque significatif.

---

## Surveillance du Transaction ID Wraparound

### Qu'est-ce que le wraparound ?

PostgreSQL identifie chaque transaction par un **Transaction ID (XID)** stocké sur **32 bits**, soit ~4,3 milliards de valeurs possibles. Au-delà, les XID « bouclent » (wraparound) — et comme PostgreSQL utilise la comparaison d'XID pour décider si une ligne est visible, un wraparound non maîtrisé entraînerait des **données qui semblent disparaître**.

Pour éviter cela, VACUUM marque les vieilles transactions comme « gelées » (`FROZEN`) : leur XID est remplacé par une marque spéciale qui dit « toujours visible ». L'XID est ainsi recyclable.

**Si ce processus prend du retard**, PostgreSQL passe par plusieurs stades de protection :

| Stade | Distance avant wraparound | Comportement |
|-------|---------------------------|--------------|
| Normal | > 200 M | Autovacuum standard |
| Anti-wraparound auto | < ~2,1 G (`autovacuum_freeze_max_age` atteint) | autovacuum se relance même si la table a été désactivée |
| **Failsafe** (PG 14+) | `age(relfrozenxid)` > `vacuum_failsafe_age` (défaut 1,6 G) | VACUUM ignore le cost-based throttling et fonce |
| ⚠️ Warning | < 40 M | Messages `WARNING` dans les logs |
| 🛑 Mode read-only | < 3 M (`age` > ~2,143 G) | **PostgreSQL refuse toute nouvelle transaction qui écrit** : `ERROR: database is not accepting commands that assign new XIDs to avoid wraparound data loss` |

### Mesurer la proximité du wraparound

**Niveau base de données** :

```sql
SELECT
    datname,
    age(datfrozenxid)            AS xid_age,
    mxid_age(datminmxid)         AS mxid_age,
    ROUND(100.0 * age(datfrozenxid) / 2147483648, 2) AS pct_wraparound
FROM pg_database  
ORDER BY age(datfrozenxid) DESC;  
```

**Interprétation de `pct_wraparound`** (avec wraparound théorique à 2³¹ ≈ 2,15 G) :

| Pct | État | Action |
|-----|------|--------|
| < 30 % | 🟢 Normal | RAS |
| 30 – 50 % | 🟡 À surveiller | Vérifier que l'autovacuum tourne bien |
| 50 – 70 % | 🟠 Préoccupant | Lancer un `VACUUM` manuel pendant les heures creuses |
| > 70 % | 🔴 Critique | Action immédiate : voir « Procédure d'urgence » ci-dessous |

**Niveau table** — trouver les tables qui retiennent le plus l'horizon `xmin` :

```sql
SELECT
    c.oid::regclass AS table_name,
    age(c.relfrozenxid) AS xid_age,
    pg_size_pretty(pg_table_size(c.oid)) AS taille,
    -- Inclut aussi la TOAST table associée
    GREATEST(
        age(c.relfrozenxid),
        age(COALESCE(t.relfrozenxid, c.relfrozenxid))
    ) AS xid_age_with_toast
FROM pg_class c  
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid  
WHERE c.relkind IN ('r', 'm')          -- tables + matviews  
  AND c.relnamespace NOT IN (
        SELECT oid FROM pg_namespace
        WHERE nspname IN ('pg_catalog', 'information_schema')
      )
ORDER BY xid_age_with_toast DESC  
LIMIT 20;  
```

> 💡 **Nouveauté PG 18 — `pg_class.relallfrozen`** : une nouvelle colonne PG 18 indique le nombre de pages **déjà gelées** dans la visibility map de chaque table. Combinée à `relallvisible` (pages visibles à tous) et `relpages` (pages totales), elle permet de mesurer la **progression** du gel et de cibler les tables qui ont encore beaucoup de pages à geler :  
>  
> ```sql  
> SELECT  
>     c.oid::regclass AS table_name,  
>     c.relpages,  
>     c.relallvisible,  
>     c.relallfrozen,                                                    -- (PG 18+)  
>     c.relpages - c.relallfrozen AS pages_a_geler,  
>     ROUND(100.0 * c.relallfrozen / NULLIF(c.relpages, 0), 1) AS pct_frozen,  
>     age(c.relfrozenxid) AS xid_age  
> FROM pg_class c  
> WHERE c.relkind IN ('r', 'm')  
>   AND c.relpages > 1000                  -- filtre les petites tables  
>   AND c.relnamespace NOT IN (  
>         SELECT oid FROM pg_namespace  
>         WHERE nspname IN ('pg_catalog', 'information_schema')  
>       )  
> ORDER BY pages_a_geler DESC  
> LIMIT 20;  
> ```  
>  
> ⚠️ `relallvisible` et `relallfrozen` sont des **estimations** mises à jour par VACUUM et ANALYZE — pas en temps réel.

### Paramètres clés

| Paramètre | Défaut | Rôle |
|-----------|--------|------|
| `autovacuum_freeze_max_age` | 200 M | Seuil au-delà duquel autovacuum **force** un vacuum anti-wraparound, même `autovacuum_enabled = off` |
| `vacuum_freeze_min_age` | 50 M | Une ligne dont l'XID est plus vieux que ça est éligible au gel |
| `vacuum_freeze_table_age` | 150 M | Force un scan complet de la table au prochain VACUUM |
| `vacuum_failsafe_age` | 1,6 G | (PG 14+) VACUUM bascule en mode urgence (skip cost-delay, skip index cleanup) |
| `autovacuum_multixact_freeze_max_age` | 400 M | Idem pour les MultiXact IDs |
| `vacuum_multixact_failsafe_age` | 1,6 G | (PG 14+) Failsafe MultiXact |

### Procédure d'urgence si `pct_wraparound > 70 %`

L'ordre des actions est important — chaque étape libère l'horizon `xmin` pour permettre aux VACUUM suivants de finalement geler les anciennes transactions :

```sql
-- 1) Transactions préparées (2PC) oubliées : elles bloquent l'horizon xmin
SELECT * FROM pg_prepared_xacts ORDER BY prepared;
-- Si oublié : ROLLBACK PREPARED 'gid' ou COMMIT PREPARED 'gid'

-- 2) Transactions longues ou idle in transaction
SELECT pid, age(backend_xmin) AS xmin_age, state,
       NOW() - xact_start AS xact_age, query
FROM pg_stat_activity  
WHERE backend_xmin IS NOT NULL  
ORDER BY age(backend_xmin) DESC;  
-- pg_terminate_backend(pid) pour les sessions vraiment bloquantes

-- 3) Replication slots qui retiennent l'horizon (xmin ou catalog_xmin)
SELECT slot_name, active, age(xmin) AS xmin_age,
       age(catalog_xmin) AS catalog_xmin_age
FROM pg_replication_slots  
WHERE xmin IS NOT NULL OR catalog_xmin IS NOT NULL  
ORDER BY GREATEST(age(xmin), age(catalog_xmin)) DESC;  
-- pg_drop_replication_slot('xxx') sur les slots inactifs et inutiles

-- 4) Une fois l'horizon libéré, lancer un VACUUM normal sur la base
VACUUM (VERBOSE, ANALYZE);
-- ⚠️ Ne PAS faire VACUUM FULL (réécrit la table, génère lui-même des XID, et prend AccessExclusiveLock)
-- ⚠️ L'option FREEZE (`VACUUM FREEZE`) force le gel de TOUTES les lignes, même celles plus récentes
--    que `vacuum_freeze_min_age`. Inutile en cas de wraparound : un VACUUM normal en mode
--    agressif gèle déjà tout ce qui est nécessaire pour repousser l'horizon, sans effort superflu.
```

> ℹ️ **PG 18 — single-user mode** : il n'est plus nécessaire de redémarrer en mode mono-utilisateur (`postgres --single`) pour récupérer d'un wraparound. Les `VACUUM` peuvent tourner en mode multi-utilisateur normal, même quand le serveur est passé en lecture seule pour cause de wraparound imminent.

### Alerter en amont

```sql
-- Alerte (exemple Grafana / Prometheus)
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    CASE
        WHEN age(datfrozenxid) > 1.9e9 THEN '🔴 CRITICAL'  -- < 250 M restants
        WHEN age(datfrozenxid) > 1.5e9 THEN '🟠 WARNING'    -- < 650 M restants
        WHEN age(datfrozenxid) > 1.0e9 THEN '🟡 NOTICE'     -- < 1,15 G restants
        ELSE '🟢 OK'
    END AS status
FROM pg_database  
WHERE datname NOT IN ('template0');   -- template0 est figée  
```

> 💡 **Le wraparound est l'un des seuls scénarios pouvant rendre PostgreSQL totalement indisponible**. C'est la métrique la plus importante à surveiller — bien plus prioritaire que le cache hit ratio ou le bloat. Un monitoring sérieux doit alerter dès `age(datfrozenxid) > 1,5 milliard`.

---

## Limites et considérations

### Ce que les statistiques ne disent PAS

Ces métriques sont puissantes, mais elles ne capturent pas tout :

1. **Pourquoi un VACUUM a été annulé** : Vous voyez qu'il y a eu annulation, mais pas la cause (lock, timeout, erreur)  
2. **Détails de performance** : Combien de temps a duré chaque VACUUM, combien de pages ont été nettoyées  
3. **Historique détaillé** : Les compteurs sont cumulatifs, pas horodatés individuellement

Pour ces informations, vous devez consulter :
- Les logs PostgreSQL (configurez `log_autovacuum_min_duration`)
- Les vues en temps réel (`pg_stat_progress_vacuum`, `pg_stat_activity`)

### Réinitialisation des statistiques

Les statistiques dans `pg_stat_all_tables` sont **cumulatives**.

> ℹ️ **Évolution PG 15+** : depuis PostgreSQL 15, le collecteur de statistiques traditionnel a été remplacé par un système basé sur la **mémoire partagée** (commit 5891c7a8e). Le paramètre `stats_temp_directory` n'existe plus. Les statistiques sont automatiquement persistées sur disque lors d'un arrêt normal du serveur (dans `pg_stat/`) et rechargées au redémarrage. Un arrêt brutal (crash) entraîne en revanche leur perte.

Pour réinitialiser manuellement :

```sql
-- Réinitialiser les statistiques d'une table spécifique
SELECT pg_stat_reset_single_table_counters('ma_table'::regclass);

-- Réinitialiser toutes les statistiques (ATTENTION !)
SELECT pg_stat_reset();
```

**⚠️ Attention** : Réinitialiser les statistiques efface l'historique. Ne le faites qu'en connaissance de cause.

### Impact sur les performances

La collecte de ces statistiques a un **impact négligeable** sur les performances :
- Les compteurs sont incrémentés en mémoire
- Pas d'I/O supplémentaire
- Overhead < 0.1%

---

## Exemples de requêtes prêtes à l'emploi

### Vue synthétique pour monitoring quotidien

```sql
CREATE OR REPLACE VIEW v_vacuum_analyze_health AS  
SELECT  
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    autovacuum_count,
    autoanalyze_count,
    -- ⭐ Nouveau en PG 18 : temps cumulés (en secondes)
    ROUND((total_autovacuum_time / 1000)::numeric, 1)  AS sec_total_autovacuum,
    ROUND((total_autoanalyze_time / 1000)::numeric, 1) AS sec_total_autoanalyze,
    -- ⭐ Coût moyen par passe
    ROUND((total_autovacuum_time / NULLIF(autovacuum_count, 0))::numeric, 1)
        AS ms_moyen_autovacuum,
    last_autovacuum,
    last_autoanalyze,
    EXTRACT(EPOCH FROM (NOW() - last_autovacuum))/3600  AS hours_since_vacuum,
    EXTRACT(EPOCH FROM (NOW() - last_autoanalyze))/3600 AS hours_since_analyze
FROM pg_stat_user_tables  
WHERE n_live_tup > 100;  

-- Utilisation : tables au-dessus du seuil de bloat
SELECT * FROM v_vacuum_analyze_health  
WHERE dead_pct > 10  
ORDER BY dead_pct DESC;  
```

### Identifier les tables "à risque"

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    autovacuum_count,
    ROUND(EXTRACT(EPOCH FROM (NOW() - last_autovacuum))/3600, 1) AS hours_since_vacuum,
    CASE
        WHEN n_dead_tup > 100000 AND dead_pct > 20 THEN 'CRITICAL - Manual VACUUM needed'
        WHEN n_dead_tup > 50000 AND dead_pct > 15 THEN 'WARNING - Monitor closely'
        WHEN hours_since_vacuum > 48 THEN 'INFO - Check autovacuum config'
        ELSE 'OK'
    END AS risk_level
FROM pg_stat_user_tables  
WHERE n_live_tup > 1000  
ORDER BY n_dead_tup DESC  
LIMIT 20;  
```

### Export pour monitoring externe (format JSON)

```sql
SELECT json_agg(t)  
FROM (  
    SELECT
        schemaname,
        relname,
        n_dead_tup,
        autovacuum_count,
        ROUND(total_autovacuum_time::numeric, 1) AS total_autovacuum_ms,
        EXTRACT(EPOCH FROM (NOW() - last_autovacuum)) AS seconds_since_vacuum
    FROM pg_stat_user_tables
    WHERE n_live_tup > 1000
) t;
```

---

## Conclusion

Les nouvelles statistiques de VACUUM et ANALYZE dans PostgreSQL 18 représentent une **amélioration majeure** pour l'observabilité et la maintenance des bases de données.

### Ce que vous gagnez avec PostgreSQL 18

✅ **Visibilité complète** : Savoir non seulement QUAND mais COMMENT les opérations se sont déroulées

✅ **Diagnostic rapide** : Identifier immédiatement les tables problématiques

✅ **Proactivité** : Détecter les problèmes avant qu'ils n'impactent les performances

✅ **Monitoring automatisé** : Créer des alertes précises basées sur des métriques fiables

✅ **Optimisation ciblée** : Ajuster la configuration autovacuum table par table en connaissance de cause

### Points clés à retenir

1. **VACUUM** récupère l'espace et prévient le wraparound
2. **ANALYZE** met à jour les statistiques pour le planificateur
3. Les vraies nouvelles colonnes PG 18 sont `total_vacuum_time`, `total_autovacuum_time`, `total_analyze_time`, `total_autoanalyze_time` — elles mesurent le **temps cumulé** en maintenance, en complément des compteurs `*_count` qui existaient déjà
4. Surveillez le **temps moyen par passe** (`total_autovacuum_time / autovacuum_count`) en plus du `dead_ratio`
5. Combinez ces métriques avec `pg_stat_activity` (filtre `backend_type = 'autovacuum worker'`) et `pg_locks` pour identifier les workers bloqués

### Prochaines étapes recommandées

1. **Mettre à jour vers PostgreSQL 18** dès que possible pour bénéficier de ces statistiques  
2. **Créer des vues personnalisées** adaptées à vos besoins de monitoring  
3. **Configurer des alertes** dans votre outil de monitoring (Grafana, Prometheus, Datadog)  
4. **Former l'équipe** à interpréter ces nouvelles métriques  
5. **Documenter** les seuils d'alerte spécifiques à votre contexte

### Ressources complémentaires

- Documentation officielle : [PostgreSQL 18 Release Notes - Monitoring Enhancements](https://www.postgresql.org/docs/18/monitoring-stats.html)
- Article : "Understanding VACUUM and ANALYZE in PostgreSQL"
- Wiki : "Autovacuum Tuning Guide"

---

**💡 Conseil final** : Commencez par surveiller ces métriques pendant quelques semaines sans intervenir. Observez les patterns normaux de votre base. Cela vous permettra de définir des seuils d'alerte pertinents et d'éviter les fausses alertes.

**🎯 Objectif ultime** : Zéro surprise. Vous devez être alerté d'un problème potentiel avant qu'il n'impacte vos utilisateurs !

⏭️ [Nouveauté PG 18 : Statistiques I/O et WAL par backend](/14-observabilite-et-monitoring/04-statistiques-io-wal-backend-pg18.md)
