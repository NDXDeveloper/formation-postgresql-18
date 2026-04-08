🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.11. Maintenance des Index : REINDEX, VACUUM FULL

## Introduction

Les index, comme toute structure de données, se **dégradent** avec le temps. Les opérations d'écriture (INSERT, UPDATE, DELETE) créent progressivement des **espaces vides** (bloat) et de la **fragmentation**, réduisant les performances. La maintenance régulière des index est donc essentielle.

**Analogie** : Imaginez une bibliothèque. Au fil du temps, les livres sont déplacés, certains sont retirés, laissant des trous dans les étagères. Périodiquement, il faut réorganiser la bibliothèque pour optimiser l'espace et faciliter la recherche. C'est exactement ce que font `REINDEX` et `VACUUM FULL` pour PostgreSQL.

**Ce que vous apprendrez** :
- Pourquoi les index se dégradent
- Quand et comment utiliser REINDEX
- Différences entre VACUUM et VACUUM FULL
- Stratégies de maintenance sans downtime
- Monitoring du bloat
- Automatisation

**Prérequis** : Avoir lu les sections 13.1 à 13.3 sur les index et les stratégies de scan.

---

## 1. Comprendre le Bloat et la Fragmentation

### 1.1. Qu'est-ce que le Bloat ?

Le **bloat** (gonflement) est l'accumulation d'**espace inutilisé** dans les index et les tables.

**Causes** :

#### Cause 1 : DELETE

```sql
-- Création d'une table avec index
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    prix NUMERIC
);
CREATE INDEX idx_produits_prix ON produits(prix);

-- Insertion de 10,000 lignes
INSERT INTO produits SELECT i, 'Produit ' || i, i * 10 FROM generate_series(1, 10000) i;

-- Suppression de 5,000 lignes
DELETE FROM produits WHERE id <= 5000;
```

**Que se passe-t-il ?**

Dans l'index `idx_produits_prix` :
```
Avant DELETE :  
Page 1 : [Prix: 10 → TID1, Prix: 20 → TID2, Prix: 30 → TID3, ...]  
Page 2 : [Prix: 50 → TID5, Prix: 60 → TID6, ...]  
...

Après DELETE :  
Page 1 : [VIDE, VIDE, VIDE, Prix: 40 → TID4, ...]   ← Espaces morts  
Page 2 : [VIDE, Prix: 60 → TID6, ...]                ← Espaces morts  
...
```

**Résultat** : Les pages d'index contiennent beaucoup d'espaces vides, mais l'index reste de la même taille sur disque.

#### Cause 2 : UPDATE

Les UPDATE dans PostgreSQL fonctionnent comme **DELETE + INSERT** en raison du MVCC (Multi-Version Concurrency Control).

```sql
-- Update de 5,000 lignes
UPDATE produits SET prix = prix * 1.1 WHERE id > 5000;
```

**Effet** :
- Anciennes versions marquées comme mortes dans l'index
- Nouvelles versions ajoutées à l'index
- Bloat accumulé

#### Cause 3 : Fragmentation

Avec le temps, les entrées d'index sont insérées dans un ordre non optimal :

```
Index B-Tree idéal (séquentiel) :  
Page 1 : [1, 2, 3, 4]  
Page 2 : [5, 6, 7, 8]  
Page 3 : [9, 10, 11, 12]  

Index fragmenté (après beaucoup d'INSERT/DELETE) :  
Page 1 : [1, 7, 12]  
Page 2 : [2, 9]  
Page 3 : [3, 4, 5, 6, 8, 10, 11]   ← Page "chaude" (overused)  
Page 4 : []                         ← Page vide  
```

**Conséquence** : Plus de pages à lire pour la même quantité de données = performances dégradées.

### 1.2. Impact du Bloat

#### Sur les Performances

**Exemple** : Index de 100 MB avec 40% de bloat

| Métrique | Sans bloat | Avec 40% bloat | Impact |
|----------|------------|----------------|--------|
| Taille disque | 100 MB | 140 MB | +40% |
| Pages à lire | 12,500 | 17,500 | +40% |
| Cache Hit Ratio | 98% | 85% | -13% |
| Temps Index Scan | 50 ms | 85 ms | +70% |

**Analyse** : Un index bloaté de 40% peut être **70% plus lent** !

#### Sur l'Espace Disque

```sql
-- Vérifier la taille d'un index
SELECT
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes  
WHERE schemaname = 'public'  
ORDER BY pg_relation_size(indexrelid) DESC;  
```

**Résultat avec bloat** :
```
 index_name           | index_size
----------------------|------------
 idx_orders_date      | 450 MB      (devrait être 280 MB)
 idx_clients_email    | 320 MB      (devrait être 210 MB)
```

### 1.3. Détecter le Bloat

#### Méthode 1 : Extension pgstattuple

```sql
-- Installation
CREATE EXTENSION pgstattuple;

-- Analyser un index
SELECT * FROM pgstatindex('idx_produits_prix');
```

**Résultat** :
```
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages | empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
---------|------------|------------|---------------|----------------|------------|-------------|---------------|------------------|-------------------
 4       | 2          | 147456     | 3             | 5              | 180        | 10          | 45            | 65.23            | 28.5
```

**Colonnes importantes** :
- `avg_leaf_density` : Densité moyenne des pages feuilles (idéal > 90%)  
- `leaf_fragmentation` : Fragmentation (idéal < 10%)  
- `deleted_pages` : Pages mortes

**Calcul du bloat** :
```sql
SELECT
    indexrelname,
    round(100 * (1 - avg_leaf_density / 100), 1) AS bloat_pct
FROM pgstatindex('idx_produits_prix')  
JOIN pg_stat_user_indexes ON indexrelid = indexrelid;  
```

**Seuil d'alerte** :
- Bloat < 20% → ✅ OK
- Bloat 20-40% → ⚠️ À surveiller
- Bloat > 40% → 🔴 REINDEX recommandé

#### Méthode 2 : Requête Approximative (Sans Extension)

```sql
SELECT
    schemaname,
    tablename,
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes  
WHERE schemaname = 'public'  
  AND idx_scan > 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Indicateur de bloat** : Si `tuples_read` >> `tuples_fetched`, l'index est inefficace (possiblement bloaté).

---

## 2. REINDEX : Reconstruire les Index

### 2.1. Qu'est-ce que REINDEX ?

`REINDEX` reconstruit complètement un index en :
1. Créant un **nouvel index** depuis zéro  
2. Supprimant l'**ancien index** bloaté  
3. Renommant le nouvel index

**Résultat** : Index compact, sans bloat, performant.

### 2.2. Syntaxe et Variantes

#### REINDEX INDEX : Un index spécifique

```sql
REINDEX INDEX idx_produits_prix;
```

**⚠️ Attention** : Pose un **verrou exclusif** (EXCLUSIVE LOCK) sur la table !
- Les SELECT continuent
- Les INSERT/UPDATE/DELETE sont **bloqués**

**Durée** : Proportionnelle à la taille de l'index (ex: 10 GB = 5-15 minutes).

#### REINDEX TABLE : Tous les index d'une table

```sql
REINDEX TABLE produits;
```

Reconstruit :
- Index PRIMARY KEY
- Tous les index secondaires
- Index UNIQUE

#### REINDEX SCHEMA : Tous les index d'un schéma

```sql
REINDEX SCHEMA public;
```

**⚠️ Attention** : Très intrusif, peut bloquer toute l'application !

#### REINDEX DATABASE : Toute la base

```sql
REINDEX DATABASE mabase;
```

**Usage** : Très rare, seulement en maintenance planifiée.

#### REINDEX SYSTEM : Index système

```sql
REINDEX SYSTEM mabase;
```

**Usage** : Après corruption ou upgrade majeur de PostgreSQL.

### 2.3. REINDEX CONCURRENTLY : Sans Downtime

**Nouveauté PostgreSQL 12+** : Reconstruction sans bloquer les écritures.

**Syntaxe** :
```sql
REINDEX INDEX CONCURRENTLY idx_produits_prix;
```

**Fonctionnement** :

```
1. Créer un nouvel index temporaire (en arrière-plan)
   → Écritures continuent sur l'ancien index

2. Synchroniser les nouvelles écritures vers le nouvel index

3. Valider le nouvel index (verrou bref)

4. Remplacer l'ancien index par le nouveau

5. Supprimer l'ancien index
```

**Avantages** :
- ✅ Aucun blocage des écritures (ou très bref, < 1 seconde)  
- ✅ Application reste disponible

**Inconvénients** :
- ❌ Plus lent (2-3× le temps d'un REINDEX normal)  
- ❌ Consomme plus d'espace disque temporairement (2× l'index)  
- ❌ Ne peut pas être exécuté dans une transaction

**Comparaison** :

| Critère | REINDEX | REINDEX CONCURRENTLY |
|---------|---------|----------------------|
| **Blocage écritures** | Oui (EXCLUSIVE LOCK) | Non (< 1s) |
| **Durée** | Rapide | 2-3× plus lent |
| **Espace disque** | 1× index | 2× index (temporaire) |
| **Transaction** | Oui | Non |
| **Échec** | Rollback auto | Index INVALID reste |

### 2.4. Exemple Complet

**Scénario** : Table `orders` avec 100 GB, index `idx_orders_date` de 10 GB, 35% de bloat.

#### Avec Downtime (Maintenance Programmée)

```sql
-- Fenêtre de maintenance : 02h00 - 04h00

-- Optionnel : Vérifier le bloat avant
SELECT * FROM pgstatindex('idx_orders_date');

-- REINDEX (rapide, mais bloque)
REINDEX INDEX idx_orders_date;
-- Durée : ~8 minutes

-- Vérifier après
SELECT * FROM pgstatindex('idx_orders_date');
-- avg_leaf_density : 95% (excellent !)
```

#### Sans Downtime (Production Continue)

```sql
-- Production 24/7, pas de fenêtre de maintenance

-- REINDEX CONCURRENTLY
REINDEX INDEX CONCURRENTLY idx_orders_date;
-- Durée : ~20 minutes (mais application reste disponible)

-- En cas d'échec (rare), l'index reste INVALID
-- Vérifier :
SELECT indexname, indexdef  
FROM pg_indexes  
WHERE indexname LIKE '%_ccnew';  

-- Nettoyer si nécessaire
DROP INDEX CONCURRENTLY idx_orders_date_ccnew;
```

### 2.5. Quand Utiliser REINDEX ?

✅ **Cas d'usage** :

1. **Bloat > 30-40%** : Performances dégradées  
2. **Après DELETE massif** : Beaucoup d'entrées mortes  
3. **Après corruption** : Erreur "index corrupted"  
4. **Optimisation périodique** : Maintenance préventive (ex: mensuelle)

❌ **Quand NE PAS utiliser** :

1. **Index récemment créé** : Aucun bloat  
2. **Bloat < 20%** : Coût/bénéfice défavorable  
3. **Système en surcharge** : REINDEX consomme CPU/I/O

---

## 3. VACUUM vs VACUUM FULL

### 3.1. VACUUM : Nettoyage Léger

**Fonctionnement** :
- Marque les espaces morts comme **réutilisables**
- Ne libère PAS l'espace disque au système d'exploitation
- Peut s'exécuter en parallèle avec les écritures (pas de verrou exclusif)

```sql
VACUUM produits;

-- Avec analyse des statistiques
VACUUM ANALYZE produits;
```

**Effet sur un index** :
```
Avant VACUUM :  
Index : 100 MB (60 MB actifs, 40 MB morts)  

Après VACUUM :  
Index : 100 MB (60 MB actifs, 40 MB réutilisables)  
        ↑ Taille inchangée sur disque !
```

**Avantage** : Rapide, non bloquant.

**Inconvénient** : Ne réduit pas la taille de l'index.

### 3.2. VACUUM FULL : Nettoyage Complet

**Fonctionnement** :
- Réécrit **complètement** la table ET les index
- Libère l'espace disque au système d'exploitation
- Pose un **verrou exclusif** (EXCLUSIVE LOCK)

```sql
VACUUM FULL produits;
```

**Effet** :
```
Avant VACUUM FULL :  
Table : 500 MB (300 MB actifs, 200 MB morts)  
Index : 100 MB (60 MB actifs, 40 MB morts)  

Après VACUUM FULL :  
Table : 300 MB (300 MB actifs, 0 MB morts)  ← Taille réduite !  
Index : 60 MB (60 MB actifs, 0 MB morts)    ← Taille réduite !  
```

**Avantages** :
- ✅ Récupération réelle d'espace disque  
- ✅ Table et index compacts (comme neufs)

**Inconvénients** :
- ❌ Verrou exclusif (bloque tout)  
- ❌ Très lent (réécrit tout)  
- ❌ Nécessite 2× l'espace disque temporairement  
- ❌ Ne peut pas être interrompu (CTRL+C ne fonctionne pas)

### 3.3. Comparaison

| Critère | VACUUM | VACUUM FULL | REINDEX |
|---------|--------|-------------|---------|
| **Cible** | Table + Index | Table + Index | Index seul |
| **Espace libéré** | Marqué réutilisable | Rendu au système | Rendu au système |
| **Verrou** | Partagé (non bloquant) | Exclusif (bloque tout) | Exclusif (ou CONCURRENTLY) |
| **Durée** | Rapide | Très lent | Moyen |
| **Utilisation** | Maintenance courante | Urgence (espace disque) | Bloat index important |

### 3.4. Quand Utiliser VACUUM FULL ?

✅ **Cas d'usage rares** :

1. **Espace disque critique** : Disque à 95% plein  
2. **Après DELETE massif** : 70-80% des lignes supprimées  
3. **Réduction de table** : Table historique purgée

**Exemple** :
```sql
-- Situation : Table de 500 GB avec 400 GB d'anciennes données supprimées
DELETE FROM logs WHERE created_at < NOW() - INTERVAL '1 year';
-- Résultat : Table toujours 500 GB (espace marqué réutilisable)

-- Récupérer l'espace disque
VACUUM FULL logs;
-- Résultat : Table réduite à 100 GB (400 GB libérés)
```

❌ **Ne PAS utiliser VACUUM FULL si** :

- VACUUM régulier suffit
- Pas de problème d'espace disque
- Pas de fenêtre de maintenance (production 24/7)

**Alternative recommandée** : `pg_repack` (voir section 3.5).

### 3.5. Alternative : pg_repack

**Extension** : Réorganise tables et index **sans verrou exclusif**.

**Installation** :
```bash
# Debian/Ubuntu
sudo apt install postgresql-contrib

# Activer dans PostgreSQL
CREATE EXTENSION pg_repack;
```

**Utilisation** :
```bash
# Réorganiser une table spécifique
pg_repack -d mabase -t produits

# Réorganiser toute la base
pg_repack -d mabase
```

**Avantages sur VACUUM FULL** :
- ✅ Pas de verrou exclusif (< 1 seconde)  
- ✅ Application reste disponible  
- ✅ Récupération réelle d'espace disque

**Inconvénient** :
- ❌ Nécessite 2× l'espace disque temporairement

---

## 4. Autovacuum et Maintenance Automatique

### 4.1. Qu'est-ce que l'Autovacuum ?

PostgreSQL exécute automatiquement `VACUUM` via le processus **autovacuum**.

**Configuration** (dans `postgresql.conf`) :
```ini
autovacuum = on                              # Activé par défaut  
autovacuum_max_workers = 3                   # 3 workers simultanés  
autovacuum_naptime = 1min                    # Vérification toutes les 1 minute  

# Seuils de déclenchement
autovacuum_vacuum_threshold = 50             # Nombre minimum de tuples modifiés  
autovacuum_vacuum_scale_factor = 0.2         # 20% de la table modifiée  

# Seuils pour ANALYZE
autovacuum_analyze_threshold = 50  
autovacuum_analyze_scale_factor = 0.1        # 10% de la table modifiée  
```

**Formule de déclenchement** :
```
Seuil VACUUM = autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × nb_tuples)

Exemple : Table de 100,000 lignes  
Seuil = 50 + (0.2 × 100,000) = 20,050 modifications  
```

Autovacuum se déclenche après 20,050 INSERT/UPDATE/DELETE.

### 4.2. Nouveauté PostgreSQL 18 : Autovacuum Amélioré

**Nouveauté 1 : autovacuum_vacuum_max_threshold**

```ini
# Nouveau paramètre PG 18
autovacuum_vacuum_max_threshold = 100000000   # 100 millions (défaut: infini)
```

**Problème résolu** : Sur très grandes tables (> 1 milliard de lignes), le seuil calculé peut être énorme.

**Exemple** :
```
Table : 10 milliards de lignes  
Seuil PG ≤ 17 : 50 + (0.2 × 10,000,000,000) = 2,000,000,050 (2 milliards !)  
Seuil PG 18  : min(2,000,000,050, 100,000,000) = 100,000,000 (100 millions)  
```

→ Autovacuum se déclenche plus fréquemment sur grandes tables.

**Nouveauté 2 : Ajustement Dynamique des Workers**

PostgreSQL 18 ajuste dynamiquement le nombre de workers autovacuum selon la charge :

```ini
autovacuum_worker_slots = 10   # Nouveauté PG 18 (au lieu de autovacuum_max_workers)
```

**Fonctionnement** :
- Charge faible → 2-3 workers actifs
- Charge élevée → Jusqu'à 10 workers
- Auto-régulation selon CPU et I/O disponibles

### 4.3. Surveiller l'Autovacuum

```sql
-- Dernières exécutions d'autovacuum
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    n_mod_since_analyze,
    autovacuum_count
FROM pg_stat_user_tables  
WHERE schemaname = 'public'  
ORDER BY n_mod_since_analyze DESC;  
```

**Indicateurs** :
- `last_autovacuum IS NULL` → Autovacuum jamais exécuté (problème !)  
- `n_mod_since_analyze` élevé → Table "en retard"

**Requête : Tables nécessitant VACUUM** :
```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables  
WHERE n_dead_tup > 1000  
  AND n_live_tup > 0
ORDER BY dead_pct DESC;
```

**Seuil d'alerte** :
- `dead_pct > 20%` → Autovacuum en retard, forcer VACUUM manuel  
- `dead_pct > 50%` → Critique, VACUUM immédiat

---

## 5. Stratégies de Maintenance

### 5.1. Maintenance Préventive (Recommended)

**Principe** : Maintenance régulière pour éviter les crises.

**Planning recommandé** :

| Fréquence | Action | Cible |
|-----------|--------|-------|
| **Quotidien** | Autovacuum | Automatique (toutes tables) |
| **Hebdomadaire** | ANALYZE | Tables critiques |
| **Mensuel** | REINDEX CONCURRENTLY | Index avec bloat > 30% |
| **Trimestriel** | Audit complet | Toute la base |
| **Annuel** | VACUUM FULL (si nécessaire) | Tables historiques purgées |

**Script de maintenance hebdomadaire** :
```sql
-- Créer une fonction de maintenance
CREATE OR REPLACE FUNCTION maintenance_hebdomadaire() RETURNS void AS $$  
BEGIN  
    -- 1. Analyser les tables critiques
    ANALYZE clients;
    ANALYZE commandes;
    ANALYZE produits;

    -- 2. Reindex les index bloatés (> 30%)
    -- (Vérifier le bloat avec pgstatindex avant)

    -- 3. Nettoyer les statistiques obsolètes
    VACUUM ANALYZE pg_statistic;

    RAISE NOTICE 'Maintenance hebdomadaire terminée';
END;
$$ LANGUAGE plpgsql;

-- Appel manuel
SELECT maintenance_hebdomadaire();
```

**Automatisation avec pg_cron** :
```sql
-- Installation
CREATE EXTENSION pg_cron;

-- Planifier : Tous les dimanches à 02h00
SELECT cron.schedule('maintenance-hebdo', '0 2 * * 0', 'SELECT maintenance_hebdomadaire()');
```

### 5.2. Maintenance Réactive (En Cas de Problème)

**Scénario** : Requêtes soudainement lentes.

**Diagnostic** :
```sql
-- 1. Vérifier le bloat des index
SELECT
    schemaname,
    tablename,
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan
FROM pg_stat_user_indexes  
WHERE schemaname = 'public'  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 10;  

-- 2. Analyser un index suspect avec pgstatindex
SELECT * FROM pgstatindex('idx_commandes_date');

-- 3. Si bloat > 40%, REINDEX
REINDEX INDEX CONCURRENTLY idx_commandes_date;

-- 4. Vérifier l'amélioration
SELECT * FROM pgstatindex('idx_commandes_date');
```

### 5.3. Maintenance sans Downtime

**Objectif** : Maintenance 24/7 sans arrêt de service.

**Checklist** :

1. **Utiliser REINDEX CONCURRENTLY**
```sql
REINDEX INDEX CONCURRENTLY idx_name;
```

2. **Éviter VACUUM FULL** → Utiliser pg_repack
```bash
pg_repack -d mabase -t table_name
```

3. **Monitorer l'impact** :
```sql
-- Pendant la maintenance, surveiller les locks
SELECT
    pid,
    usename,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity  
WHERE wait_event IS NOT NULL;  
```

4. **Planifier aux heures creuses** (même si non bloquant)

### 5.4. Maintenance d'Urgence (Espace Disque Critique)

**Scénario** : Disque à 95% plein, base de données menacée.

**Actions immédiates** :

**Étape 1** : Identifier les plus gros objets
```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
FROM pg_tables  
WHERE schemaname = 'public'  
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC  
LIMIT 5;  
```

**Étape 2** : Purger les données anciennes
```sql
-- Exemple : Logs de plus de 6 mois
DELETE FROM logs WHERE created_at < NOW() - INTERVAL '6 months';
```

**Étape 3** : VACUUM FULL (avec fenêtre de maintenance)
```sql
-- Attention : Bloque la table !
VACUUM FULL logs;
```

**Étape 4** : Vérifier l'espace récupéré
```bash
df -h
```

---

## 6. Monitoring et Alerting

### 6.1. Métriques Clés à Surveiller

#### Métrique 1 : Bloat des Index

**Requête** :
```sql
-- Avec extension pgstattuple
SELECT
    schemaname,
    tablename,
    indexrelname,
    round(100 * (1 - avg_leaf_density / 100), 1) AS bloat_pct,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes  
CROSS JOIN LATERAL pgstatindex(indexrelid)  
WHERE schemaname = 'public'  
  AND (1 - avg_leaf_density / 100) > 0.3  -- Bloat > 30%
ORDER BY bloat_pct DESC;
```

**Alerte** : Si bloat > 40% sur index fréquemment utilisé.

#### Métrique 2 : Tables Non Vacuumées

**Requête** :
```sql
SELECT
    schemaname,
    relname,
    last_autovacuum,
    NOW() - last_autovacuum AS time_since_vacuum,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables  
WHERE last_autovacuum IS NULL  
   OR NOW() - last_autovacuum > INTERVAL '7 days'
ORDER BY n_dead_tup DESC;
```

**Alerte** : Si table non vacuumée depuis > 7 jours et `n_dead_tup` élevé.

#### Métrique 3 : Taille des Index

**Requête** :
```sql
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS current_size,
    idx_scan AS scans,
    idx_tup_read AS reads
FROM pg_stat_user_indexes  
WHERE schemaname = 'public'  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 10;  
```

**Alerte** : Si taille augmente anormalement (croissance > 20% par mois).

### 6.2. Dashboard de Monitoring

**Outils recommandés** :

#### Prometheus + Grafana

**Métriques exportées** (via postgres_exporter) :
- `pg_stat_user_indexes_idx_blks_read`  
- `pg_stat_user_indexes_idx_blks_hit`  
- `pg_stat_user_tables_n_dead_tup`  
- `pg_database_size_bytes`

**Dashboard exemple** : https://grafana.com/grafana/dashboards/9628

#### pganalyze

Service SaaS spécialisé PostgreSQL :
- Détection automatique de bloat
- Recommandations de REINDEX
- Alertes proactives

#### Script Maison

```python
import psycopg2

conn = psycopg2.connect("dbname=mabase")  
cur = conn.cursor()  

# Vérifier le bloat
cur.execute("""
    SELECT indexrelname,
           round(100 * (1 - avg_leaf_density / 100), 1) AS bloat_pct
    FROM pg_stat_user_indexes
    CROSS JOIN LATERAL pgstatindex(indexrelid)
    WHERE schemaname = 'public'
""")

for index, bloat in cur.fetchall():
    if bloat > 40:
        print(f"⚠️ ALERTE : Index {index} a {bloat}% de bloat !")
        # Envoyer notification (email, Slack, PagerDuty, etc.)
```

---

## 7. Bonnes Pratiques

### 7.1. Checklist de Maintenance

- [ ] Autovacuum activé et configuré correctement  
- [ ] Monitoring du bloat (hebdomadaire)  
- [ ] REINDEX planifié (mensuel) sur index critiques  
- [ ] Alerting configuré (bloat > 40%, dead_tup élevé)  
- [ ] Documentation des fenêtres de maintenance  
- [ ] Tests de REINDEX CONCURRENTLY avant production  
- [ ] Backup avant VACUUM FULL

### 7.2. Optimisations de Configuration

**Pour Tables avec Beaucoup d'Écritures** :

```sql
-- Réduire le seuil autovacuum (VACUUM plus fréquent)
ALTER TABLE commandes SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5% au lieu de 20%
    autovacuum_vacuum_threshold = 100
);
```

**Pour Grandes Tables (> 1 GB)** :

```sql
-- Augmenter les coûts pour autovacuum (moins agressif)
ALTER TABLE logs SET (
    autovacuum_vacuum_cost_delay = 20,      -- Délai entre pages
    autovacuum_vacuum_cost_limit = 200      -- Limite de coût
);
```

### 7.3. Éviter les Pièges

❌ **Piège 1** : REINDEX sans vérifier l'espace disque

**Problème** : REINDEX nécessite 2× la taille de l'index temporairement.

**Solution** : Vérifier avant
```bash
df -h
# Assurer au moins 2× la taille du plus gros index disponible
```

❌ **Piège 2** : VACUUM FULL en production

**Problème** : Verrou exclusif bloque tout.

**Solution** : Utiliser pg_repack ou planifier fenêtre de maintenance.

❌ **Piège 3** : Oublier ANALYZE après REINDEX

**Problème** : Statistiques obsolètes.

**Solution** : Toujours faire
```sql
REINDEX INDEX idx_name;  
ANALYZE table_name;  
```

❌ **Piège 4** : REINDEX pendant heures de pointe

**Problème** : Même CONCURRENTLY consomme I/O et CPU.

**Solution** : Planifier aux heures creuses (nuit, week-end).

---

## 8. Cas Pratiques

### 8.1. Cas 1 : Application E-Commerce

**Contexte** :
- Table `orders` : 50 millions de lignes
- Index `idx_orders_date` : 5 GB
- Bloat détecté : 45%
- Contrainte : Production 24/7

**Solution** :
```sql
-- Dimanche 03h00 (heure creuse)
REINDEX INDEX CONCURRENTLY idx_orders_date;
-- Durée : 30 minutes
-- Impact : Aucun blocage

-- Vérifier
SELECT * FROM pgstatindex('idx_orders_date');
-- Bloat : 2% (excellent !)
```

### 8.2. Cas 2 : Purge de Logs Anciens

**Contexte** :
- Table `logs` : 500 GB
- Purge de 400 GB de données anciennes
- Disque à 85% plein

**Solution** :
```sql
-- Étape 1 : Purge
DELETE FROM logs WHERE created_at < '2024-01-01';
-- Résultat : Table toujours 500 GB (espace marqué réutilisable)

-- Étape 2 : VACUUM (rapide, mais n'aide pas l'espace disque)
VACUUM logs;
-- Résultat : Toujours 500 GB

-- Étape 3 : Fenêtre de maintenance (samedi 02h00 - 06h00)
VACUUM FULL logs;
-- Durée : 3 heures
-- Résultat : Table réduite à 100 GB (400 GB libérés !)

-- Étape 4 : Vérification
SELECT pg_size_pretty(pg_total_relation_size('logs'));
-- Résultat : 100 GB
```

### 8.3. Cas 3 : Migration sans Downtime

**Contexte** :
- Migration PostgreSQL 15 → 18
- 20 tables avec bloat > 30%
- Contrainte : Aucune interruption

**Solution** :
```bash
# Avant migration : Nettoyer
for table in clients commandes produits ...; do
    pg_repack -d mabase -t $table
done

# Migration avec pg_upgrade
pg_upgrade --link ...

# Après migration : ANALYZE toutes les tables
psql -d mabase -c "ANALYZE"

# Vérifier les performances
psql -d mabase -c "
    SELECT tablename, last_analyze
    FROM pg_stat_user_tables
    WHERE schemaname = 'public'
"
```

---

## Points Clés à Retenir

🔑 **Bloat = espace inutilisé** dans les index (causé par DELETE, UPDATE).

🔑 **REINDEX** reconstruit l'index depuis zéro, éliminant le bloat.

🔑 **REINDEX CONCURRENTLY** : Sans downtime (PG 12+), mais 2-3× plus lent.

🔑 **VACUUM** : Marque l'espace réutilisable, ne libère pas au système.

🔑 **VACUUM FULL** : Libère réellement l'espace, mais bloque tout (dernier recours).

🔑 **pg_repack** : Alternative à VACUUM FULL sans verrou exclusif.

🔑 **Autovacuum** : Maintenance automatique (vérifier qu'il fonctionne !).

🔑 **PostgreSQL 18** : Autovacuum amélioré (max_threshold, worker_slots dynamiques).

🔑 **Monitoring crucial** : Surveiller bloat > 30%, dead_tup, derniers vacuum.

🔑 **Maintenance préventive** : Mensuelle (REINDEX) >> réactive (VACUUM FULL).

🔑 **pgstattuple** : Extension indispensable pour mesurer le bloat.

🔑 **Heures creuses** : Planifier REINDEX même CONCURRENTLY (consomme I/O).

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL** : [REINDEX](https://www.postgresql.org/docs/current/sql-reindex.html), [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)  
- **Extension pgstattuple** : [Documentation](https://www.postgresql.org/docs/current/pgstattuple.html)  
- **pg_repack** : [GitHub](https://github.com/reorg/pg_repack)  
- **Section précédente** : 13.10. Prepared Statements et performance  
- **Section suivante** : Chapitre 14. Observabilité et Monitoring

---


⏭️ [Observabilité et Monitoring](/14-observabilite-et-monitoring/README.md)
