🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6. Le Planificateur de Requêtes et les Statistiques (pg_stats)

## Introduction

Le **planificateur de requêtes** (Query Planner ou Query Optimizer) est le "cerveau" de PostgreSQL. Quand vous exécutez une requête SQL, le planificateur analyse toutes les façons possibles de l'exécuter et choisit le **plan d'exécution** le plus efficace.

Pour prendre ces décisions, le planificateur s'appuie sur des **statistiques** détaillées sur vos données, stockées dans la vue système `pg_stats`.

**Analogie** : Imaginez que vous planifiez un trajet en voiture. Vous avez plusieurs routes possibles. Le GPS (planificateur) utilise des statistiques (trafic, limitations de vitesse, distance) pour choisir le meilleur itinéraire. Sans statistiques à jour, le GPS pourrait vous envoyer sur une route embouteillée !

**Prérequis** : Avoir lu les sections 13.1 à 13.3 sur les stratégies de scan et les index.

---

## 1. Qu'est-ce que le Planificateur de Requêtes ?

### 1.1. Rôle du Planificateur

Le planificateur transforme votre requête SQL (déclarative) en un **plan d'exécution** (procédural).

```
Requête SQL                    Plan d'exécution
(Que veux-tu ?)               (Comment l'obtenir ?)
    ↓                              ↓
SELECT * FROM clients     →    Index Scan on idx_client  
WHERE id = 12345               Index Cond: (id = 12345)  
                               Estimated cost: 8.44
```

### 1.2. Le Processus de Planification

```
1. Analyse syntaxique (Parser)
   ↓
2. Réécriture (Rewriter) : Application des vues, règles
   ↓
3. Planification (Planner) : Génération des plans possibles
   ↓
4. Optimisation : Sélection du meilleur plan
   ↓
5. Exécution (Executor)
```

**Phase critique** : L'étape 3-4 où le planificateur doit choisir parmi **potentiellement des millions** de plans possibles.

### 1.3. Exemple Simple

```sql
SELECT c.nom, o.montant  
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris';  
```

**Plans possibles** (simplifiés) :

**Plan A** : Seq Scan clients → Filter Paris → Join avec commandes  
**Plan B** : Index Scan sur idx_ville → Join avec commandes  
**Plan C** : Seq Scan commandes → Hash Join avec clients filtrés  

**Question** : Lequel choisir ?

**Réponse** : Ça dépend des statistiques !
- Combien de clients à Paris ? (10 ou 100 000 ?)
- Taille de la table commandes ?
- Index disponibles ?
- Distribution des données ?

→ Le planificateur utilise les **statistiques** pour calculer le coût de chaque plan et choisir le meilleur.

---

## 2. Le Modèle de Coût PostgreSQL

### 2.1. Concept de Coût

PostgreSQL attribue un **coût** (cost) à chaque opération. Le coût est une unité **abstraite** qui combine :
- Lectures disque (I/O)
- Calculs CPU
- Mémoire utilisée

**Important** : Le coût n'est PAS un temps en secondes, c'est une valeur relative pour comparer les plans.

### 2.2. Unités de Coût

PostgreSQL définit des coûts de base :

```sql
-- Coût de lecture d'une page séquentielle (default: 1.0)
SHOW seq_page_cost;

-- Coût de lecture d'une page aléatoire (default: 4.0)
SHOW random_page_cost;

-- Coût du traitement CPU par ligne (default: 0.01)
SHOW cpu_tuple_cost;

-- Coût d'un opérateur CPU (default: 0.0025)
SHOW cpu_operator_cost;
```

**Interprétation** :
- `random_page_cost = 4.0` : Une lecture aléatoire coûte 4× plus qu'une lecture séquentielle
- Disques SSD : Souvent configuré à `random_page_cost = 1.1-1.5` (lectures aléatoires quasi aussi rapides)

### 2.3. Calcul du Coût d'un Seq Scan

```
Coût = (Nombre de pages × seq_page_cost) + (Nombre de lignes × cpu_tuple_cost)
```

**Exemple** :
- Table de 1000 pages (8 MB)
- 100 000 lignes

```
Coût = (1000 × 1.0) + (100000 × 0.01)
     = 1000 + 1000
     = 2000
```

### 2.4. Calcul du Coût d'un Index Scan

```
Coût = (Pages index × random_page_cost)
     + (Pages table × random_page_cost)
     + (Lignes × cpu_tuple_cost)
```

**Exemple** : Recherche de 100 lignes via index
- 10 pages d'index
- 100 pages de table (1 ligne par page en moyenne)

```
Coût = (10 × 4.0) + (100 × 4.0) + (100 × 0.01)
     = 40 + 400 + 1
     = 441
```

**Comparaison** :
- Seq Scan : 2000
- Index Scan : 441

→ PostgreSQL choisit l'Index Scan ! ✅

---

## 3. Les Statistiques : La Base des Décisions

### 3.1. Pourquoi les Statistiques sont Cruciales

Sans statistiques précises, le planificateur est **aveugle** :

```sql
-- Sans statistiques
SELECT * FROM clients WHERE ville = 'Paris';

-- PostgreSQL devine : "Peut-être 10% des lignes ?"
-- En réalité : 90% des clients sont à Paris !

→ Mauvais choix de plan → Requête lente
```

**Principe** : Garbage In, Garbage Out
- Statistiques obsolètes → Mauvais plans → Mauvaises performances

### 3.2. Types de Statistiques Collectées

PostgreSQL collecte plusieurs types de statistiques :

1. **Cardinalité** : Nombre total de lignes  
2. **Valeurs distinctes** : Nombre de valeurs uniques (n_distinct)  
3. **NULL frequency** : Proportion de NULL (null_frac)  
4. **Histogrammes** : Distribution des valeurs  
5. **Most Common Values (MCV)** : Valeurs les plus fréquentes  
6. **Corrélation** : Ordre physique vs ordre logique

### 3.3. La Vue pg_stats

`pg_stats` est une vue qui expose les statistiques de chaque colonne :

```sql
SELECT
    schemaname,
    tablename,
    attname,           -- Nom de la colonne
    n_distinct,        -- Nombre de valeurs distinctes
    null_frac,         -- Fraction de NULL
    avg_width,         -- Taille moyenne en octets
    most_common_vals,  -- Valeurs les plus communes
    most_common_freqs  -- Fréquences correspondantes
FROM pg_stats  
WHERE tablename = 'clients';  
```

---

## 4. Comprendre pg_stats en Détail

### 4.1. Colonnes Essentielles

#### n_distinct : Cardinalité

Nombre de valeurs distinctes dans la colonne.

```sql
SELECT attname, n_distinct  
FROM pg_stats  
WHERE tablename = 'clients';  
```

**Exemple** :
```
 attname  | n_distinct
----------|------------
 id       | -1.0        ← -1 = 100% des lignes sont uniques (PRIMARY KEY)
 ville    | 450         ← 450 villes distinctes
 age      | 80          ← 80 âges différents
 sexe     | 2           ← 2 valeurs (M/F)
```

**Interprétation** :
- `n_distinct = -1` : Toutes les lignes sont uniques (ratio 1:1)  
- `n_distinct > 0` : Nombre absolu de valeurs distinctes  
- `n_distinct < 0` : Fraction (ex: -0.5 = 50% des lignes sont uniques)

**Usage par le planificateur** :
- Faible n_distinct (2-10) → Index moins utile, sauf index partiel
- Élevé n_distinct → Index très sélectif → Index Scan préféré

#### null_frac : Proportion de NULL

```sql
SELECT attname, null_frac  
FROM pg_stats  
WHERE tablename = 'clients';  
```

**Exemple** :
```
 attname     | null_frac
-------------|----------
 id          | 0.0       ← Aucun NULL (NOT NULL)
 telephone   | 0.15      ← 15% de NULL
 email       | 0.02      ← 2% de NULL
```

**Usage par le planificateur** :
```sql
SELECT * FROM clients WHERE telephone IS NULL;
-- PostgreSQL sait : 15% des lignes → Estime 15,000 lignes sur 100,000
```

#### avg_width : Taille Moyenne

Taille moyenne des valeurs en octets.

```sql
SELECT attname, avg_width  
FROM pg_stats  
WHERE tablename = 'clients';  
```

**Exemple** :
```
 attname  | avg_width
----------|----------
 id       | 4         ← INTEGER (4 octets)
 nom      | 12        ← VARCHAR, moyenne 12 caractères
 email    | 28        ← Emails plus longs
```

**Usage par le planificateur** :
- Estime la quantité de mémoire nécessaire pour les hash tables, sorts, etc.

### 4.2. Histogrammes et Distribution

#### most_common_vals et most_common_freqs

Liste des valeurs les plus fréquentes et leurs proportions.

```sql
SELECT
    attname,
    most_common_vals,
    most_common_freqs
FROM pg_stats  
WHERE tablename = 'clients' AND attname = 'ville';  
```

**Résultat** :
```
attname | most_common_vals                  | most_common_freqs
--------|-----------------------------------|------------------
ville   | {Paris,Lyon,Marseille,Toulouse}   | {0.35,0.15,0.12,0.08}
```

**Interprétation** :
- 35% des clients sont à Paris
- 15% à Lyon
- 12% à Marseille
- 8% à Toulouse
- 30% répartis dans les autres villes

**Usage par le planificateur** :

```sql
SELECT * FROM clients WHERE ville = 'Paris';
-- PostgreSQL calcule : 35% × 100,000 = 35,000 lignes attendues
-- → Trop pour un Index Scan → Préfère Seq Scan ou Bitmap Scan
```

```sql
SELECT * FROM clients WHERE ville = 'Dijon';
-- Dijon n'est pas dans les MCV → Estimation par défaut (ex: 0.1%)
-- → 0.1% × 100,000 = 100 lignes
-- → Index Scan optimal
```

#### histogram_bounds : Distribution Complète

Pour les valeurs non couvertes par MCV, PostgreSQL construit un histogramme.

```sql
SELECT attname, histogram_bounds  
FROM pg_stats  
WHERE tablename = 'commandes' AND attname = 'montant';  
```

**Résultat** :
```
attname  | histogram_bounds
---------|--------------------------------------------------
montant  | {10,25,50,75,100,150,200,300,500,1000,5000}
```

**Interprétation** : Les buckets (seaux) d'histogramme :
- Bucket 1 : [10, 25[ → ~10% des lignes
- Bucket 2 : [25, 50[ → ~10% des lignes
- ...
- Bucket 10 : [1000, 5000] → ~10% des lignes

**Usage par le planificateur** :

```sql
SELECT * FROM commandes WHERE montant BETWEEN 100 AND 200;
-- PostgreSQL utilise l'histogramme :
-- Bucket [100,150] + Bucket [150,200] ≈ 20% des lignes
-- → Estime 20,000 lignes sur 100,000
```

### 4.3. Corrélation Physique

```sql
SELECT attname, correlation  
FROM pg_stats  
WHERE tablename = 'commandes';  
```

**Résultat** :
```
 attname        | correlation
----------------|------------
 id             | 1.0        ← Parfaitement corrélé
 created_at     | 0.98       ← Quasi-corrélé
 client_id      | 0.05       ← Aucune corrélation
```

**Interprétation** :

**Corrélation = 1.0** : Les valeurs sont physiquement triées sur le disque dans le même ordre que les valeurs logiques.

```
Ordre logique : id = 1, 2, 3, 4, 5, 6...  
Ordre physique : [Bloc 1: id=1,2,3] [Bloc 2: id=4,5,6]...  
→ Lecture séquentielle efficace
```

**Corrélation = 0.0** : Aucun ordre, valeurs dispersées aléatoirement.

```
Ordre logique : client_id = 1, 2, 3, 4, 5...  
Ordre physique : [Bloc 1: id=783,45,2] [Bloc 2: id=1,456,99]...  
→ Lectures aléatoires (Random I/O)
```

**Usage par le planificateur** :

```sql
SELECT * FROM commandes ORDER BY created_at LIMIT 100;
```

- Si `correlation(created_at) = 0.98` → Les premières valeurs sont dans les premiers blocs → Index Scan très rapide
- Si `correlation(created_at) = 0.05` → Valeurs éparpillées → Index Scan avec beaucoup de Random I/O

---

## 5. ANALYZE : Mise à Jour des Statistiques

### 5.1. Qu'est-ce que ANALYZE ?

`ANALYZE` est la commande qui collecte les statistiques sur les tables.

```sql
-- Analyser une table spécifique
ANALYZE clients;

-- Analyser toutes les tables de la base
ANALYZE;

-- Analyser et obtenir des détails
ANALYZE VERBOSE clients;
```

### 5.2. Fonctionnement d'ANALYZE

1. **Échantillonnage** : ANALYZE ne lit PAS toute la table, il échantillonne un sous-ensemble de lignes

```
Table : 10 millions de lignes  
ANALYZE lit : ~30,000 lignes (échantillon)  
→ Très rapide, même sur grandes tables
```

2. **Calcul des statistiques** : À partir de l'échantillon, PostgreSQL calcule :
   - n_distinct (estimation)
   - Histogrammes
   - MCV (Most Common Values)
   - null_frac
   - etc.

3. **Stockage** : Les statistiques sont stockées dans `pg_statistic` (vue `pg_stats`)

### 5.3. Taille de l'Échantillon

Contrôlé par le paramètre `default_statistics_target` (default: 100).

```sql
-- Afficher la valeur actuelle
SHOW default_statistics_target;

-- Modifier globalement
ALTER DATABASE mabase SET default_statistics_target = 200;

-- Modifier pour une colonne spécifique
ALTER TABLE clients ALTER COLUMN ville SET STATISTICS 500;
```

**Valeurs** :
- `100` (default) : Bon compromis (échantillon de ~30,000 lignes)  
- `1000` : Maximum (échantillon de ~300,000 lignes) → Plus précis, mais ANALYZE plus lent  
- `10` : Minimum → ANALYZE très rapide, mais statistiques imprécises

**Quand augmenter** :
- Colonnes avec distributions très irrégulières
- Tables critiques avec requêtes complexes
- Après augmentation : `ANALYZE clients;`

### 5.4. Quand Exécuter ANALYZE ?

#### Automatique : Autovacuum

PostgreSQL exécute automatiquement `ANALYZE` via le processus **autovacuum**.

```sql
-- Vérifier le dernier ANALYZE
SELECT
    schemaname,
    relname,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze  -- Nombre de modifications depuis dernier ANALYZE
FROM pg_stat_user_tables  
WHERE relname = 'clients';  
```

**Résultat** :
```
 schemaname | relname  | last_analyze        | last_autoanalyze    | n_mod_since_analyze
------------|----------|---------------------|---------------------|--------------------
 public     | clients  | 2025-11-20 10:15:00 | 2025-11-21 08:30:00 | 1234
```

**Autovacuum déclenche ANALYZE quand** :
```
n_mod_since_analyze > autovacuum_analyze_threshold + (autovacuum_analyze_scale_factor × nombre_lignes)
```

**Valeurs par défaut** :
- `autovacuum_analyze_threshold = 50`  
- `autovacuum_analyze_scale_factor = 0.1` (10%)

**Exemple** : Table de 100,000 lignes
```
Seuil = 50 + (0.1 × 100,000) = 10,050 modifications
```

Après 10,050 INSERT/UPDATE/DELETE, autovacuum déclenche ANALYZE.

#### Manuel : Quand Forcer ANALYZE

Exécutez manuellement `ANALYZE` dans ces cas :

1. **Après un chargement massif de données** :
```sql
COPY clients FROM '/data/clients.csv';  
ANALYZE clients;  
```

2. **Après une modification importante du schéma** :
```sql
ALTER TABLE clients ADD COLUMN new_field VARCHAR(100);  
UPDATE clients SET new_field = ...;  
ANALYZE clients;  
```

3. **Avant une requête critique** (rare) :
```sql
ANALYZE clients;
-- Requête complexe importante
SELECT ...
```

4. **Après détection de mauvais plans** :
```sql
-- Si EXPLAIN montre des estimations très fausses
ANALYZE clients;  
ANALYZE commandes;  
```

---

## 6. Lecture d'un Plan avec les Statistiques

### 6.1. EXPLAIN : Voir les Estimations

```sql
EXPLAIN SELECT * FROM clients WHERE ville = 'Paris';
```

**Résultat** :
```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
  Filter: (ville = 'Paris'::text)
```

**Interprétation** :
- `cost=0.00..2123.00` : Coût estimé (startup..total)  
- `rows=35000` : **Estimation basée sur pg_stats** (35% × 100,000)  
- `width=50` : Taille moyenne des lignes en octets

### 6.2. EXPLAIN ANALYZE : Comparer Estimations vs Réalité

```sql
EXPLAIN ANALYZE SELECT * FROM clients WHERE ville = 'Paris';
```

**Résultat** :
```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
                     (actual time=0.045..23.456 rows=34987 loops=1)
  Filter: (ville = 'Paris'::text)
  Rows Removed by Filter: 65013
Planning Time: 0.234 ms  
Execution Time: 25.891 ms  
```

**Comparaison** :
- **Estimé** : `rows=35000`  
- **Réel** : `actual ... rows=34987`  
- **Écart** : 0.04% → Excellente estimation ! ✅

### 6.3. Détecter les Mauvaises Estimations

```sql
EXPLAIN ANALYZE SELECT * FROM clients WHERE ville = 'Dijon';
```

**Résultat** :
```
Index Scan using idx_ville on clients  (cost=0.42..8.44 rows=1 width=50)
                                       (actual time=0.123..456.789 rows=25000 loops=1)
  Index Cond: (ville = 'Dijon'::text)
```

**Problème** :
- **Estimé** : `rows=1` (0.001%)  
- **Réel** : `actual ... rows=25000` (25%)  
- **Écart** : 2,500,000% ! 🔴

**Cause** : Statistiques obsolètes (Dijon était rare, maintenant fréquent)

**Solution** :
```sql
ANALYZE clients;
```

---

## 7. Paramètres du Planificateur

### 7.1. Paramètres de Coût

#### seq_page_cost et random_page_cost

```sql
-- Afficher les valeurs
SHOW seq_page_cost;      -- Default: 1.0  
SHOW random_page_cost;   -- Default: 4.0  
```

**Ajustement pour SSD** :
```sql
-- Dans postgresql.conf ou par session
SET random_page_cost = 1.1;  -- SSD : accès aléatoire presque aussi rapide
```

**Impact** : Rend les Index Scans plus attractifs.

#### effective_cache_size

Mémoire que PostgreSQL suppose disponible pour le cache OS.

```sql
SHOW effective_cache_size;  -- Default: 4GB
```

**Recommandation** : 50-75% de la RAM totale

```sql
-- Serveur avec 64 GB RAM
ALTER SYSTEM SET effective_cache_size = '48GB';  
SELECT pg_reload_conf();  
```

**Impact** : Influence le choix entre Index Scan et Seq Scan (planificateur suppose que les données fréquentes sont en cache).

### 7.2. Désactiver Temporairement des Stratégies

Pour déboguer ou forcer un plan spécifique :

```sql
-- Désactiver Seq Scan (force index si possible)
SET enable_seqscan = off;

-- Désactiver Index Scan
SET enable_indexscan = off;

-- Désactiver Bitmap Scan
SET enable_bitmapscan = off;

-- Désactiver Nested Loop Join
SET enable_nestloop = off;
```

**⚠️ Attention** : À utiliser uniquement pour **tests/debug**. Ne jamais désactiver en production !

### 7.3. Forcer la Replanification

Par défaut, PostgreSQL met en cache les plans (Prepared Statements).

```sql
-- Invalider le cache de plans
DISCARD PLANS;

-- Ou re-préparer après ANALYZE
DEALLOCATE ALL;
```

---

## 8. Problèmes Courants et Solutions

### 8.1. Problème : Statistiques Obsolètes

**Symptôme** :
- Requêtes soudainement lentes
- EXPLAIN montre des estimations très fausses

**Diagnostic** :
```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_mod_since_analyze,
    last_autoanalyze
FROM pg_stat_user_tables  
WHERE n_mod_since_analyze > n_live_tup * 0.2  -- Plus de 20% de modifications  
ORDER BY n_mod_since_analyze DESC;  
```

**Solution** :
```sql
ANALYZE;  -- Sur toute la base
-- Ou spécifique
ANALYZE table_problematique;
```

### 8.2. Problème : Distribution Très Irrégulière

**Symptôme** :
- Certaines valeurs ont des millions de lignes, d'autres 1-10
- Plans suboptimaux pour les valeurs rares

**Exemple** :
```sql
-- Table logs avec colonne 'level'
-- 'INFO': 99.9% des lignes
-- 'ERROR': 0.1% des lignes

SELECT * FROM logs WHERE level = 'ERROR';  -- Devrait être rapide (100 lignes)
-- Mais PostgreSQL estime mal car INFO domine les statistiques
```

**Solution 1** : Augmenter statistics_target
```sql
ALTER TABLE logs ALTER COLUMN level SET STATISTICS 1000;  
ANALYZE logs;  
```

**Solution 2** : Index partiel
```sql
CREATE INDEX idx_logs_errors ON logs(timestamp) WHERE level = 'ERROR';
```

### 8.3. Problème : Fonctions Utilisateur Opaques

**Symptôme** :
PostgreSQL ne peut pas estimer la sélectivité des fonctions personnalisées.

```sql
SELECT * FROM clients WHERE ma_fonction_custom(nom) = true;
-- PostgreSQL devine : 0.5% par défaut (souvent faux)
```

**Solution** : Déclarer la volatilité et la sélectivité de la fonction
```sql
CREATE OR REPLACE FUNCTION ma_fonction_custom(nom TEXT) RETURNS BOOLEAN AS $$
    ...
$$ LANGUAGE plpgsql
IMMUTABLE  -- ou STABLE  
COST 100   -- Coût CPU estimé  
ROWS 50;   -- Nombre de lignes attendues en sortie (pour fonctions set-returning)  
```

### 8.4. Problème : Jointures avec Mauvaises Estimations

**Symptôme** :
```sql
EXPLAIN ANALYZE  
SELECT * FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris';  

-- Estimation : 100 lignes
-- Réel : 10,000 lignes
```

**Solution** : ANALYZE les deux tables
```sql
ANALYZE clients;  
ANALYZE commandes;  
```

Si problème persiste, vérifier les statistiques de corrélation :
```sql
SELECT
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats  
WHERE tablename IN ('clients', 'commandes');  
```

---

## 9. Monitoring et Maintenance des Statistiques

### 9.1. Surveiller la Fraîcheur des Statistiques

**Requête de monitoring** :
```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS live_rows,
    n_mod_since_analyze AS modifications,
    last_autoanalyze,
    CASE
        WHEN n_mod_since_analyze > n_live_tup * 0.2 THEN '⚠️ URGENT'
        WHEN n_mod_since_analyze > n_live_tup * 0.1 THEN '⚠️ Soon'
        ELSE '✓ OK'
    END AS status
FROM pg_stat_user_tables  
WHERE schemaname = 'public'  
ORDER BY n_mod_since_analyze DESC  
LIMIT 20;  
```

### 9.2. Script de Maintenance Automatique

```sql
-- Créer une fonction pour forcer ANALYZE sur les tables "stale"
CREATE OR REPLACE FUNCTION maintain_statistics() RETURNS void AS $$  
BEGIN  
    EXECUTE (
        SELECT string_agg('ANALYZE ' || schemaname || '.' || relname || ';', E'\n')
        FROM pg_stat_user_tables
        WHERE n_mod_since_analyze > n_live_tup * 0.1
    );
    RAISE NOTICE 'Statistics updated';
END;
$$ LANGUAGE plpgsql;

-- Appel périodique (via cron ou pg_cron)
SELECT maintain_statistics();
```

### 9.3. Nouveauté PostgreSQL 18 : Statistiques VACUUM/ANALYZE

PostgreSQL 18 enrichit `pg_stat_all_tables` avec des statistiques détaillées :

```sql
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,        -- Nouveauté PG 18
    autovacuum_count,    -- Nouveauté PG 18
    analyze_count,       -- Nouveauté PG 18
    autoanalyze_count    -- Nouveauté PG 18
FROM pg_stat_all_tables  
WHERE schemaname = 'public';  
```

**Utilité** : Suivre la fréquence de maintenance et détecter les tables sous-maintenues.

---

## 10. Outils d'Analyse des Statistiques

### 10.1. Extension pg_qualstats

**Installation** :
```sql
CREATE EXTENSION pg_qualstats;
```

**Fonction** : Collecte des statistiques sur les prédicats (WHERE clauses) les plus fréquents.

**Utilité** : Identifier quelles colonnes bénéficieraient le plus d'index.

### 10.2. Extension HypoPG (Hypothetical Indexes)

```sql
CREATE EXTENSION hypopg;

-- Créer un index hypothétique (pas réellement créé)
SELECT hypopg_create_index('CREATE INDEX ON clients(ville)');

-- Tester le plan
EXPLAIN SELECT * FROM clients WHERE ville = 'Paris';

-- Voir que PostgreSQL considère l'index hypothétique !

-- Supprimer l'index hypothétique
SELECT hypopg_drop_index(indexrelid) FROM hypopg_list_indexes;
```

**Utilité** : Tester l'impact d'un index **avant de le créer** (pas de coût de création).

### 10.3. pgAdmin : Visualisation des Statistiques

pgAdmin propose une interface graphique pour visualiser :
- Histogrammes de pg_stats
- Distribution des valeurs
- MCV (Most Common Values)

**Accès** : pgAdmin → Table → Statistics

---

## 11. Exemple Complet : Diagnostic et Optimisation

### 11.1. Scénario : Requête Lente

```sql
-- Requête problématique (7 secondes)
SELECT c.nom, COUNT(o.id)  
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris'  
  AND o.created_at > '2024-01-01'
GROUP BY c.nom;
```

### 11.2. Étape 1 : EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT c.nom, COUNT(o.id)  
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris'  
  AND o.created_at > '2024-01-01'
GROUP BY c.nom;
```

**Résultat** :
```
HashAggregate  (cost=250000.00..250500.00 rows=50000 width=20)
               (actual time=7234.567..7235.123 rows=5 loops=1)
  Group Key: c.nom
  ->  Hash Join  (cost=15000.00..240000.00 rows=500000 width=12)
                 (actual time=123.456..7000.234 rows=5000 loops=1)
        Hash Cond: (o.client_id = c.id)
        ->  Seq Scan on commandes o  (cost=0.00..180000.00 rows=500000 width=8)
                                     (actual time=0.012..3456.789 rows=500000 loops=1)
              Filter: (created_at > '2024-01-01')
              Rows Removed by Filter: 500000
        ->  Hash  (cost=12000.00..12000.00 rows=50000 width=12)
                  (actual time=120.234..120.234 rows=35000 loops=1)
              Buckets: 65536  Batches: 1  Memory Usage: 2048kB
              ->  Seq Scan on clients c  (cost=0.00..12000.00 rows=50000 width=12)
                                        (actual time=0.023..98.765 rows=35000 loops=1)
                    Filter: (ville = 'Paris')
                    Rows Removed by Filter: 65000
Planning Time: 1.234 ms  
Execution Time: 7235.456 ms  
```

### 11.3. Étape 2 : Analyser les Estimations

**Problèmes détectés** :

1. **Seq Scan sur commandes** :
   - Estimé : 500,000 lignes
   - Réel : 500,000 lignes
   - ✅ Bonne estimation, mais pas d'index sur `created_at`

2. **Seq Scan sur clients** :
   - Estimé : 50,000 lignes pour `ville='Paris'`
   - Réel : 35,000 lignes
   - ✅ Estimation correcte

3. **Pas d'index utilisé** : Aucun index disponible

### 11.4. Étape 3 : Vérifier les Statistiques

```sql
-- Statistiques sur commandes.created_at
SELECT
    attname,
    n_distinct,
    null_frac,
    correlation
FROM pg_stats  
WHERE tablename = 'commandes' AND attname = 'created_at';  
```

**Résultat** :
```
 attname     | n_distinct | null_frac | correlation
-------------|------------|-----------|------------
 created_at  | 365        | 0.0       | 0.95
```

**Interprétation** :
- `correlation = 0.95` : Dates physiquement triées → Index très efficace  
- `n_distinct = 365` : Une année de dates

### 11.5. Étape 4 : Créer les Index Appropriés

```sql
-- Index 1 : Sur clients.ville (faible cardinalité → index partiel meilleur)
CREATE INDEX idx_clients_paris ON clients(id) WHERE ville = 'Paris';

-- Index 2 : Sur commandes.created_at (pour le range)
CREATE INDEX idx_commandes_date ON commandes(created_at);

-- Mettre à jour les statistiques
ANALYZE clients;  
ANALYZE commandes;  
```

### 11.6. Étape 5 : Retester

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT c.nom, COUNT(o.id)  
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris'  
  AND o.created_at > '2024-01-01'
GROUP BY c.nom;
```

**Nouveau plan** :
```
HashAggregate  (cost=8500.00..8550.00 rows=5000 width=20)
               (actual time=123.456..123.567 rows=5 loops=1)
  Group Key: c.nom
  ->  Hash Join  (cost=4000.00..8000.00 rows=5000 width=12)
                 (actual time=45.123..120.234 rows=5000 loops=1)
        Hash Cond: (o.client_id = c.id)
        ->  Index Scan using idx_commandes_date on commandes o
              (cost=0.43..3500.00 rows=50000 width=8)
              (actual time=0.012..80.123 rows=50000 loops=1)
              Index Cond: (created_at > '2024-01-01')
        ->  Hash  (cost=3800.00..3800.00 rows=35000 width=12)
                  (actual time=40.234..40.234 rows=35000 loops=1)
              ->  Index Scan using idx_clients_paris on clients c
                    (cost=0.42..3800.00 rows=35000 width=12)
                    (actual time=0.023..30.567 rows=35000 loops=1)
                    Index Cond: (ville = 'Paris')
Planning Time: 0.345 ms  
Execution Time: 124.123 ms  
```

**Résultat** :
- **Avant** : 7235 ms  
- **Après** : 124 ms  
- **Gain** : **58×** plus rapide ! 🚀

**Clés du succès** :
1. Index bien placés  
2. Statistiques à jour  
3. Planificateur a choisi les bons plans grâce aux statistiques

---

## Points Clés à Retenir

🔑 **Le planificateur est le cerveau de PostgreSQL** : Il choisit le plan d'exécution optimal basé sur les statistiques.

🔑 **pg_stats = base de décision** : Contient toutes les statistiques utilisées par le planificateur (n_distinct, MCV, histogrammes, corrélation).

🔑 **ANALYZE régulièrement** : Après chargements massifs, modifications de schéma, ou si les estimations sont fausses.

🔑 **Autovacuum s'occupe de l'automatisation** : Mais vérifiez qu'il s'exécute régulièrement avec `pg_stat_user_tables`.

🔑 **Comparaison estimations vs réalité** : Utilisez `EXPLAIN ANALYZE` pour détecter les mauvaises estimations (signe de statistiques obsolètes).

🔑 **default_statistics_target** : Augmentez pour colonnes critiques avec distributions complexes.

🔑 **Corrélation physique** : Impact majeur sur le choix Index Scan vs Seq Scan pour les ranges et ORDER BY.

🔑 **Monitoring proactif** : Surveillez `n_mod_since_analyze` pour anticiper les besoins de maintenance.

🔑 **Index + Statistiques = Duo gagnant** : Un index sans statistiques à jour est sous-utilisé.

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL** : [Planner Statistics](https://www.postgresql.org/docs/current/planner-stats.html)  
- **Section précédente** : 13.5. Stratégies d'indexation avancées  
- **Section suivante** : 13.7. Lecture et analyse d'un EXPLAIN  
- **Livre recommandé** : "The Art of PostgreSQL" par Dimitri Fontaine

---


⏭️ [Lecture et analyse d'un EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)](/13-indexation-et-optimisation/07-lecture-analyse-explain.md)
