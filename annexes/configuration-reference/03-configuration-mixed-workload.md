🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration PostgreSQL pour Mixed Workload (Équilibre OLTP/OLAP)

## Introduction : Qu'est-ce qu'un Mixed Workload ?

### Définition

Un **Mixed Workload** (charge de travail mixte) est un scénario où PostgreSQL doit gérer **simultanément** :

- **Des opérations OLTP** : Transactions courtes, écritures fréquentes, faible latence  
- **Des requêtes OLAP** : Analyses complexes, agrégations massives, longues durées

C'est le cas le plus courant en production réelle, mais aussi le plus complexe à optimiser.

### Exemples d'Applications Mixed Workload

- **Application SaaS avec analytics intégré** :
  - OLTP : Utilisateurs créent/modifient des données en temps réel
  - OLAP : Dashboards et rapports sur ces mêmes données

- **E-commerce avec BI** :
  - OLTP : Commandes, paiements, gestion du panier
  - OLAP : Analyses de ventes, prévisions de stock, segmentation clients

- **CRM** :
  - OLTP : Gestion des contacts, opportunités, tâches
  - OLAP : Rapports de performance commerciale, funnel d'acquisition

- **Application mobile avec analytics** :
  - OLTP : API backend pour l'application
  - OLAP : Analyses d'usage, métriques produit

### Le Défi du Mixed Workload

Le problème fondamental : **les optimisations OLTP et OLAP sont contradictoires**.

| Aspect | Optimisation OLTP | Optimisation OLAP | Conflit |
|--------|-------------------|-------------------|---------|
| **work_mem** | Faible (16MB) | Élevé (1GB) | ⚠️ |
| **Connexions** | Nombreuses (300+) | Peu (30) | ⚠️ |
| **Index** | B-Tree partout | BRIN + sélectif | ⚠️ |
| **Parallélisation** | Limitée | Maximale | ⚠️ |
| **Autovacuum** | Agressif | Modéré | ⚠️ |

**Question clé :** Comment trouver le bon équilibre ?

---

## Stratégies d'Architecture

Avant même de configurer PostgreSQL, vous avez plusieurs options architecturales :

### Option 1 : Séparation Physique (Recommandée si Possible)

**Principe :** Deux instances PostgreSQL séparées

```
┌─────────────────┐
│  APP OLTP       │
│  (transactions) │
└────────┬────────┘
         │
         v
┌────────────────────┐      Réplication      ┌────────────────────┐
│ PostgreSQL PRIMARY │ ─────────────────────>│ PostgreSQL REPLICA │
│     (OLTP)         │      (async/sync)     │     (OLAP)         │
│                    │                       │   Read-Only        │
└────────────────────┘                       └──────────┬─────────┘
                                                        │
                                                        v
                                             ┌──────────────────┐
                                             │   DASHBOARDS     │
                                             │   ANALYTICS      │
                                             └──────────────────┘
```

**Avantages :**
- ✅ Isolation totale : OLAP n'impacte pas OLTP  
- ✅ Configuration optimale pour chaque charge  
- ✅ Scaling indépendant  
- ✅ Sécurité : replica read-only

**Inconvénients :**
- ❌ Coût : Deux serveurs  
- ❌ Complexité opérationnelle  
- ❌ Latence de réplication (données pas 100% fraîches)

**Quand utiliser :**
- Budget le permet
- OLAP consomme > 30% des ressources
- Latence de quelques secondes acceptable pour analytics

---

### Option 2 : Instance Unique avec Séparation Logique

**Principe :** Une instance, mais isolation par bases de données ou schémas

```
┌──────────────────────────────────┐
│    PostgreSQL Instance           │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Database: prod_oltp       │  │  ← Connexions app
│  │  (tables transactionnelles)│  │
│  └────────────────────────────┘  │
│                                  │
│  ┌──────────────────────────┐    │
│  │  Database: prod_analytics│    │  ← Connexions BI
│  │  (vues matérialisées,    │    │
│  │   tables agrégées)       │    │
│  └──────────────────────────┘    │
└──────────────────────────────────┘
```

**Avantages :**
- ✅ Un seul serveur à gérer  
- ✅ Données fraîches (même instance)  
- ✅ Moins coûteux

**Inconvénients :**
- ⚠️ Compétition pour ressources (CPU, RAM, I/O)  
- ⚠️ Configuration = compromis

**Quand utiliser :**
- Budget limité
- OLAP léger (< 30% des ressources)
- Données doivent être fraîches (temps réel)

---

### Option 3 : Instance Unique avec Gestion de Ressources

**Principe :** PostgreSQL avec connection pooling et isolation par rôles

```
┌─────────────┐                    ┌─────────────────────┐
│  APP OLTP   │──> PgBouncer ────> │                     │
└─────────────┘    (pool OLTP)     │                     │
                                   │   PostgreSQL        │
┌─────────────┐                    │                     │
│  DASHBOARDS │──> PgBouncer ────> │  - Resource groups  │
└─────────────┘    (pool OLAP)     │  - Query timeouts   │
                                   │  - work_mem par role│
                                   └─────────────────────┘
```

**Techniques :**
- Connection pooling séparé (OLTP vs OLAP)
- `work_mem` par rôle/session
- Query timeouts différenciés
- Priority queuing (PgBouncer)

**Quand utiliser :**
- Architecture actuelle avec instance unique
- Besoin de contrôle fin
- OLAP modéré (20-40% des ressources)

---

## Configuration Mixed Workload : Le Compromis

### Philosophie Générale

**Règle d'or :** Prioriser OLTP (latence critique) tout en permettant OLAP (throughput).

**Approche :**
1. Configuration de base = **70% OLTP, 30% OLAP**  
2. Ajustements dynamiques par session/rôle  
3. Monitoring pour détecter déséquilibres

---

## Paramètres Clés pour Mixed Workload

### 1. Gestion de la Mémoire

#### **shared_buffers** : Valeur Modérée

**Configuration recommandée :**
```ini
shared_buffers = 25% de la RAM (standard)
```

**Raisonnement :**
- 25% fonctionne bien pour les deux types de charges
- Plus n'améliore pas nécessairement les performances

**Exemples :**
- Serveur 64 GB RAM → `shared_buffers = 16GB`
- Serveur 128 GB RAM → `shared_buffers = 32GB`

**Configuration dans postgresql.conf :**
```ini
shared_buffers = 16GB
```

---

#### **effective_cache_size** : Indicateur pour le Planificateur

**Configuration recommandée :**
```ini
effective_cache_size = 75% de la RAM
```

**Configuration dans postgresql.conf :**
```ini
effective_cache_size = 48GB  # Pour 64GB RAM
```

---

#### **work_mem** : La Clé du Compromis

**Le Dilemme :**
- OLTP a besoin : 16-32MB (beaucoup de connexions)
- OLAP a besoin : 256MB-2GB (opérations massives)

**Solution 1 : Valeur Globale Modérée**

```ini
work_mem = 64MB  # Compromis entre 16MB (OLTP) et 1GB (OLAP)
```

**Calcul :**
```
work_mem = (RAM × 0.30) / max_connections / 2
```

**Exemple :** Serveur 64GB, 200 connexions
```
work_mem = (64 × 0.30) / 200 / 2 = 48MB ≈ 64MB
```

**Solution 2 : Ajustement Dynamique par Rôle (Recommandée)**

```sql
-- Rôle OLTP : work_mem faible
CREATE ROLE app_oltp WITH LOGIN PASSWORD 'xxx';  
ALTER ROLE app_oltp SET work_mem = '32MB';  

-- Rôle OLAP : work_mem élevé
CREATE ROLE analytics_user WITH LOGIN PASSWORD 'xxx';  
ALTER ROLE analytics_user SET work_mem = '512MB';  
```

**Avantage :** Chaque type de charge a sa propre configuration.

**Solution 3 : Ajustement par Session**

Application peut ajuster dynamiquement :

```python
# Python avec psycopg3
# Requête OLTP normale
conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# Requête OLAP : augmenter work_mem temporairement
conn.execute("SET LOCAL work_mem = '1GB'")  
conn.execute("""  
    SELECT product_id, SUM(amount)
    FROM fact_sales
    WHERE sale_date >= '2024-01-01'
    GROUP BY product_id
""")
```

**Configuration finale dans postgresql.conf :**
```ini
work_mem = 64MB  # Valeur par défaut (compromis)
# Ajustements par rôle ou session pour OLAP
```

---

#### **maintenance_work_mem** : Important pour les Deux

**Configuration recommandée :**
```ini
maintenance_work_mem = 1-2GB
```

**Raisonnement :**
- OLTP a besoin d'index et autovacuum efficaces
- OLAP a besoin de maintenance rapide sur grandes tables

**Configuration dans postgresql.conf :**
```ini
maintenance_work_mem = 2GB
```

---

#### **hash_mem_multiplier** : Favoriser OLAP Légèrement

**Configuration recommandée :**
```ini
hash_mem_multiplier = 1.5
```

**Configuration dans postgresql.conf :**
```ini
hash_mem_multiplier = 1.5
```

---

### 2. Gestion des Connexions

#### **max_connections** : Dimensionner pour OLTP

**Configuration recommandée :**
```ini
max_connections = 200-300
```

**Raisonnement :**
- OLTP a besoin de beaucoup de connexions
- OLAP utilise peu de connexions mais longues

**Stratégie :**
- Connection pooling (PgBouncer) pour OLTP
- Connexions directes pour OLAP (queries longues)

**Configuration dans postgresql.conf :**
```ini
max_connections = 250
```

---

### 3. Parallélisation : Modérée

#### **Compromis Parallélisation**

**Configuration recommandée :**
```ini
max_worker_processes = 16  # Nombre de CPU  
max_parallel_workers_per_gather = 4  # Compromis (OLTP: 2, OLAP: 8)  
max_parallel_workers = 12  # 75% des workers (laisser de la marge)  
max_parallel_maintenance_workers = 4  
```

**Raisonnement :**
- OLTP ne bénéficie pas beaucoup de parallélisation
- OLAP en a besoin mais pas au détriment de OLTP
- Limiter `max_parallel_workers` à 75% laisse des CPU pour OLTP

**Ajustement par rôle :**
```sql
-- Désactiver parallélisation pour OLTP (latence plus prévisible)
ALTER ROLE app_oltp SET max_parallel_workers_per_gather = 0;

-- Activer pour OLAP
ALTER ROLE analytics_user SET max_parallel_workers_per_gather = 8;
```

**Configuration dans postgresql.conf :**
```ini
max_worker_processes = 16  
max_parallel_workers_per_gather = 4  
max_parallel_workers = 12  
max_parallel_maintenance_workers = 4  

# Seuils de parallélisation modérés
parallel_setup_cost = 500  # Entre OLTP (1000) et OLAP (100)  
parallel_tuple_cost = 0.05  # Entre OLTP (0.1) et OLAP (0.01)  
```

---

### 4. WAL et Checkpoints : Optimiser pour OLTP

En mixed workload, **prioriser OLTP** pour les écritures.

**Configuration recommandée :**
```ini
wal_buffers = 32MB  
max_wal_size = 4GB  
min_wal_size = 1GB  
checkpoint_timeout = 15min  
checkpoint_completion_target = 0.9  
wal_compression = on  
```

**Raisonnement :** Même configuration que OLTP pur (écritures OLTP ne doivent pas être pénalisées).

---

### 5. Autovacuum : Équilibre

**Le Dilemme :**
- OLTP a besoin d'autovacuum agressif (beaucoup UPDATE/DELETE)
- OLAP préfère autovacuum moins agressif (moins de concurrence)

**Configuration recommandée :**
```ini
autovacuum = on  
autovacuum_max_workers = 4-6  
autovacuum_naptime = 30s  # Compromis entre 10s (OLTP) et 5min (OLAP)  

# Seuils modérés
autovacuum_vacuum_threshold = 50  
autovacuum_vacuum_scale_factor = 0.1  # Compromis entre 0.05 (OLTP) et 0.2  
autovacuum_analyze_threshold = 50  
autovacuum_analyze_scale_factor = 0.05  

# Throttling modéré
autovacuum_vacuum_cost_delay = 2ms  # Compromis entre 0 (OLAP) et 2ms (OLTP)  
autovacuum_vacuum_cost_limit = 1000  # Compromis entre -1 (OLAP) et 2000 (OLTP)  
```

**Stratégie par table :**
```sql
-- Tables OLTP : autovacuum plus agressif
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.05
);

-- Tables OLAP/historiques : autovacuum moins agressif
ALTER TABLE fact_sales_history SET (
    autovacuum_vacuum_scale_factor = 0.2,
    autovacuum_analyze_scale_factor = 0.1
);
```

---

### 6. Planificateur : Optimiser pour les Deux

#### **random_page_cost** : Valeur SSD/NVMe

**Configuration recommandée :**
```ini
random_page_cost = 1.1  # SSD/NVMe  
seq_page_cost = 1.0  
```

---

#### **effective_io_concurrency** : Élevé pour SSD

**Configuration recommandée :**
```ini
effective_io_concurrency = 200-300
```

---

#### **default_statistics_target** : Modéré

**Configuration recommandée :**
```ini
default_statistics_target = 200  # Compromis entre 100 (OLTP) et 500 (OLAP)
```

**Ajustement par table :**
```sql
-- Tables OLAP critiques : statistiques détaillées
ALTER TABLE fact_sales ALTER COLUMN sale_date SET STATISTICS 500;

-- Tables OLTP : statistiques standard
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 100;
```

---

#### **JIT : Activer avec Seuil Élevé**

**Configuration recommandée :**
```ini
jit = on  
jit_above_cost = 200000  # Seulement pour requêtes OLAP longues  
jit_inline_above_cost = 500000  
jit_optimize_above_cost = 500000  
```

**Raisonnement :** JIT bénéficie surtout à OLAP, mais overhead pour OLTP. Seuil élevé évite impact sur OLTP.

---

### 7. I/O Asynchrone (PostgreSQL 18)

**Configuration recommandée :**
```ini
io_method = 'worker'  # Si kernel Linux 5.1+ avec io_uring
```

**Impact :** Bénéficie aux deux charges (OLTP et OLAP).

---

### 8. Timeouts : Protection contre Requêtes Longues

**Configuration recommandée :**
```ini
# Timeout global conservateur
statement_timeout = 0  # Pas de limite par défaut

# Timeout transaction idle (important !)
idle_in_transaction_session_timeout = 10min  # Éviter transactions zombies

# Lock timeout
lock_timeout = 30s  # Éviter deadlocks prolongés
```

**Ajustement par rôle :**
```sql
-- OLTP : timeouts stricts
ALTER ROLE app_oltp SET statement_timeout = '30s';  
ALTER ROLE app_oltp SET idle_in_transaction_session_timeout = '1min';  

-- OLAP : timeouts généreux
ALTER ROLE analytics_user SET statement_timeout = '30min';  
ALTER ROLE analytics_user SET idle_in_transaction_session_timeout = '1h';  
```

---

## Configuration Complète Mixed Workload

### Scénario : Serveur 64 GB RAM, 16 CPU, SSD NVMe, 250 connexions

```ini
# =====================================
# CONFIGURATION POSTGRESQL MIXED WORKLOAD
# Serveur : 64 GB RAM, 16 CPU, SSD NVMe
# =====================================

# ===================================
# MÉMOIRE (compromis OLTP/OLAP)
# ===================================
shared_buffers = 16GB                   # 25% de 64GB  
effective_cache_size = 48GB             # 75% de 64GB  
work_mem = 64MB                         # Compromis (ajustable par rôle)  
maintenance_work_mem = 2GB  
hash_mem_multiplier = 1.5               # Légèrement favoriser OLAP  

# ===================================
# CONNEXIONS (dimensionné OLTP)
# ===================================
max_connections = 250  
superuser_reserved_connections = 3  

# ===================================
# PARALLÉLISATION (modérée)
# ===================================
max_worker_processes = 16               # Tous les CPU  
max_parallel_workers_per_gather = 4     # Compromis entre 2 (OLTP) et 8 (OLAP)  
max_parallel_workers = 12               # 75% (laisser marge pour OLTP)  
max_parallel_maintenance_workers = 4  

# Seuils de parallélisation modérés
parallel_setup_cost = 500               # Entre 1000 (OLTP) et 100 (OLAP)  
parallel_tuple_cost = 0.05              # Entre 0.1 (OLTP) et 0.01 (OLAP)  
min_parallel_table_scan_size = 8MB  
min_parallel_index_scan_size = 512kB  

# ===================================
# WAL et CHECKPOINTS (optimisé OLTP)
# ===================================
wal_level = replica  
wal_buffers = 32MB  
max_wal_size = 4GB  
min_wal_size = 1GB  
checkpoint_timeout = 15min  
checkpoint_completion_target = 0.9  
wal_compression = on  

# Durabilité
synchronous_commit = on  
fsync = on  

# ===================================
# AUTOVACUUM (équilibré)
# ===================================
autovacuum = on  
autovacuum_max_workers = 5  
autovacuum_naptime = 30s                # Compromis 10s (OLTP) et 5min (OLAP)  

# Seuils modérés
autovacuum_vacuum_threshold = 50  
autovacuum_vacuum_scale_factor = 0.1  
autovacuum_analyze_threshold = 50  
autovacuum_analyze_scale_factor = 0.05  

# Throttling modéré
autovacuum_vacuum_cost_delay = 2ms  
autovacuum_vacuum_cost_limit = 1000  

# ===================================
# PLANIFICATEUR (SSD/NVMe)
# ===================================
random_page_cost = 1.1  
seq_page_cost = 1.0  
effective_io_concurrency = 300  

# Statistiques modérées
default_statistics_target = 200         # Compromis 100 (OLTP) et 500 (OLAP)

# JIT avec seuil élevé (OLAP seulement)
jit = on  
jit_above_cost = 200000  
jit_inline_above_cost = 500000  
jit_optimize_above_cost = 500000  

# PostgreSQL 18 : I/O asynchrone
io_method = 'worker'

# ===================================
# TIMEOUTS
# ===================================
statement_timeout = 0                   # Ajusté par rôle  
idle_in_transaction_session_timeout = 10min  
lock_timeout = 30s  

# ===================================
# LOGGING et MONITORING
# ===================================
logging_collector = on  
log_destination = 'stderr'  
log_directory = 'log'  
log_filename = 'postgresql-%Y-%m-%d.log'  
log_rotation_age = 1d  
log_rotation_size = 200MB  

log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '  
log_min_duration_statement = 1000       # Log requêtes > 1 seconde  
log_checkpoints = on  
log_connections = on  
log_disconnections = on  
log_lock_waits = on  
log_temp_files = 0                      # Log fichiers temporaires  
log_autovacuum_min_duration = 0  

# Monitoring
track_activities = on  
track_counts = on  
track_io_timing = on  
track_functions = pl  
compute_query_id = on  

# Extensions
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

## Stratégies de Séparation Logique

### 1. Rôles et Permissions Différenciées

**Créer des rôles séparés :**

```sql
-- =====================
-- RÔLE OLTP
-- =====================
CREATE ROLE app_oltp WITH LOGIN PASSWORD 'secure_password_oltp';

-- Configuration OLTP
ALTER ROLE app_oltp SET work_mem = '32MB';  
ALTER ROLE app_oltp SET statement_timeout = '30s';  
ALTER ROLE app_oltp SET idle_in_transaction_session_timeout = '1min';  
ALTER ROLE app_oltp SET max_parallel_workers_per_gather = 0;  -- Pas de parallélisme  

-- Permissions OLTP (lecture/écriture sur tables transactionnelles)
GRANT CONNECT ON DATABASE mydb TO app_oltp;  
GRANT USAGE ON SCHEMA public TO app_oltp;  
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_oltp;  
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_oltp;  

-- =====================
-- RÔLE OLAP
-- =====================
CREATE ROLE analytics_user WITH LOGIN PASSWORD 'secure_password_analytics';

-- Configuration OLAP
ALTER ROLE analytics_user SET work_mem = '512MB';  
ALTER ROLE analytics_user SET statement_timeout = '30min';  
ALTER ROLE analytics_user SET idle_in_transaction_session_timeout = '1h';  
ALTER ROLE analytics_user SET max_parallel_workers_per_gather = 8;  -- Parallélisme max  
ALTER ROLE analytics_user SET default_statistics_target = 500;  

-- Permissions OLAP (lecture seule)
GRANT CONNECT ON DATABASE mydb TO analytics_user;  
GRANT USAGE ON SCHEMA public TO analytics_user;  
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics_user;  
-- Pas de INSERT/UPDATE/DELETE pour analytics

-- =====================
-- RÔLE ETL (chargement de données)
-- =====================
CREATE ROLE etl_user WITH LOGIN PASSWORD 'secure_password_etl';

ALTER ROLE etl_user SET work_mem = '1GB';  
ALTER ROLE etl_user SET maintenance_work_mem = '4GB';  
ALTER ROLE etl_user SET max_parallel_maintenance_workers = 8;  

GRANT CONNECT ON DATABASE mydb TO etl_user;  
GRANT USAGE ON SCHEMA public TO etl_user;  
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO etl_user;  
```

---

### 2. Connection Pooling Différencié

**Architecture avec PgBouncer :**

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
mydb_oltp = host=localhost dbname=mydb  
mydb_olap = host=localhost dbname=mydb  

[pgbouncer]
listen_addr = *  
listen_port = 6432  
auth_type = scram-sha-256  

# Pool OLTP : Transaction pooling (beaucoup de connexions courtes)
pool_mode = transaction  
default_pool_size = 20  
max_client_conn = 1000  
reserve_pool_size = 10  

# Pool OLAP : Session pooling (peu de connexions longues)
# Configurer dans section [databases]
; mydb_olap = ... pool_mode=session pool_size=5
```

**Utilisation :**
```python
# Application OLTP
conn_oltp = psycopg.connect("host=pgbouncer port=6432 dbname=mydb_oltp user=app_oltp")

# Dashboards OLAP
conn_olap = psycopg.connect("host=pgbouncer port=6432 dbname=mydb_olap user=analytics_user")
```

---

### 3. Séparation par Base de Données

**Structure :**

```sql
-- Base transactionnelle
CREATE DATABASE myapp_oltp;

-- Base analytique (vues matérialisées, agrégations)
CREATE DATABASE myapp_analytics;
```

**ETL pour synchroniser :**

```sql
-- Dans myapp_analytics, créer FDW vers myapp_oltp
CREATE EXTENSION postgres_fdw;

CREATE SERVER oltp_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'localhost', dbname 'myapp_oltp', port '5432');

CREATE USER MAPPING FOR analytics_user
    SERVER oltp_server
    OPTIONS (user 'etl_user', password 'xxx');

-- Importer schéma
IMPORT FOREIGN SCHEMA public
    FROM SERVER oltp_server
    INTO public;

-- Créer vues matérialisées sur données importées
CREATE MATERIALIZED VIEW mv_sales_summary AS  
SELECT  
    DATE_TRUNC('day', created_at) AS day,
    product_id,
    SUM(amount) AS total_amount
FROM orders  
GROUP BY DATE_TRUNC('day', created_at), product_id;  

-- Rafraîchir périodiquement (pg_cron)
SELECT cron.schedule('refresh-sales-mv', '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_summary');
```

---

## Optimisations Mixtes : Le Meilleur des Deux Mondes

### 1. Tables : Partitionnement Intelligent

**Principe :** Partitionner tables historiques (OLAP) tout en gardant données récentes (OLTP) accessibles.

```sql
-- Table partitionnée par mois
CREATE TABLE orders (
    order_id BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status VARCHAR(20),
    amount NUMERIC(10,2)
) PARTITION BY RANGE (created_at);

-- Partition "hot" (OLTP) : mois en cours
CREATE TABLE orders_current PARTITION OF orders
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

-- Partitions "warm" (mixed) : 2-3 derniers mois
CREATE TABLE orders_2025_10 PARTITION OF orders
    FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');

CREATE TABLE orders_2025_09 PARTITION OF orders
    FOR VALUES FROM ('2025-09-01') TO ('2025-10-01');

-- Partitions "cold" (OLAP) : historique
CREATE TABLE orders_2025_08 PARTITION OF orders
    FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');
-- ... etc
```

**Index différenciés :**

```sql
-- Partition "hot" : index B-Tree (OLTP)
CREATE INDEX idx_orders_current_user ON orders_current(user_id);  
CREATE INDEX idx_orders_current_status ON orders_current(status);  

-- Partitions "cold" : index BRIN (OLAP)
CREATE INDEX idx_orders_2025_08_created_brin ON orders_2025_08
    USING BRIN (created_at) WITH (pages_per_range = 128);
```

**Avantage :**
- OLTP accède rapidement aux données récentes (partition petite, index B-Tree)
- OLAP scanne efficacement l'historique (BRIN compact)

---

### 2. Vues Matérialisées pour OLAP, Tables Réelles pour OLTP

**Architecture :**

```
┌────────────────────┐
│  Tables OLTP       │  ← INSERT/UPDATE/DELETE fréquents
│  (orders, users)   │
└──────────┬─────────┘
           │
           │ Rafraîchissement périodique
           v
┌────────────────────┐
│ Vues Matérialisées │  ← SELECT complexes (dashboards)
│ (mv_sales_summary) │
└────────────────────┘
```

**Exemple :**

```sql
-- Vue matérialisée rafraîchie toutes les heures
CREATE MATERIALIZED VIEW mv_daily_sales AS  
SELECT  
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS num_orders,
    SUM(amount) AS total_amount,
    AVG(amount) AS avg_amount
FROM orders  
WHERE created_at >= CURRENT_DATE - INTERVAL '90 days'  
GROUP BY DATE_TRUNC('day', created_at);  

CREATE UNIQUE INDEX ON mv_daily_sales(day);

-- Rafraîchir sans bloquer les lectures
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
```

**Dashboards interrogent la vue matérialisée (rapide) au lieu de la table OLTP (lent).**

---

### 3. Index Couvrants (Covering Index) pour OLAP

**Principe :** Index contenant toutes les colonnes nécessaires → pas besoin d'accéder à la table.

```sql
-- Requête OLAP fréquente
SELECT product_id, SUM(amount), COUNT(*)  
FROM orders  
WHERE created_at >= '2025-01-01' AND status = 'completed'  
GROUP BY product_id;  

-- Index couvrant (INCLUDE clause)
CREATE INDEX idx_orders_covering ON orders (status, created_at)
    INCLUDE (product_id, amount);

-- PostgreSQL peut répondre avec index-only scan (très rapide)
```

---

### 4. Colonnes Générées (PostgreSQL 12+) pour Dénormalisation Partielle

**Concept :** Stocker des calculs fréquents sans duplication manuelle.

```sql
-- Ajouter colonne générée pour année/mois (utilisé dans GROUP BY OLAP)
ALTER TABLE orders
    ADD COLUMN year_month INT GENERATED ALWAYS AS (
        EXTRACT(YEAR FROM created_at)::INT * 100 + EXTRACT(MONTH FROM created_at)::INT
    ) STORED;

-- Index sur colonne générée
CREATE INDEX idx_orders_year_month ON orders(year_month);

-- Requête OLAP simplifiée et rapide
SELECT year_month, SUM(amount)  
FROM orders  
GROUP BY year_month;  
```

---

### 5. Nouveauté PostgreSQL 18 : Colonnes Virtuelles pour Calculs

**Différence avec STORED :** Ne prend pas d'espace disque (calculé à la volée).

```sql
-- PostgreSQL 18
ALTER TABLE orders
    ADD COLUMN amount_with_tax NUMERIC
    GENERATED ALWAYS AS (amount * 1.20) VIRTUAL;

-- Requête utilise la colonne virtuelle
SELECT amount_with_tax FROM orders;
-- Calcul à la volée, pas de stockage supplémentaire
```

**Avantage :** Simplifie requêtes OLAP sans coût de stockage.

---

## Monitoring Spécifique Mixed Workload

### 1. Identifier les Types de Requêtes

```sql
-- Requêtes par application/rôle
SELECT
    usename,
    application_name,
    state,
    COUNT(*) AS num_queries,
    AVG(EXTRACT(EPOCH FROM (NOW() - query_start))) AS avg_duration_sec
FROM pg_stat_activity  
WHERE state = 'active'  
GROUP BY usename, application_name, state  
ORDER BY avg_duration_sec DESC;  
```

**Interprétation :**
- `app_oltp` avec `avg_duration_sec < 1s` → OK  
- `analytics_user` avec `avg_duration_sec 10-60s` → OK  
- `app_oltp` avec `avg_duration_sec > 5s` → ⚠️ Problème !

---

### 2. Contention Mémoire (work_mem Insuffisant)

```sql
-- Requêtes utilisant des fichiers temporaires
SELECT
    usename,
    query,
    temp_blks_written * 8192 / 1024 / 1024 AS temp_mb
FROM pg_stat_statements pss  
JOIN pg_user pu ON pss.userid = pu.usesysid  
WHERE temp_blks_written > 0  
ORDER BY temp_blks_written DESC  
LIMIT 20;  
```

**Action :**
- Si `usename = 'analytics_user'` → Augmenter `work_mem` pour ce rôle
- Si `usename = 'app_oltp'` → Requête OLTP anormale, optimiser !

---

### 3. Compétition CPU

```sql
-- Extension pg_stat_kcache nécessaire
SELECT
    usename,
    query,
    calls,
    total_exec_time,
    user_time,  -- Temps CPU utilisateur
    system_time  -- Temps CPU système
FROM pg_stat_statements pss  
JOIN pg_stat_kcache psk USING (queryid)  
JOIN pg_user pu ON pss.userid = pu.usesysid  
ORDER BY (user_time + system_time) DESC  
LIMIT 10;  
```

---

### 4. Impact Autovacuum sur OLTP

```sql
-- Vérifier si autovacuum bloque des requêtes OLTP
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity  
WHERE wait_event = 'VacuumDelay'  
   OR query ILIKE '%vacuum%';
```

**Si beaucoup de wait :** Réduire agressivité autovacuum ou augmenter `autovacuum_vacuum_cost_limit`.

---

### 5. Locks : OLAP Bloque OLTP ?

```sql
-- Trouver les locks qui bloquent
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks  
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid  
JOIN pg_catalog.pg_locks blocking_locks  
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
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid  
WHERE NOT blocked_locks.granted;  
```

**Action :** Si requête OLAP bloque OLTP, tuer la requête OLAP :
```sql
SELECT pg_terminate_backend(blocking_pid);
```

---

## Checklist Mixed Workload

### Configuration Générale
- [ ] **shared_buffers** = 25% RAM  
- [ ] **effective_cache_size** = 75% RAM  
- [ ] **work_mem** = 64MB (valeur par défaut modérée)  
- [ ] **maintenance_work_mem** = 2GB  
- [ ] **max_connections** = 200-300 (dimensionné OLTP)

### Parallélisation
- [ ] **max_parallel_workers_per_gather** = 4 (compromis)  
- [ ] **max_parallel_workers** = 75% des CPU (laisser marge OLTP)

### Rôles et Séparation
- [ ] Rôle OLTP créé avec `work_mem = 32MB`, `statement_timeout = 30s`  
- [ ] Rôle OLAP créé avec `work_mem = 512MB`, `statement_timeout = 30min`  
- [ ] Connection pooling différencié (PgBouncer ou similaire)

### Tables et Index
- [ ] Partitionnement sur tables mixtes (hot/warm/cold)  
- [ ] Index B-Tree sur partitions "hot" (OLTP)  
- [ ] Index BRIN sur partitions "cold" (OLAP)  
- [ ] Vues matérialisées pour agrégations fréquentes  
- [ ] Index couvrants pour requêtes OLAP récurrentes

### Autovacuum
- [ ] **autovacuum_naptime** = 30s (compromis)  
- [ ] **autovacuum_vacuum_scale_factor** = 0.1  
- [ ] Configuration par table pour OLTP vs OLAP

### Monitoring
- [ ] pg_stat_statements activé  
- [ ] Surveillance des requêtes par rôle/application  
- [ ] Surveillance fichiers temporaires (work_mem insuffisant ?)  
- [ ] Surveillance locks (OLAP bloque OLTP ?)  
- [ ] Surveillance CPU par type de charge

### Planificateur
- [ ] **random_page_cost** = 1.1 (SSD)  
- [ ] **default_statistics_target** = 200 (compromis)  
- [ ] Statistiques détaillées (500) sur colonnes OLAP critiques

---

## Erreurs Courantes à Éviter

### ❌ Erreur 1 : Configuration Extrême (100% OLTP ou 100% OLAP)
**Symptôme :** Une charge souffre au détriment de l'autre  
**Solution :** Trouver le compromis 70/30 et ajuster dynamiquement par rôle  

### ❌ Erreur 2 : Pas de Séparation des Rôles
**Symptôme :** Impossible de distinguer OLTP et OLAP dans le monitoring  
**Solution :** Créer rôles séparés avec configurations dédiées  

### ❌ Erreur 3 : Pas de Connection Pooling
**Symptôme :** Épuisement des connexions, OLTP ralenti  
**Solution :** PgBouncer avec pools séparés OLTP/OLAP  

### ❌ Erreur 4 : OLAP Bloque OLTP (Locks)
**Symptôme :** Transactions OLTP attendent des requêtes OLAP longues  
**Solution :**  
- Utiliser vues matérialisées (pas de locks)
- Isoler OLAP dans réplica read-only
- `lock_timeout` pour OLTP

### ❌ Erreur 5 : Pas de Partitionnement
**Symptôme :** Requêtes OLAP scannent des millions de lignes inutiles  
**Solution :** Partitionner par date (hot/warm/cold)  

### ❌ Erreur 6 : work_mem Global Trop Élevé
**Symptôme :** OOM sur charges OLTP (beaucoup de connexions)  
**Solution :** work_mem modéré global + ajustement par rôle OLAP  

### ❌ Erreur 7 : Pas de Monitoring Différencié
**Symptôme :** Impossible de diagnostiquer quel type de charge pose problème  
**Solution :** Utiliser `application_name` et séparer monitoring OLTP vs OLAP  

---

## Stratégies Avancées

### 1. Réplication pour Isolation Complète

**Architecture recommandée pour mixed workload important :**

```
┌─────────────────────┐
│  APP OLTP           │
└──────────┬──────────┘
           │
           v
┌─────────────────────────────┐   Streaming Replication  ┌───────────────────┐
│  PostgreSQL PRIMARY         │ ───────────────────────> │ REPLICA (OLAP)    │
│  - OLTP optimisé            │   (asynchrone)           │ - OLAP optimisé   │
│  - Écriture/Lecture         │                          │ - Lecture seule   │
│  - work_mem = 32MB          │                          │ - work_mem = 1GB  │
│  - max_connections = 300    │                          │ - Parallélisme max│
└─────────────────────────────┘                          └────────┬──────────┘
                                                                  │
                                                                  v
                                                         ┌─────────────────┐
                                                         │  DASHBOARDS     │
                                                         │  ANALYTICS      │
                                                         └─────────────────┘
```

**Avantages :**
- ✅ Isolation totale (OLAP n'impacte jamais OLTP)  
- ✅ Configuration optimale pour chaque charge  
- ✅ Scaling indépendant

**Configuration PRIMARY (postgresql.conf) :**
```ini
# PRIMARY : Optimisé OLTP
shared_buffers = 16GB  
work_mem = 32MB  
max_connections = 300  
max_parallel_workers_per_gather = 2  

# Réplication
wal_level = replica  
max_wal_senders = 3  
```

**Configuration REPLICA (postgresql.conf) :**
```ini
# REPLICA : Optimisé OLAP
shared_buffers = 32GB  
work_mem = 1GB  
max_connections = 50  
max_parallel_workers_per_gather = 8  
hot_standby = on  
hot_standby_feedback = on  # Important !  
```

**hot_standby_feedback :** Empêche PRIMARY de nettoyer trop tôt (OLAP lit anciennes versions).

---

### 2. Extension pg_cron pour Automatisation

```sql
CREATE EXTENSION pg_cron;

-- Rafraîchir vues matérialisées pendant heures creuses
SELECT cron.schedule(
    'refresh-mv-sales',
    '0 2 * * *',  -- 2h du matin
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_summary'
);

-- Analyser tables OLAP tous les jours
SELECT cron.schedule(
    'analyze-fact-tables',
    '0 3 * * *',
    'ANALYZE fact_sales'
);

-- Vacuum tables OLTP plus fréquemment
SELECT cron.schedule(
    'vacuum-oltp-tables',
    '0 */4 * * *',  -- Toutes les 4h
    'VACUUM (ANALYZE) orders'
);
```

---

### 3. Utiliser UNLOGGED Tables pour Données Temporaires

**Concept :** Tables non-WAL pour staging ETL (plus rapides).

```sql
-- Table staging pour chargement ETL
CREATE UNLOGGED TABLE staging_sales (
    LIKE fact_sales INCLUDING ALL
);

-- Charger rapidement (pas de WAL)
COPY staging_sales FROM '/data/sales.csv' WITH CSV;

-- Valider et copier vers table finale
INSERT INTO fact_sales SELECT * FROM staging_sales;

-- Nettoyer
TRUNCATE staging_sales;
```

**⚠️ Attention :** UNLOGGED = perte de données si crash. Acceptable pour staging.

---

## Cas Réels : Exemples de Configuration

### Cas 1 : Startup SaaS (80% OLTP, 20% OLAP)

**Contexte :**
- Application Web avec dashboard intégré
- 10k utilisateurs, 500k requêtes/jour
- Dashboards consultés 100× par jour

**Configuration :**
```ini
# Prioriser OLTP
shared_buffers = 16GB  
work_mem = 32MB  # Défaut bas (OLTP)  
max_connections = 300  
max_parallel_workers_per_gather = 2  

# Rôle OLAP avec ajustements
ALTER ROLE analytics_user SET work_mem = '256MB';  
ALTER ROLE analytics_user SET max_parallel_workers_per_gather = 6;  

# Vues matérialisées pour dashboards (rafraîchir toutes les heures)
```

---

### Cas 2 : E-commerce avec BI (60% OLTP, 40% OLAP)

**Contexte :**
- Site e-commerce avec backoffice analytics
- Analyste BI fait des requêtes ad-hoc
- Rapports quotidiens automatisés

**Configuration :**
```ini
# Équilibre
shared_buffers = 24GB  
work_mem = 64MB  # Compromis  
max_connections = 200  
max_parallel_workers_per_gather = 4  

# Rôle OLAP avec beaucoup de ressources
ALTER ROLE data_analyst SET work_mem = '1GB';  
ALTER ROLE data_analyst SET max_parallel_workers_per_gather = 8;  

# Partitionnement + vues matérialisées
# Séparation logique : schema 'analytics' pour OLAP
```

---

### Cas 3 : Application Legacy (50/50 OLTP/OLAP)

**Contexte :**
- Application monolithique, impossible de séparer
- Mix constant de transactions et analyses
- Budget limité (une seule instance)

**Configuration :**
```ini
# Compromis fort
shared_buffers = 20GB  
work_mem = 64MB  
max_connections = 250  
max_parallel_workers_per_gather = 4  

# Timeouts pour protéger OLTP
statement_timeout = 60s  # Global, requêtes > 1min tuées  
idle_in_transaction_session_timeout = 5min  

# Connection pooling obligatoire (PgBouncer)
# Surveillance intensive (pg_stat_statements)
```

---

## Conclusion

Le **Mixed Workload** est le scénario le plus courant et le plus complexe. Il nécessite :

### Principes Clés

1. **Compromis Intelligent** : Configuration par défaut modérée (70% OLTP, 30% OLAP)  
2. **Ajustements Dynamiques** : Utiliser rôles, sessions, application_name  
3. **Séparation Logique** : Rôles, bases, schémas, connection pooling  
4. **Monitoring Différencié** : Distinguer OLTP et OLAP dans les métriques  
5. **Architecture Progressive** : Commencer simple, migrer vers réplication si besoin

### Checklist Finale

**Configuration de Base :**
- [ ] Paramètres globaux = compromis OLTP/OLAP  
- [ ] Rôles séparés avec configurations dédiées  
- [ ] Connection pooling différencié

**Optimisations :**
- [ ] Partitionnement (hot/warm/cold)  
- [ ] Index mixtes (B-Tree + BRIN selon partition)  
- [ ] Vues matérialisées pour OLAP fréquent  
- [ ] Colonnes générées pour simplifier requêtes

**Monitoring :**
- [ ] pg_stat_statements par rôle  
- [ ] Fichiers temporaires (work_mem insuffisant ?)  
- [ ] Locks (OLAP bloque OLTP ?)  
- [ ] CPU et I/O par type de charge

**Évolution :**
- [ ] Si OLAP > 30% → Envisager réplication  
- [ ] Si locks fréquents → Vues matérialisées ou réplication  
- [ ] Si OOM → Réduire work_mem global, augmenter par rôle OLAP

### Métriques de Succès

**OLTP :**
- Latence P99 < 100ms
- Pas de lock_timeout dépassés
- Cache hit ratio > 99%

**OLAP :**
- Requêtes complexes < 30s (objectif raisonnable)
- Pas de fichiers temporaires excessifs
- Parallélisation active (EXPLAIN montre workers)

### Ressources

- **PgBouncer** : https://www.pgbouncer.org/  
- **pg_stat_statements** : Extension essentielle  
- **pg_cron** : Automatisation (vues matérialisées, vacuum)  
- **Patroni** : HA avec réplication automatique

Le Mixed Workload demande de la **finesse** et du **monitoring constant**. Commencez par une configuration équilibrée, mesurez, ajustez, et évoluez vers une architecture séparée si les besoins le justifient.

Bonne chance ! 🎯

⏭️ [Développement local](/annexes/configuration-reference/04-configuration-developpement-local.md)
