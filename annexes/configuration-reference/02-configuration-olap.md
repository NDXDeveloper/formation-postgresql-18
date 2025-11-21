🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration PostgreSQL pour OLAP (Data Warehouse, Analytics)

## Introduction : Qu'est-ce que l'OLAP ?

### Définition

**OLAP** signifie **Online Analytical Processing** (Traitement Analytique en Ligne). C'est un type de charge de travail caractérisé par :

- **Requêtes complexes et longues** : Analyses qui peuvent durer de quelques secondes à plusieurs minutes
- **Agrégations massives** : Calculs sur des millions voire des milliards de lignes
- **Principalement des lectures** : Peu ou pas d'écritures pendant les heures d'analyse
- **Faible concurrence** : Quelques analystes/rapports simultanés (vs milliers d'utilisateurs en OLTP)
- **Données historiques** : Conservation de plusieurs années de données pour analyse de tendances

### Exemples d'Applications OLAP

- **Business Intelligence (BI)** : Tableaux de bord, rapports de ventes, KPIs
- **Data Warehouse** : Entrepôt de données consolidées de plusieurs sources
- **Analytics avancés** : Analyse de cohortes, segmentation clients, prévisions
- **Reporting financier** : Bilans, états financiers, analyses de rentabilité
- **Big Data Analytics** : Analyse de logs, télémétrie, IoT

### OLAP vs OLTP : Tableau Comparatif

| Critère | OLTP | OLAP |
|---------|------|------|
| **Objectif** | Gérer les opérations quotidiennes | Analyser et prendre des décisions |
| **Type d'opérations** | INSERT, UPDATE, DELETE fréquents | SELECT complexes avec GROUP BY, JOIN |
| **Volume par requête** | 1-1000 lignes | Millions à milliards de lignes |
| **Durée des requêtes** | < 100ms | Secondes à minutes |
| **Concurrence** | Très élevée (milliers) | Faible (quelques dizaines) |
| **Fraîcheur des données** | Temps réel | Batch quotidien/horaire acceptable |
| **Modèle de données** | Normalisé (3NF) | Dénormalisé (Star Schema, Snowflake) |
| **Index** | B-Tree sur clés primaires/étrangères | B-Tree + BRIN + vues matérialisées |
| **Optimisation** | Latence et throughput | Débit (throughput) de données |

> **Pour les débutants** : OLTP c'est la caisse du supermarché (rapide, beaucoup de clients), OLAP c'est le bureau du directeur qui analyse les ventes de l'année (lent, calculs complexes, peu de personnes).

---

## Architecture Typique d'un Data Warehouse

### Schéma en Étoile (Star Schema)

Le modèle le plus courant en OLAP :

```
                      ┌─────────────┐
                      │   DIM_TIME  │
                      │  (date_id)  │
                      └──────┬──────┘
                             │
        ┌────────────────────┼───────────────────┐
        │                    │                   │
   ┌────┴───────┐      ┌─────┴──────┐     ┌──────┴──────┐
   │DIM_PRODUCT │      │  FACT_SALES│     │DIM_CUSTOMER │
   │(product_id)│──────│(measure)   │─────│(customer_id)│
   └────────────┘      │sum, avg,etc│     └─────────────┘
                       └─────┬──────┘
                             │
                       ┌─────┴──────┐
                       │ DIM_STORE  │
                       │ (store_id) │
                       └────────────┘
```

**Composants :**
- **Tables de faits (FACT)** : Mesures numériques (ventes, quantités, revenus)
- **Tables de dimensions (DIM)** : Contexte (qui, quoi, où, quand)

**Exemple concret :**
```sql
-- Table de faits (des millions de lignes)
CREATE TABLE fact_sales (
    sale_id BIGINT PRIMARY KEY,
    date_id INT REFERENCES dim_time(date_id),
    product_id INT REFERENCES dim_product(product_id),
    customer_id INT REFERENCES dim_customer(customer_id),
    store_id INT REFERENCES dim_store(store_id),
    quantity INT,
    amount NUMERIC(10,2),
    cost NUMERIC(10,2)
);

-- Dimension Temps (quelques milliers de lignes)
CREATE TABLE dim_time (
    date_id INT PRIMARY KEY,
    date DATE,
    year INT,
    quarter INT,
    month INT,
    week INT,
    day_of_week INT,
    is_weekend BOOLEAN
);
```

---

## Objectifs de Configuration pour OLAP

Pour une charge de travail OLAP, nous cherchons à optimiser PostgreSQL selon ces principes :

1. **Maximiser le débit de données** (MB/s lus et traités)
2. **Paralléliser les requêtes** (utiliser tous les CPU)
3. **Optimiser les agrégations** (GROUP BY, SUM, AVG sur millions de lignes)
4. **Favoriser les scans séquentiels** (lire de grandes portions de tables)
5. **Réduire l'importance de la latence** (OK si requête prend 30s au lieu de 5s)
6. **Allouer beaucoup de mémoire par requête** (pour tris et hachages massifs)

---

## Paramètres Clés pour OLAP

### 1. Gestion de la Mémoire

#### **shared_buffers** : Cache Moins Critique qu'en OLTP

**Qu'est-ce que c'est ?**
Mémoire partagée pour le cache de pages. En OLAP, c'est moins critique car :
- Les requêtes scannent des millions de lignes (difficile de tout cacher)
- Le système d'exploitation (OS) gère bien le cache fichier pour les gros volumes

**Configuration recommandée pour OLAP :**
```ini
shared_buffers = 25% de la RAM (comme OLTP, mais moins critique)
```

**Exemples :**
- Serveur avec 64 GB RAM → `shared_buffers = 16GB`
- Serveur avec 128 GB RAM → `shared_buffers = 32GB`
- Serveur avec 256 GB RAM → `shared_buffers = 64GB`

**Configuration dans postgresql.conf :**
```ini
shared_buffers = 32GB
```

---

#### **effective_cache_size** : Très Important !

**Configuration recommandée pour OLAP :**
```ini
effective_cache_size = 75-80% de la RAM totale
```

**Pourquoi important ?**
Le planificateur de requêtes utilise cette valeur pour décider entre scan séquentiel et scan d'index. Avec beaucoup de cache, il peut mieux optimiser les plans complexes.

**Exemples :**
- Serveur avec 128 GB RAM → `effective_cache_size = 96GB` (75%)
- Serveur avec 256 GB RAM → `effective_cache_size = 200GB` (78%)

**Configuration dans postgresql.conf :**
```ini
effective_cache_size = 96GB
```

---

#### **work_mem** : LE Paramètre Critique pour OLAP

**Qu'est-ce que c'est ?**
Mémoire allouée pour les opérations de tri, hachage, et construction de bitmaps. En OLAP, les requêtes font des tris massifs et des GROUP BY sur des millions de lignes.

**⚠️ Différence majeure avec OLTP :**
- OLTP : work_mem faible (16-32MB) car beaucoup de connexions
- OLAP : work_mem élevé (256MB-2GB) car peu de connexions mais opérations massives

**Configuration recommandée pour OLAP :**
```ini
work_mem = 256MB à 2GB (selon RAM et nombre de connexions)
```

**Calcul pratique pour OLAP :**
```
work_mem = (RAM totale × 50%) / max_connections / 2
```

**Pourquoi 50% et non 25% comme OLTP ?**
Parce qu'en OLAP vous avez beaucoup moins de connexions simultanées (10-50 vs 200-500 en OLTP).

**Exemples :**
- Serveur 128 GB, 20 connexions → `work_mem = (128 × 0.5) / 20 / 2 = 1.6GB`
- Serveur 256 GB, 30 connexions → `work_mem = (256 × 0.5) / 30 / 2 = 2GB`

**Configuration dans postgresql.conf :**
```ini
work_mem = 1GB
```

**💡 Impact direct :**
- `work_mem` trop faible → PostgreSQL écrit sur disque (TEMP FILES) → requête 10× plus lente
- `work_mem` suffisant → Tout reste en RAM → requête rapide

**Vérifier l'utilisation :**
```sql
-- Voir les requêtes qui utilisent des fichiers temporaires
SELECT
    query,
    temp_blks_written
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC;
```

Si vous voyez beaucoup de `temp_blks_written`, augmentez `work_mem` !

---

#### **maintenance_work_mem** : Encore Plus Important qu'en OLTP

**Configuration recommandée pour OLAP :**
```ini
maintenance_work_mem = 2-8GB
```

**Pourquoi si élevé ?**
- Tables de faits avec des milliards de lignes
- Index massifs à construire ou reconstruire
- VACUUM sur tables énormes

**Configuration dans postgresql.conf :**
```ini
maintenance_work_mem = 4GB
```

---

#### **hash_mem_multiplier** : Nouveau PostgreSQL 13+

**Qu'est-ce que c'est ?**
Multiplicateur de mémoire pour les opérations de hachage (hash joins, hash aggregates).

**Configuration recommandée pour OLAP :**
```ini
hash_mem_multiplier = 2.0 à 3.0
```

**Exemple :**
Avec `work_mem = 1GB` et `hash_mem_multiplier = 2.0` :
- Opérations de hachage peuvent utiliser jusqu'à **2GB**
- Autres opérations (tri) restent à 1GB

**Configuration dans postgresql.conf :**
```ini
hash_mem_multiplier = 2.0
```

---

### 2. Parallélisation : Utiliser Tous les CPU

En OLAP, une seule requête doit pouvoir utiliser plusieurs CPU en parallèle pour traiter des millions de lignes plus rapidement.

#### **max_worker_processes** : Nombre Total de Workers

**Configuration recommandée pour OLAP :**
```ini
max_worker_processes = Nombre de CPU (ou plus)
```

**Exemples :**
- Serveur 16 CPU → `max_worker_processes = 16`
- Serveur 32 CPU → `max_worker_processes = 32`

**Configuration dans postgresql.conf :**
```ini
max_worker_processes = 32
```

---

#### **max_parallel_workers_per_gather** : Workers par Requête

**Qu'est-ce que c'est ?**
Nombre maximum de workers qu'une seule opération parallèle (Gather) peut utiliser.

**Configuration recommandée pour OLAP :**
```ini
max_parallel_workers_per_gather = 8 à 16
```

**Exemple :**
Si vous scannez une table de 100 millions de lignes, PostgreSQL peut diviser le travail en 8 morceaux traités en parallèle.

**Configuration dans postgresql.conf :**
```ini
max_parallel_workers_per_gather = 8
```

---

#### **max_parallel_workers** : Pool Global de Workers

**Configuration recommandée pour OLAP :**
```ini
max_parallel_workers = max_worker_processes (utiliser tous les workers disponibles)
```

**Configuration dans postgresql.conf :**
```ini
max_parallel_workers = 32
```

---

#### **max_parallel_maintenance_workers** : Parallélisation de Maintenance

**Qu'est-ce que c'est ?**
Nombre de workers pour opérations de maintenance (CREATE INDEX, VACUUM).

**Configuration recommandée pour OLAP :**
```ini
max_parallel_maintenance_workers = 4 à 8
```

**Impact :**
Création d'index sur table de 1 milliard de lignes :
- Sans parallélisation : 2 heures
- Avec 8 workers : 20 minutes

**Configuration dans postgresql.conf :**
```ini
max_parallel_maintenance_workers = 8
```

---

#### **parallel_setup_cost** et **parallel_tuple_cost** : Seuils de Parallélisation

**Par défaut :**
- `parallel_setup_cost = 1000` (coût d'initialisation du parallélisme)
- `parallel_tuple_cost = 0.1` (coût par tuple en parallèle)

**Configuration recommandée pour OLAP :**
```ini
parallel_setup_cost = 100  # Réduire pour favoriser parallélisation
parallel_tuple_cost = 0.01  # Réduire pour favoriser parallélisation
```

**Pourquoi ?**
En OLAP, les requêtes traitent des millions de lignes, donc le coût d'initialisation est négligeable. On veut paralléliser le plus possible.

---

### 3. Optimisation des Scans

#### **seq_page_cost** : Coût d'un Scan Séquentiel

**Par défaut :** 1.0

**Configuration recommandée pour OLAP (SSD/NVMe) :**
```ini
seq_page_cost = 1.0  (garder par défaut)
```

---

#### **random_page_cost** : Coût d'un Accès Aléatoire

**Configuration recommandée pour OLAP (SSD/NVMe) :**
```ini
random_page_cost = 1.0 à 1.2
```

**Pourquoi proche de seq_page_cost ?**
Avec des SSD/NVMe, les accès aléatoires sont presque aussi rapides que les séquentiels. De plus, en OLAP, on scanne souvent de larges portions de tables, donc les scans séquentiels sont préférables.

**Configuration dans postgresql.conf :**
```ini
random_page_cost = 1.1
```

---

#### **effective_io_concurrency** : I/O Simultanés

**Configuration recommandée pour OLAP (SSD/NVMe) :**
```ini
effective_io_concurrency = 200-300 (SSD)
effective_io_concurrency = 300-500 (NVMe)
```

**Configuration dans postgresql.conf :**
```ini
effective_io_concurrency = 400
```

---

#### **Nouveauté PostgreSQL 18 : I/O Asynchrone**

PostgreSQL 18 introduit le sous-système I/O asynchrone qui peut améliorer significativement les performances des scans séquentiels massifs.

**Configuration dans postgresql.conf :**
```ini
io_method = 'async'  # Nécessite Linux 5.1+ avec io_uring
```

**Impact typique :** Amélioration de 20-30% sur les requêtes avec gros scans séquentiels.

---

### 4. Gestion des Connexions

#### **max_connections** : Peu de Connexions en OLAP

**Configuration recommandée pour OLAP :**
```ini
max_connections = 20-100
```

**Pourquoi si peu ?**
- En OLAP, vous avez quelques analystes/rapports simultanés
- Chaque requête consomme beaucoup de ressources (CPU, RAM)
- 10 requêtes parallèles avec 8 workers chacune = 80 CPU utilisés !

**Configuration dans postgresql.conf :**
```ini
max_connections = 50
```

---

### 5. WAL et Checkpoints : Moins Critiques qu'en OLTP

#### **Configuration Recommandée**

En OLAP, vous avez peu d'écritures (chargements batch quotidiens/horaires). Les paramètres WAL sont donc moins critiques.

```ini
# WAL
wal_level = replica                     # Pour réplication si besoin
max_wal_size = 4GB                      # Suffisant pour OLAP
min_wal_size = 1GB
wal_compression = on                    # Économiser de l'espace

# Checkpoints (peut être plus espacés qu'en OLTP)
checkpoint_timeout = 30min              # Défaut 5min trop fréquent
checkpoint_completion_target = 0.9
```

---

### 6. Autovacuum : Configuration Différente

#### **Autovacuum en OLAP**

En OLAP :
- Peu de UPDATE/DELETE (données historiques, append-only souvent)
- Mais tables énormes → VACUUM prend du temps
- Besoin de statistiques à jour pour bon planning

**Configuration recommandée pour OLAP :**

```ini
# Autovacuum moins agressif qu'en OLTP
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 5min               # Moins fréquent qu'en OLTP (10s)

# Seuils plus élevés (tables plus grandes)
autovacuum_vacuum_threshold = 1000      # Défaut 50
autovacuum_vacuum_scale_factor = 0.1    # Défaut 0.2
autovacuum_analyze_threshold = 1000
autovacuum_analyze_scale_factor = 0.05  # Important : statistiques à jour !

# Laisser autovacuum consommer plus de ressources (moins de concurrence)
autovacuum_vacuum_cost_delay = 0        # Pas de ralentissement artificiel
autovacuum_vacuum_cost_limit = -1       # Pas de limite
```

**⚠️ ANALYZE est plus important que VACUUM en OLAP**
Des statistiques obsolètes = mauvais plans de requête = requêtes 100× plus lentes !

---

### 7. Statistiques pour le Planificateur

#### **default_statistics_target** : Très Important en OLAP

**Qu'est-ce que c'est ?**
Niveau de détail des statistiques collectées par ANALYZE. Plus c'est élevé, meilleur est le planning pour requêtes complexes.

**Valeur par défaut :** 100

**Configuration recommandée pour OLAP :**
```ini
default_statistics_target = 500 à 1000
```

**Impact :**
- Défaut (100) : Histogrammes avec 100 buckets
- OLAP (500) : Histogrammes avec 500 buckets → meilleure estimation de cardinalité

**⚠️ Coût :** ANALYZE prend plus de temps et consomme plus de mémoire, mais c'est acceptable en OLAP.

**Configuration dans postgresql.conf :**
```ini
default_statistics_target = 500
```

**Configuration par colonne (pour colonnes critiques) :**
```sql
-- Pour colonnes utilisées dans WHERE/JOIN/GROUP BY
ALTER TABLE fact_sales
    ALTER COLUMN date_id SET STATISTICS 1000;
```

---

### 8. Paramètres JIT (Just-In-Time Compilation)

PostgreSQL peut compiler les expressions SQL en code machine pour accélérer l'exécution.

#### **jit** : Activer le JIT

**Configuration recommandée pour OLAP :**
```ini
jit = on  # Activé par défaut dans PostgreSQL 11+
jit_above_cost = 100000  # Seuil d'activation (requêtes longues)
jit_inline_above_cost = 500000
jit_optimize_above_cost = 500000
```

**Impact :** Amélioration de 10-30% sur requêtes avec beaucoup d'expressions (calculs, CASE, etc.)

---

## Configuration Complète pour OLAP

Voici une configuration **complète** pour un serveur PostgreSQL 18 dédié à OLAP :

### Scénario : Serveur 128 GB RAM, 32 CPU, SSD NVMe, 30 connexions max

```ini
# ===================================
# MÉMOIRE
# ===================================
shared_buffers = 32GB                   # 25% de 128GB
effective_cache_size = 96GB             # 75% de 128GB
work_mem = 1GB                          # (128 × 0.5) / 30 / 2
maintenance_work_mem = 4GB
hash_mem_multiplier = 2.0               # 2× work_mem pour hash operations

# ===================================
# PARALLÉLISATION (crucial pour OLAP)
# ===================================
max_worker_processes = 32               # Tous les CPU
max_parallel_workers_per_gather = 8     # Jusqu'à 8 workers par opération
max_parallel_workers = 32               # Pool global
max_parallel_maintenance_workers = 8

# Favoriser parallélisation (réduire seuils)
parallel_setup_cost = 100               # Défaut 1000
parallel_tuple_cost = 0.01              # Défaut 0.1
min_parallel_table_scan_size = 8MB      # Défaut 8MB
min_parallel_index_scan_size = 512kB    # Défaut 512kB

# Force parallélisation pour tests (désactiver en prod)
# force_parallel_mode = on

# ===================================
# CONNEXIONS (peu en OLAP)
# ===================================
max_connections = 50
superuser_reserved_connections = 3

# ===================================
# WAL et CHECKPOINTS
# ===================================
wal_level = replica
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_timeout = 30min              # Plus espacé qu'en OLTP
checkpoint_completion_target = 0.9
wal_compression = on
wal_buffers = 64MB

# Durabilité
synchronous_commit = on
fsync = on

# ===================================
# AUTOVACUUM (adapté OLAP)
# ===================================
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 5min

# Seuils plus élevés (grandes tables)
autovacuum_vacuum_threshold = 1000
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_threshold = 1000
autovacuum_analyze_scale_factor = 0.05

# Pas de throttling (peu de concurrence)
autovacuum_vacuum_cost_delay = 0
autovacuum_vacuum_cost_limit = -1

# ===================================
# PLANIFICATEUR (optimisé OLAP + SSD)
# ===================================
random_page_cost = 1.1                  # SSD/NVMe
seq_page_cost = 1.0
effective_io_concurrency = 400          # NVMe

# Statistiques détaillées (crucial OLAP)
default_statistics_target = 500

# JIT pour requêtes complexes
jit = on
jit_above_cost = 100000
jit_inline_above_cost = 500000
jit_optimize_above_cost = 500000

# PostgreSQL 18 : I/O asynchrone
io_method = 'async'

# ===================================
# LIMITES DE REQUÊTE
# ===================================
# Optionnel : limites pour éviter requêtes infinies
# statement_timeout = 600000            # 10 minutes max
# idle_in_transaction_session_timeout = 300000  # 5 min

# ===================================
# LOGGING et MONITORING
# ===================================
logging_collector = on
log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 500MB               # Plus gros qu'en OLTP

log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_min_duration_statement = 5000       # Log requêtes > 5 secondes
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                      # Log tous fichiers temp (crucial OLAP)
log_autovacuum_min_duration = 0         # Log tous autovacuum

# Monitoring étendu
track_activities = on
track_counts = on
track_io_timing = on
track_functions = pl
compute_query_id = on                   # Pour pg_stat_statements

# Extensions monitoring
shared_preload_libraries = 'pg_stat_statements'

# ===================================
# SÉCURITÉ
# ===================================
ssl = on
ssl_prefer_server_ciphers = on
password_encryption = scram-sha-256

# ===================================
# TIMEZONE et LOCALE
# ===================================
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

---

## Optimisations Spécifiques OLAP

### 1. Partitionnement : Indispensable pour Grandes Tables

#### **Pourquoi Partitionner en OLAP ?**

Tables de faits avec plusieurs années de données = milliards de lignes. Avantages du partitionnement :
- **Partition Pruning** : PostgreSQL ignore les partitions non concernées
- **Maintenance ciblée** : VACUUM/ANALYZE seulement les partitions récentes
- **Archivage facile** : Détacher partitions anciennes
- **Requêtes plus rapides** : Traiter 50M lignes au lieu de 2B

#### **Exemple : Partitionnement par Mois**

```sql
-- Table parent (partitionnée)
CREATE TABLE fact_sales (
    sale_id BIGINT,
    sale_date DATE NOT NULL,
    product_id INT,
    customer_id INT,
    amount NUMERIC(10,2),
    quantity INT
) PARTITION BY RANGE (sale_date);

-- Créer des partitions pour chaque mois
CREATE TABLE fact_sales_2024_01 PARTITION OF fact_sales
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE fact_sales_2024_02 PARTITION OF fact_sales
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- ... et ainsi de suite
```

#### **Requête avec Partition Pruning**

```sql
-- Analyse des ventes de janvier 2024
SELECT product_id, SUM(amount)
FROM fact_sales
WHERE sale_date >= '2024-01-01' AND sale_date < '2024-02-01'
GROUP BY product_id;

-- PostgreSQL scanne SEULEMENT fact_sales_2024_01
-- (50M lignes au lieu de 2B lignes)
```

#### **Automatiser avec pg_partman**

Extension pour gérer automatiquement les partitions :

```sql
CREATE EXTENSION pg_partman;

-- Créer partitions automatiquement
SELECT partman.create_parent(
    p_parent_table => 'public.fact_sales',
    p_control => 'sale_date',
    p_type => 'native',
    p_interval => 'monthly',
    p_premake => 3  -- Créer 3 mois à l'avance
);

-- Job pour maintenir les partitions
SELECT partman.run_maintenance('public.fact_sales');
```

---

### 2. Index Adaptés à OLAP

#### **BRIN : L'Index OLAP par Excellence**

**BRIN** (Block Range Index) est idéal pour OLAP :
- **Très compact** : 1000× plus petit qu'un B-Tree
- **Données séquentielles** : Parfait pour tables partitionnées par date
- **Faible maintenance** : Peu d'espace disque

**Exemple :**

```sql
-- Table de 1 milliard de lignes triées par date
-- B-Tree : 20 GB
-- BRIN : 20 MB

CREATE INDEX idx_sales_date_brin ON fact_sales
    USING BRIN (sale_date) WITH (pages_per_range = 128);
```

**Quand utiliser BRIN ?**
- Colonnes avec corrélation physique (date d'insertion ≈ ordre sur disque)
- Tables en append-only (INSERT uniquement, pas de UPDATE)
- Requêtes avec prédicats sur plages (WHERE date BETWEEN ... AND ...)

---

#### **B-Tree : Toujours Utile**

Pour les dimensions (petites tables) :

```sql
-- Dimensions : B-Tree classiques
CREATE INDEX idx_dim_product_id ON dim_product(product_id);
CREATE INDEX idx_dim_customer_id ON dim_customer(customer_id);
```

---

#### **GIN pour Colonnes JSONB**

Si vous stockez des données semi-structurées :

```sql
-- Colonne JSONB avec métadonnées flexibles
ALTER TABLE fact_sales ADD COLUMN metadata JSONB;

CREATE INDEX idx_sales_metadata_gin ON fact_sales
    USING GIN (metadata jsonb_path_ops);

-- Requête rapide sur JSONB
SELECT * FROM fact_sales
WHERE metadata @> '{"campaign": "summer2024"}';
```

---

### 3. Vues Matérialisées : Pré-Calculer les Agrégations

#### **Concept**

Une vue matérialisée stocke physiquement le résultat d'une requête complexe. C'est comme un "cache" de résultats.

**Avantages :**
- Requêtes instantanées (déjà calculées)
- Réduire charge sur tables de faits

**Inconvénients :**
- Données pas en temps réel (besoin de REFRESH)
- Consomme de l'espace disque

#### **Exemple : Ventes par Mois et Produit**

```sql
-- Calcul initial : 30 secondes sur 1 milliard de lignes
CREATE MATERIALIZED VIEW mv_sales_by_month_product AS
SELECT
    DATE_TRUNC('month', sale_date) AS month,
    product_id,
    SUM(amount) AS total_amount,
    SUM(quantity) AS total_quantity,
    COUNT(*) AS num_sales
FROM fact_sales
GROUP BY DATE_TRUNC('month', sale_date), product_id;

-- Index sur la vue matérialisée
CREATE INDEX idx_mv_sales_month_product
    ON mv_sales_by_month_product(month, product_id);

-- Requête sur vue : < 10ms (au lieu de 30 secondes)
SELECT * FROM mv_sales_by_month_product
WHERE month = '2024-01-01';
```

#### **Rafraîchir les Données**

```sql
-- Rafraîchissement complet (bloquant)
REFRESH MATERIALIZED VIEW mv_sales_by_month_product;

-- Rafraîchissement concurrent (non bloquant, nécessite UNIQUE index)
CREATE UNIQUE INDEX ON mv_sales_by_month_product(month, product_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_by_month_product;
```

#### **Automatiser avec pg_cron**

```sql
CREATE EXTENSION pg_cron;

-- Rafraîchir chaque jour à 2h du matin
SELECT cron.schedule(
    'refresh-sales-mv',
    '0 2 * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_by_month_product'
);
```

---

### 4. Modèle Dénormalisé (Star Schema)

#### **Principe : Joindre Moins, Filtrer Plus**

En OLAP, les jointures sur tables massives sont coûteuses. Stratégie : **dénormaliser** en dupliquant certaines colonnes dans la table de faits.

**Avant (normalisé) :**
```sql
-- Jointure à chaque requête (lent)
SELECT
    d.category,
    SUM(f.amount)
FROM fact_sales f
JOIN dim_product d ON f.product_id = d.product_id
WHERE d.category = 'Electronics'
GROUP BY d.category;
```

**Après (dénormalisé) :**
```sql
-- Ajouter category directement dans fact_sales
ALTER TABLE fact_sales ADD COLUMN product_category VARCHAR(50);

-- Maintenant : pas de jointure !
SELECT
    product_category,
    SUM(amount)
FROM fact_sales
WHERE product_category = 'Electronics'
GROUP BY product_category;
-- 10× plus rapide
```

**Compromis :**
- ✅ Requêtes plus rapides
- ✅ Moins de jointures
- ❌ Duplication de données (espace disque)
- ❌ Maintenance ETL plus complexe

---

### 5. Colonnes Compressées (TOAST et pg_column_compression)

PostgreSQL compresse automatiquement les grandes valeurs (TOAST), mais vous pouvez forcer la compression pour économiser l'espace :

```sql
-- PostgreSQL 14+ : Compression par colonne
ALTER TABLE fact_sales
    ALTER COLUMN description SET COMPRESSION lz4;

-- Ou pour toute nouvelle donnée
SET default_toast_compression = 'lz4';
```

**Gain typique :** 50-70% d'économie d'espace pour colonnes texte.

---

### 6. Nouveauté PostgreSQL 18 : Colonnes Générées Virtuelles

**Concept :** Colonnes calculées à la volée (non stockées physiquement), utiles pour simplifier les requêtes.

```sql
-- Avant PostgreSQL 18 : Calculer à chaque requête
SELECT
    amount * 1.20 AS amount_with_tax
FROM fact_sales;

-- PostgreSQL 18 : Colonne virtuelle
ALTER TABLE fact_sales
    ADD COLUMN amount_with_tax NUMERIC GENERATED ALWAYS AS (amount * 1.20) VIRTUAL;

-- Requête simplifiée
SELECT amount_with_tax FROM fact_sales;
```

**Avantage :** Pas de stockage supplémentaire, mais requêtes plus lisibles.

---

### 7. Requêtes Parallèles : Vérifier la Parallélisation

#### **EXPLAIN pour Vérifier**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, SUM(amount)
FROM fact_sales
WHERE sale_date >= '2024-01-01'
GROUP BY product_id;
```

**Sortie attendue pour OLAP :**
```
Finalize GroupAggregate (actual time=1234.56..1234.78 rows=1000)
  -> Gather Merge (actual time=1000.12..1200.34 rows=8000)
        Workers Planned: 8
        Workers Launched: 8
        -> Partial GroupAggregate
              -> Parallel Seq Scan on fact_sales (actual time=...)
                    Filter: (sale_date >= '2024-01-01')
```

**Points clés :**
- `Workers Planned: 8` → Parallélisation activée
- `Parallel Seq Scan` → Scan séquentiel parallèle
- `Gather Merge` → Fusion des résultats de chaque worker

**Si pas de parallélisation :**
- Vérifier `max_parallel_workers_per_gather` (doit être > 1)
- Vérifier que la table est assez grande (`min_parallel_table_scan_size`)
- Vérifier `parallel_setup_cost` (réduire si nécessaire)

---

## Monitoring Spécifique OLAP

### 1. Requêtes Utilisant des Fichiers Temporaires

**Problème :** Si `work_mem` est trop faible, PostgreSQL écrit sur disque.

```sql
-- Trouver les requêtes avec beaucoup de fichiers temp
SELECT
    query,
    calls,
    mean_exec_time,
    temp_blks_written,
    temp_blks_written * 8192 / 1024 / 1024 AS temp_mb
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

**Action :** Si vous voyez beaucoup de MB dans `temp_mb`, augmentez `work_mem`.

---

### 2. Efficacité de la Parallélisation

```sql
-- Voir les requêtes qui bénéficient de parallélisation
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_time_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
WHERE query LIKE '%Parallel%'
ORDER BY total_exec_time DESC;
```

---

### 3. État des Vues Matérialisées

```sql
-- Trouver les vues matérialisées
SELECT
    schemaname,
    matviewname,
    pg_size_pretty(pg_relation_size(schemaname||'.'||matviewname)) AS size
FROM pg_matviews;

-- Dernière date de refresh (nécessite tracking manuel ou pg_cron logs)
```

---

### 4. Taille des Tables et Index

```sql
-- Top 10 tables par taille
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

---

### 5. Statistiques ANALYZE

```sql
-- Vérifier si les statistiques sont à jour
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY last_analyze ASC NULLS FIRST;
```

**⚠️ Alerte :** Si `last_analyze` est NULL ou très ancien, lancez :
```sql
ANALYZE fact_sales;
```

---

## Checklist de Vérification OLAP

Avant de mettre en production un data warehouse, vérifiez :

### Mémoire et CPU
- [ ] **shared_buffers** = 25% RAM
- [ ] **effective_cache_size** = 75% RAM
- [ ] **work_mem** = (RAM × 50%) / max_connections / 2 (au moins 256MB)
- [ ] **maintenance_work_mem** ≥ 2GB
- [ ] **max_worker_processes** = Nombre de CPU
- [ ] **max_parallel_workers_per_gather** ≥ 4
- [ ] **max_parallel_workers** = max_worker_processes

### Parallélisation
- [ ] **parallel_setup_cost** = 100 (réduit)
- [ ] **parallel_tuple_cost** = 0.01 (réduit)
- [ ] Vérifier avec EXPLAIN que requêtes sont parallélisées

### Planificateur
- [ ] **default_statistics_target** ≥ 500
- [ ] **random_page_cost** = 1.1 (SSD/NVMe)
- [ ] **jit** = on

### Stockage et I/O
- [ ] **PostgreSQL 18** : io_method = 'async' (si compatible)
- [ ] **effective_io_concurrency** ≥ 200

### Autovacuum et Maintenance
- [ ] **autovacuum** = on
- [ ] **autovacuum_analyze_scale_factor** = 0.05
- [ ] **autovacuum_vacuum_cost_delay** = 0
- [ ] ANALYZE exécuté après chargements ETL

### Modélisation
- [ ] Tables de faits partitionnées par date
- [ ] Index BRIN sur colonnes de date
- [ ] Index B-Tree sur dimensions
- [ ] Vues matérialisées pour agrégations fréquentes
- [ ] Dénormalisation appliquée où pertinent

### Monitoring
- [ ] pg_stat_statements activé
- [ ] Logs des requêtes > 5 secondes
- [ ] Logs des fichiers temporaires (log_temp_files = 0)
- [ ] Surveillance taille des tables/index
- [ ] Surveillance état vues matérialisées

---

## Erreurs Courantes à Éviter

### ❌ Erreur 1 : work_mem Trop Faible
**Symptôme :** Requêtes très lentes, logs montrent "temporary file"
**Solution :** Augmenter work_mem progressivement (256MB → 512MB → 1GB)

### ❌ Erreur 2 : Pas de Partitionnement sur Tables Massives
**Symptôme :** Requêtes scannent toute la table (milliards de lignes)
**Solution :** Partitionner par date/région/autre critère fréquent

### ❌ Erreur 3 : Statistiques Obsolètes
**Symptôme :** Plans de requête absurdes, estimations erronées
**Solution :** ANALYZE après chaque chargement ETL

### ❌ Erreur 4 : Index B-Tree sur Tout
**Symptôme :** Maintenance lente, index énormes, peu utilisés
**Solution :** Utiliser BRIN pour colonnes séquentielles

### ❌ Erreur 5 : Pas de Vues Matérialisées
**Symptôme :** Mêmes agrégations recalculées à chaque fois (lent)
**Solution :** Créer vues matérialisées pour KPIs fréquents

### ❌ Erreur 6 : Parallélisation Désactivée
**Symptôme :** Un seul CPU utilisé malgré serveur 32 CPU
**Solution :** Vérifier paramètres `max_parallel_*`, réduire `parallel_setup_cost`

### ❌ Erreur 7 : Trop de Normalisation
**Symptôme :** Jointures sur 5+ tables, requêtes extrêmement lentes
**Solution :** Dénormaliser (dupliquer colonnes dans table de faits)

---

## ETL et Chargement de Données

### Stratégies de Chargement pour OLAP

#### **1. COPY : Le Plus Rapide**

```sql
-- Charger 10 millions de lignes en quelques secondes
COPY fact_sales FROM '/data/sales_2024_01.csv'
    WITH (FORMAT csv, HEADER true, DELIMITER ',');
```

**Optimisations :**
- Désactiver temporairement les triggers et contraintes
- Augmenter `maintenance_work_mem` pendant le chargement
- Créer les index APRÈS le chargement

```sql
-- Désactiver contraintes
ALTER TABLE fact_sales DISABLE TRIGGER ALL;

-- Charger données
COPY fact_sales FROM ...;

-- Réactiver contraintes
ALTER TABLE fact_sales ENABLE TRIGGER ALL;

-- Créer index en parallèle
CREATE INDEX CONCURRENTLY idx_sales_date ON fact_sales(sale_date);
```

---

#### **2. Nouveauté PostgreSQL 18 : Améliorations COPY**

PostgreSQL 18 améliore significativement COPY :
- Gestion du marqueur de fin `\.` en CSV
- Performances améliorées (jusqu'à 30% plus rapide)

---

#### **3. Chargement Incrémental**

```sql
-- Charger seulement les nouvelles données (delta)
INSERT INTO fact_sales
SELECT * FROM staging_sales
WHERE sale_date >= CURRENT_DATE - INTERVAL '1 day'
ON CONFLICT (sale_id) DO NOTHING;
```

---

#### **4. Partitions et ETL**

**Stratégie :** Charger dans partition temporaire, puis attacher.

```sql
-- 1. Créer table temporaire
CREATE TABLE fact_sales_2024_01_temp (LIKE fact_sales INCLUDING ALL);

-- 2. Charger données
COPY fact_sales_2024_01_temp FROM '/data/sales_2024_01.csv' WITH CSV;

-- 3. Créer index
CREATE INDEX ON fact_sales_2024_01_temp(sale_date);

-- 4. Attacher comme partition
ALTER TABLE fact_sales ATTACH PARTITION fact_sales_2024_01_temp
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Avantage :** Chargement et indexation en dehors de la table principale (pas de lock).

---

## Architectures Avancées OLAP

### 1. Data Warehouse Distribué

Pour volumes extrêmes (> 10 TB), considérez des solutions distribuées :

- **Citus** : Extension PostgreSQL pour sharding automatique
- **Greenplum** : Fork PostgreSQL MPP (Massively Parallel Processing)
- **TimescaleDB** : Extension pour séries temporelles

**Exemple Citus :**
```sql
CREATE EXTENSION citus;

-- Distribuer table de faits sur 4 nœuds
SELECT create_distributed_table('fact_sales', 'customer_id');
```

---

### 2. Columnar Storage

PostgreSQL est orienté lignes (row-based), mais pour analytics purs, le stockage colonnes est plus efficace.

**Options :**
- **Hydra Columnar** : Extension columnar pour PostgreSQL
- **pg_analytics** : Autre extension columnar

**Avantage :** Lire seulement les colonnes nécessaires (compression 10×, requêtes 5-10× plus rapides).

---

### 3. Intégration avec Outils BI

**Connexions typiques :**
- **Tableau** → PostgreSQL (via driver natif)
- **Power BI** → PostgreSQL (via ODBC/DirectQuery)
- **Metabase** → PostgreSQL (open-source BI)
- **Apache Superset** → PostgreSQL

**Bonnes pratiques :**
- Utiliser vues matérialisées pour dashboards
- Limiter nombre de connexions (pooling)
- Pré-agréger au maximum

---

## Ressources pour Aller Plus Loin

### Documentation Officielle
- [PostgreSQL Performance Optimization](https://www.postgresql.org/docs/18/performance-tips.html)
- [Partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html)
- [Parallel Query](https://www.postgresql.org/docs/18/parallel-query.html)

### Extensions Essentielles
- **pg_stat_statements** : Analyse de requêtes
- **pg_partman** : Gestion automatisée partitions
- **pg_cron** : Planification de tâches (refresh MV, vacuum, etc.)
- **TimescaleDB** : Séries temporelles
- **Citus** : Distribution multi-nœuds

### Outils
- **PGTune** : https://pgtune.leopard.in.ua/ (choisir "Data warehouse")
- **pgBadger** : Analyse de logs
- **Apache Hop / Airflow** : Orchestration ETL

### Lectures Recommandées
- *The Data Warehouse Toolkit* (Ralph Kimball) : Modélisation dimensionnelle
- *PostgreSQL Query Optimization* : Techniques avancées
- Blogs : Percona, Crunchy Data, 2ndQuadrant

---

## Conclusion

La configuration OLAP de PostgreSQL vise à :

1. **Maximiser le débit de données** (GB/s lus et traités)
2. **Paralléliser massivement** (utiliser tous les CPU)
3. **Optimiser pour agrégations** (work_mem élevé, statistiques détaillées)
4. **Partitionner intelligemment** (Partition Pruning)
5. **Pré-calculer** (vues matérialisées)
6. **Monitorer** (temp files, parallélisation, statistiques)

**Points clés à retenir :**

- ✅ **work_mem élevé** est LE paramètre critique (256MB-2GB)
- ✅ **Parallélisation** essentielle (max_parallel_workers_per_gather ≥ 4)
- ✅ **Partitionnement** obligatoire pour tables > 100M lignes
- ✅ **BRIN** index idéal pour colonnes séquentielles
- ✅ **Vues matérialisées** pour agrégations fréquentes
- ✅ **Statistiques détaillées** (default_statistics_target ≥ 500)
- ✅ **ANALYZE** après chaque chargement ETL

**Différences clés OLTP vs OLAP :**

| Paramètre | OLTP | OLAP |
|-----------|------|------|
| work_mem | 16-32MB | 256MB-2GB |
| max_connections | 200-500 | 20-100 |
| max_parallel_workers_per_gather | 2-4 | 8-16 |
| default_statistics_target | 100 | 500-1000 |
| autovacuum_naptime | 10s | 5min |
| Partitionnement | Optionnel | Obligatoire |
| Vues matérialisées | Rare | Fréquent |

PostgreSQL 18 apporte des améliorations significatives pour OLAP :
- I/O asynchrone (20-30% plus rapide)
- Colonnes virtuelles
- Optimisations du planificateur (skip scan, OR→ANY)
- Améliorations COPY

**Prochaines Étapes :**

1. Appliquer configuration sur environnement dev/staging
2. Charger données représentatives
3. Tester requêtes typiques avec EXPLAIN ANALYZE
4. Ajuster work_mem et parallélisation selon résultats
5. Créer partitions et vues matérialisées
6. Mettre en place monitoring (pg_stat_statements)
7. Automatiser ETL et refresh vues matérialisées

Bonne chance avec votre Data Warehouse PostgreSQL ! 📊🚀

⏭️ [Mixed workload (équilibre OLTP/OLAP)](/annexes/configuration-reference/03-configuration-mixed-workload.md)
