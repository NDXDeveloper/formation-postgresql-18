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

#### Nouvelles colonnes (PostgreSQL 18)

| Colonne | Description |
|---------|-------------|
| `vacuum_count` | Nombre total de VACUUM manuels terminés |
| `autovacuum_count` | Nombre total d'autovacuums terminés |
| `analyze_count` | Nombre total d'ANALYZE manuels terminés |
| `autoanalyze_count` | Nombre total d'autoanalyzes terminés |
| `vacuum_started` | Nombre de VACUUM lancés (pas forcément terminés) |
| `vacuum_completed` | Nombre de VACUUM terminés avec succès |
| `vacuum_cancelled` | Nombre de VACUUM annulés/interrompus |
| `analyze_started` | Nombre d'ANALYZE lancés |
| `analyze_completed` | Nombre d'ANALYZE terminés avec succès |
| `analyze_cancelled` | Nombre d'ANALYZE annulés/interrompus |

**Note** : Les noms exacts peuvent légèrement varier selon la version finale, mais le principe reste le même.

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
- Avec PG 18, vous pouvez voir si des tentatives ont été faites avec `vacuum_started` vs `vacuum_completed`

### Exemple 2 : Détecter les VACUUM qui échouent

```sql
SELECT
    schemaname,
    relname,
    vacuum_started,
    vacuum_completed,
    vacuum_cancelled,
    (vacuum_started - vacuum_completed) AS vacuum_failures
FROM pg_stat_user_tables  
WHERE (vacuum_started - vacuum_completed) > 0  
ORDER BY vacuum_failures DESC;  
```

**Interprétation** :
- Si `vacuum_started` > `vacuum_completed`, des VACUUM sont lancés mais ne terminent pas
- `vacuum_cancelled` vous dit combien ont été explicitement annulés
- Une différence importante indique un problème (locks, timeout, crashes)

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
    vacuum_started,
    vacuum_completed,
    (vacuum_started - vacuum_completed) AS incomplete_vacuums,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE relname = 'ma_table_problematique';  
```

**Analyse** :
- Si `incomplete_vacuums > 0` : Les VACUUM ne terminent pas (locks, timeout)
- Si `autovacuum_count` est bas : L'autovacuum ne se déclenche pas assez souvent
- Si `n_dead_tup` reste élevé : Les VACUUM ne sont pas efficaces (transactions longues)

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

**Diagnostic avec PG 18** :

```sql
-- Vérifier si des autovacuums sont lancés mais ne terminent pas
SELECT
    schemaname,
    relname,
    last_autovacuum,
    vacuum_started - vacuum_completed AS pending_vacuums,
    autovacuum_count
FROM pg_stat_user_tables  
WHERE last_autovacuum < NOW() - INTERVAL '1 day'  
  AND n_dead_tup > 10000;

-- Vérifier les processus en cours
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    query_start
FROM pg_stat_activity  
WHERE query LIKE '%autovacuum%'  
  AND state != 'idle';
```

**Analyse** :
- Si `pending_vacuums > 0` : Des autovacuums sont lancés mais bloqués
- Vérifier `pg_stat_activity` pour voir si un autovacuum est en cours
- Vérifier `pg_locks` pour identifier les verrous bloquants

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

### Requête 2 : Alertes pour VACUUM qui échouent

```sql
-- Tables où les VACUUM ne terminent jamais
SELECT
    schemaname,
    relname,
    vacuum_started,
    vacuum_completed,
    vacuum_cancelled,
    (vacuum_started - vacuum_completed) AS stuck_vacuums
FROM pg_stat_user_tables  
WHERE (vacuum_started - vacuum_completed) > 3  -- Plus de 3 échecs  
ORDER BY stuck_vacuums DESC;  
```

**Utilisation** : Configurez cette requête dans votre outil de monitoring (Grafana, Prometheus) pour générer une alerte automatique.

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

-- Métrique 2 : Taux de succès des VACUUM
SELECT
    schemaname || '.' || relname AS table_name,
    CASE
        WHEN vacuum_started > 0
        THEN vacuum_completed::float / vacuum_started
        ELSE 1
    END AS vacuum_success_rate
FROM pg_stat_user_tables;

-- Métrique 3 : Temps depuis dernier ANALYZE
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
    vacuum_started - vacuum_completed AS stuck_vacuums
FROM pg_stat_user_tables  
WHERE (vacuum_started - vacuum_completed) > 2  
   OR (n_dead_tup > 10000 AND last_autovacuum < NOW() - INTERVAL '2 days')
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
    vacuum_started,
    vacuum_completed,
    vacuum_cancelled
FROM pg_stat_user_tables  
WHERE relname = 'orders';  

  relname  |     last_autovacuum      | n_dead_tup | autovacuum_count | vacuum_started | vacuum_completed | vacuum_cancelled
-----------+--------------------------+------------+------------------+----------------+------------------+-----------------
 orders    | 2025-11-15 03:22:41.123  |     458932 |               15 |             23 |               15 |                8
```

**Vos réponses** :
- ✅ L'autovacuum a été lancé 23 fois mais n'a terminé que 15 fois  
- ✅ 8 VACUUM ont été annulés ou interrompus  
- ✅ Cela explique pourquoi `n_dead_tup` reste élevé malgré `last_autovacuum` récent  
- ✅ Action : Investiguer pourquoi les VACUUM sont interrompus (locks, timeout)

**Gain** : Diagnostic passant de plusieurs heures à quelques minutes !

---

## Bonnes pratiques et recommandations

### 1. Surveiller régulièrement les métriques clés

**Quotidiennement** :
- Tables avec `(vacuum_started - vacuum_completed) > 0`
- Tables avec `dead_ratio > 20%`
- Tables avec `time_since_analyze > 7 jours` et `n_live_tup > 100000`

**Hebdomadairement** :
- Tendances d'évolution du nombre de lignes mortes
- Fréquence moyenne d'autovacuum par table
- Tables avec grosse croissance de `n_tup_ins`

### 2. Configurer des alertes

Utilisez votre outil de monitoring pour déclencher des alertes sur :

```sql
-- Alerte Critique : VACUUM échoue systématiquement
(vacuum_started - vacuum_completed) >= 5

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

Quand vous identifiez une anomalie (ex: `vacuum_cancelled` élevé), documentez :
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

Les statistiques dans `pg_stat_all_tables` sont **cumulatives** et persistent entre les redémarrages (si `stats_temp_directory` est configuré).

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
    vacuum_started,
    vacuum_completed,
    vacuum_cancelled,
    (vacuum_started - vacuum_completed) AS vacuum_failures,
    analyze_started,
    analyze_completed,
    (analyze_started - analyze_completed) AS analyze_failures,
    last_autovacuum,
    last_autoanalyze,
    EXTRACT(EPOCH FROM (NOW() - last_autovacuum))/3600 AS hours_since_vacuum,
    EXTRACT(EPOCH FROM (NOW() - last_autoanalyze))/3600 AS hours_since_analyze
FROM pg_stat_user_tables  
WHERE n_live_tup > 100;  

-- Utilisation
SELECT * FROM v_vacuum_analyze_health  
WHERE vacuum_failures > 0 OR dead_pct > 10  
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
        vacuum_started - vacuum_completed AS vacuum_failures,
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
3. Les nouvelles colonnes (`vacuum_started`, `vacuum_completed`, `vacuum_cancelled`) donnent une visibilité sans précédent  
4. Surveillez le ratio `vacuum_failures / vacuum_started` et le `dead_ratio`  
5. Combinez ces métriques avec `pg_stat_activity` et `pg_locks` pour un diagnostic complet

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
