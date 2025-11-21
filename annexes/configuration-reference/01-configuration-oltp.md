🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration PostgreSQL pour OLTP (High Concurrency, Low Latency)

## Introduction : Qu'est-ce que l'OLTP ?

### Définition

**OLTP** signifie **Online Transaction Processing** (Traitement Transactionnel en Ligne). C'est un type de charge de travail caractérisé par :

- **Nombreuses transactions courtes** : Des milliers voire des millions de petites requêtes par seconde
- **Opérations de lecture ET d'écriture fréquentes** : INSERT, UPDATE, DELETE en temps réel
- **Faible latence requise** : Les utilisateurs attendent des réponses quasi-instantanées (< 100ms)
- **Haute concurrence** : Beaucoup d'utilisateurs/applications accèdent simultanément à la base

### Exemples d'Applications OLTP

- **E-commerce** : Ajout au panier, commandes, paiements
- **Applications bancaires** : Transactions, virements, consultations de solde
- **Réseaux sociaux** : Publications, likes, commentaires
- **Systèmes de réservation** : Réservations d'hôtels, vols, restaurants
- **Applications SaaS** : CRM, ERP, outils collaboratifs

### OLTP vs OLAP : Quelle Différence ?

| Critère | OLTP | OLAP |
|---------|------|------|
| **Type d'opérations** | Transactions courtes (lecture/écriture) | Requêtes analytiques longues (lecture) |
| **Volume par requête** | Peu de lignes (1-1000) | Millions de lignes |
| **Fréquence** | Très élevée (milliers/seconde) | Faible (quelques requêtes/heure) |
| **Latence attendue** | < 100ms | Secondes à minutes |
| **Utilisateurs** | Milliers d'utilisateurs concurrents | Quelques analystes |
| **Données** | Actuelles, opérationnelles | Historiques, agrégées |

> **Pour les débutants** : Imaginez OLTP comme une caisse de supermarché (beaucoup de clients, transactions rapides) et OLAP comme un bureau d'analyse marketing (peu de personnes, calculs complexes sur beaucoup de données).

---

## Objectifs de Configuration pour OLTP

Pour une charge de travail OLTP, nous cherchons à optimiser PostgreSQL selon ces principes :

1. **Maximiser le débit de transactions** (transactions/seconde)
2. **Minimiser la latence** (temps de réponse)
3. **Gérer efficacement la concurrence** (beaucoup de connexions simultanées)
4. **Optimiser les opérations en mémoire** (éviter les accès disque lents)
5. **Équilibrer les écritures** (WAL, checkpoints) pour éviter les pics de latence

---

## Paramètres Clés pour OLTP

### 1. Gestion de la Mémoire

#### **shared_buffers** : Le Cache Principal

**Qu'est-ce que c'est ?**
C'est la mémoire partagée où PostgreSQL stocke les pages de données les plus utilisées. Plus cette mémoire est grande, moins PostgreSQL a besoin d'aller lire sur le disque (qui est beaucoup plus lent).

**Configuration recommandée pour OLTP :**
```ini
shared_buffers = 25% de la RAM totale
```

**Exemples concrets :**
- Serveur avec 8 GB RAM → `shared_buffers = 2GB`
- Serveur avec 32 GB RAM → `shared_buffers = 8GB`
- Serveur avec 64 GB RAM → `shared_buffers = 16GB`

**Pourquoi pas plus de 25% ?**
Au-delà de 25-30%, le système d'exploitation (Linux/Windows) gère mieux le cache lui-même. PostgreSQL délègue donc une partie du caching à l'OS.

**Configuration dans postgresql.conf :**
```ini
shared_buffers = 8GB
```

---

#### **effective_cache_size** : Indiquer au Planificateur la Mémoire Disponible

**Qu'est-ce que c'est ?**
Ce paramètre n'alloue **pas** de mémoire. Il indique au planificateur de requêtes combien de mémoire totale (PostgreSQL + OS) est disponible pour le cache. Cela l'aide à choisir les meilleurs plans d'exécution.

**Configuration recommandée pour OLTP :**
```ini
effective_cache_size = 50-75% de la RAM totale
```

**Exemples concrets :**
- Serveur avec 32 GB RAM → `effective_cache_size = 24GB` (75%)
- Serveur avec 64 GB RAM → `effective_cache_size = 48GB` (75%)

**Configuration dans postgresql.conf :**
```ini
effective_cache_size = 24GB
```

---

#### **work_mem** : Mémoire par Opération de Tri/Hachage

**Qu'est-ce que c'est ?**
Mémoire allouée pour des opérations comme le tri (ORDER BY), le hachage (JOIN, GROUP BY), ou les bitmaps d'index. Si la mémoire est insuffisante, PostgreSQL écrit temporairement sur disque (très lent).

**⚠️ Attention : Multiplicateur Dangereux**
Chaque connexion peut utiliser `work_mem` **plusieurs fois** (pour plusieurs opérations dans une même requête). Avec 100 connexions utilisant chacune 3 opérations nécessitant du tri :
```
Consommation totale = 100 connexions × 3 opérations × work_mem
```

**Configuration recommandée pour OLTP :**
```ini
work_mem = 4MB à 32MB (selon RAM et nombre de connexions)
```

**Calcul pratique :**
```
work_mem maximum sécurisé = (RAM totale × 25%) / max_connections / 3
```

**Exemples :**
- Serveur 32 GB, 200 connexions → `work_mem = 16MB`
- Serveur 64 GB, 300 connexions → `work_mem = 24MB`

**Configuration dans postgresql.conf :**
```ini
work_mem = 16MB
```

**💡 Astuce pour débutants** : Commencez avec une valeur conservatrice (8-16MB). Si vous voyez des "temp files" dans les logs (disque utilisé pour trier), augmentez progressivement.

---

#### **maintenance_work_mem** : Mémoire pour les Opérations de Maintenance

**Qu'est-ce que c'est ?**
Mémoire utilisée pour les opérations de maintenance : VACUUM, CREATE INDEX, ALTER TABLE, etc.

**Configuration recommandée pour OLTP :**
```ini
maintenance_work_mem = 1-2GB (maximum recommandé : 2GB)
```

**Pourquoi important pour OLTP ?**
- Les index sont cruciaux en OLTP (beaucoup de lookups rapides)
- VACUUM doit tourner fréquemment (beaucoup de UPDATE/DELETE)
- Plus de mémoire = maintenance plus rapide = moins d'impact sur les performances

**Configuration dans postgresql.conf :**
```ini
maintenance_work_mem = 1GB
```

---

### 2. Gestion des Connexions

#### **max_connections** : Nombre Maximum de Connexions

**Qu'est-ce que c'est ?**
Le nombre maximum de connexions simultanées que PostgreSQL accepte.

**⚠️ Problème OLTP typique : Trop de connexions**
En OLTP, beaucoup d'applications Web créent une connexion par requête. Avec 100 utilisateurs simultanés et un pool de 50 connexions par serveur Web, vous pouvez rapidement atteindre des milliers de connexions.

**Configuration recommandée pour OLTP :**
```ini
max_connections = 200-500 (avec connection pooling)
max_connections = 50-100 (sans connection pooling)
```

**💡 Solution : Connection Pooling**
Au lieu de permettre 5000 connexions directes, utilisez un **pooler** (PgBouncer, PgPool) qui multiplex les connexions :

```
5000 clients → PgBouncer (mode transaction) → 200 connexions PostgreSQL
```

**Impact sur la mémoire :**
Chaque connexion consomme ~10MB de RAM (processus backend). Donc :
- 500 connexions = ~5GB RAM rien que pour les processus
- 5000 connexions = ~50GB RAM (impossible sur serveur standard)

**Configuration dans postgresql.conf :**
```ini
max_connections = 300
```

---

### 3. Gestion des Écritures et du WAL

#### **wal_buffers** : Buffer du Write-Ahead Log

**Qu'est-ce que c'est ?**
Mémoire tampon pour les écritures dans le WAL (Write-Ahead Log). Le WAL enregistre toutes les modifications avant qu'elles ne soient appliquées aux fichiers de données (principe de durabilité ACID).

**Configuration recommandée pour OLTP :**
```ini
wal_buffers = 16MB à 64MB
```

**Valeur par défaut intelligente :**
Depuis PostgreSQL 9.1, si vous ne configurez pas ce paramètre, PostgreSQL calcule automatiquement :
```
wal_buffers = 3% de shared_buffers (jusqu'à 16MB par défaut)
```

Pour OLTP avec beaucoup d'écritures, augmentez manuellement :

**Configuration dans postgresql.conf :**
```ini
wal_buffers = 32MB
```

---

#### **checkpoint_timeout** et **max_wal_size** : Gestion des Checkpoints

**Qu'est-ce qu'un checkpoint ?**
Un checkpoint est le moment où PostgreSQL écrit toutes les données modifiées en mémoire (dirty pages) sur le disque. C'est une opération **lourde** qui peut créer des pics de latence.

**Problème OLTP :**
Des checkpoints trop fréquents = pics de latence réguliers = mauvaise expérience utilisateur.

**Configuration recommandée pour OLTP :**
```ini
checkpoint_timeout = 15min (défaut : 5min)
max_wal_size = 4-8GB (défaut : 1GB)
```

**Pourquoi ces valeurs ?**
- Checkpoints moins fréquents = écritures plus étalées dans le temps
- Plus de WAL disponible = système plus souple lors de pics d'écriture

**⚠️ Contre-indication :**
Plus de WAL à rejouer en cas de crash = temps de récupération plus long (acceptable en OLTP où les pannes sont rares).

**Configuration dans postgresql.conf :**
```ini
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
```

---

#### **checkpoint_completion_target** : Étaler les Écritures

**Qu'est-ce que c'est ?**
Pourcentage de `checkpoint_timeout` pendant lequel PostgreSQL étale les écritures du checkpoint.

**Valeur par défaut :** 0.5 (écrit pendant 50% de l'intervalle)
**Recommandation OLTP :** 0.9 (écrit pendant 90% de l'intervalle)

**Pourquoi 0.9 ?**
Étaler les écritures sur une plus longue période réduit les pics d'I/O et donc les pics de latence.

**Configuration dans postgresql.conf :**
```ini
checkpoint_completion_target = 0.9
```

---

#### **synchronous_commit** : Compromise Durabilité/Performance

**Qu'est-ce que c'est ?**
Contrôle quand PostgreSQL considère une transaction comme "validée" (committed).

**Valeurs possibles :**

| Valeur | Comportement | Performance | Risque de perte |
|--------|--------------|-------------|-----------------|
| **on** (défaut) | Attend que le WAL soit écrit sur disque | Normale | Aucune |
| **remote_write** | Attend que le WAL soit envoyé au replica | Moyenne | Aucune (avec réplication) |
| **local** | Attend que le WAL soit écrit localement | Rapide | Faible (perte si crash OS) |
| **off** | Ne attend pas d'écriture disque | Très rapide | Possible (perte si crash) |

**Configuration recommandée pour OLTP :**

**Pour des données critiques (bancaire, paiements) :**
```ini
synchronous_commit = on
```

**Pour des données non critiques (logs, analytics, cache) :**
```ini
synchronous_commit = off
```

**💡 Astuce :** Vous pouvez configurer cela par session ou par transaction :
```sql
-- Pour une transaction spécifique
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO logs VALUES (...);
COMMIT;
```

---

### 4. Autovacuum : Essentiel pour OLTP

#### **Pourquoi Autovacuum est Critique en OLTP ?**

En OLTP, vous avez beaucoup de UPDATE et DELETE. Grâce au MVCC (Multiversion Concurrency Control), PostgreSQL ne supprime pas immédiatement les anciennes versions des lignes. Elles deviennent des "tuples morts" (dead tuples).

**Problèmes si Autovacuum est mal configuré :**
- Tables gonflent (bloat) → performances dégradées
- Index gonflent → scans plus lents
- Statistiques obsolètes → mauvais plans de requête

**Configuration recommandée pour OLTP :**

```ini
# Rendre autovacuum plus agressif
autovacuum = on (toujours activé !)
autovacuum_max_workers = 4 à 8 (selon nombre de CPU)
autovacuum_naptime = 10s (défaut : 1min)

# Seuils plus bas pour déclencher autovacuum plus tôt
autovacuum_vacuum_threshold = 25 (défaut : 50)
autovacuum_vacuum_scale_factor = 0.05 (défaut : 0.2)

# Accélérer les vacuum
autovacuum_vacuum_cost_delay = 2ms (défaut : 2ms, descendre à 0 si I/O disponible)
autovacuum_vacuum_cost_limit = 2000 (défaut : 200, augmenter si I/O disponible)
```

**Explication pour débutants :**

- `autovacuum_naptime` : Fréquence de réveil d'autovacuum (vérifier si des tables ont besoin de nettoyage)
- `autovacuum_vacuum_threshold` : Nombre minimum de tuples morts avant de déclencher vacuum
- `autovacuum_vacuum_scale_factor` : Pourcentage de la table en tuples morts pour déclencher vacuum

**Exemple de calcul :**
Table de 1 million de lignes avec `scale_factor = 0.05` :
```
Déclenchement quand : 25 + (1,000,000 × 0.05) = 50,025 tuples morts
```

Avec `scale_factor = 0.2` (défaut), autovacuum se déclencherait à 200,050 tuples morts → trop tard pour OLTP !

---

### 5. Paramètres d'Optimisation des Requêtes

#### **random_page_cost** : Coût des Accès Disque Aléatoires

**Qu'est-ce que c'est ?**
Estimation du coût relatif d'un accès disque aléatoire par rapport à un accès séquentiel.

**Valeur par défaut :** 4.0 (disques mécaniques)
**Recommandation OLTP (SSD/NVMe) :** 1.1 à 1.5

**Pourquoi important ?**
Avec des SSD, les accès aléatoires sont beaucoup plus rapides qu'avec des disques mécaniques. Si vous gardez 4.0, PostgreSQL sous-estime l'intérêt des index et fait trop de scans séquentiels.

**Configuration dans postgresql.conf :**
```ini
random_page_cost = 1.1  # SSD/NVMe
```

---

#### **effective_io_concurrency** : Opérations I/O Parallèles

**Qu'est-ce que c'est ?**
Nombre d'opérations I/O simultanées que le stockage peut gérer efficacement.

**Valeur par défaut :** 1 (disque mécanique)
**Recommandation OLTP :**
- SSD : 200-300
- NVMe : 300-400

**Configuration dans postgresql.conf :**
```ini
effective_io_concurrency = 200  # SSD
```

---

#### **Nouveauté PostgreSQL 18 : I/O Asynchrone**

PostgreSQL 18 introduit un nouveau sous-système I/O asynchrone qui peut améliorer les performances jusqu'à 3× pour certaines charges OLTP.

**Configuration dans postgresql.conf :**
```ini
io_method = 'async'  # Défaut : 'sync'
```

**⚠️ Attention :** Nécessite un noyau Linux récent (5.1+) avec support io_uring.

---

### 6. Statistiques et Monitoring

#### **track_activities et track_counts** : Monitoring Essentiel

**Configuration dans postgresql.conf :**
```ini
track_activities = on
track_counts = on
track_io_timing = on  # Mesure le temps I/O (léger overhead)
```

**Pourquoi important en OLTP ?**
Vous devez pouvoir identifier rapidement :
- Les requêtes lentes
- Les locks qui bloquent
- Les tables qui ont besoin de vacuum

---

#### **log_min_duration_statement** : Tracer les Requêtes Lentes

**Configuration dans postgresql.conf :**
```ini
log_min_duration_statement = 100ms  # Log toute requête > 100ms
```

**Stratégie OLTP :**
- Production : 50-100ms (tracer les requêtes anormalement lentes)
- Développement : 10ms (optimiser agressivement)

---

### 7. Paramètres de Planification de Requêtes

```ini
# Augmenter les ressources allouées au planificateur
default_statistics_target = 100  # Défaut : 100, augmenter à 200-500 pour tables critiques
```

---

## Configuration Complète pour OLTP

Voici une configuration **complète** pour un serveur PostgreSQL 18 dédié à OLTP :

### Scénario : Serveur 32 GB RAM, 8 CPU, SSD NVMe, 300 connexions max

```ini
# ===================================
# MÉMOIRE
# ===================================
shared_buffers = 8GB                    # 25% de 32GB
effective_cache_size = 24GB             # 75% de 32GB
work_mem = 16MB                         # (32GB × 0.25) / 300 / 3
maintenance_work_mem = 1GB
wal_buffers = 32MB

# ===================================
# CONNEXIONS
# ===================================
max_connections = 300
superuser_reserved_connections = 3

# ===================================
# WAL et CHECKPOINTS
# ===================================
wal_level = replica                     # Pour réplication
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
wal_compression = on                    # Compresser le WAL (économise disque)

# Durabilité (ajuster selon criticité des données)
synchronous_commit = on                 # 'off' si données non critiques
fsync = on                              # Ne JAMAIS désactiver en production !

# ===================================
# AUTOVACUUM (agressif pour OLTP)
# ===================================
autovacuum = on
autovacuum_max_workers = 6
autovacuum_naptime = 10s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_threshold = 25
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000

# ===================================
# PLANIFICATEUR (optimisé SSD/NVMe)
# ===================================
random_page_cost = 1.1                  # SSD/NVMe
effective_io_concurrency = 300          # NVMe
default_statistics_target = 100

# PostgreSQL 18 : I/O asynchrone
io_method = 'async'

# ===================================
# PARALLÉLISATION (si applicable)
# ===================================
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4

# ===================================
# LOGGING et MONITORING
# ===================================
logging_collector = on
log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB

log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_min_duration_statement = 100ms      # Log requêtes > 100ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on                     # Log attentes de verrous
log_temp_files = 0                      # Log tous les fichiers temporaires

track_activities = on
track_counts = on
track_io_timing = on
track_functions = pl                    # Tracker fonctions PL/pgSQL

# ===================================
# SÉCURITÉ
# ===================================
ssl = on
ssl_prefer_server_ciphers = on
password_encryption = scram-sha-256     # SHA-256 (pas MD5 !)

# ===================================
# TIMEZONE et LOCALE
# ===================================
timezone = 'UTC'                        # TOUJOURS UTC en production !
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

---

## Optimisations Supplémentaires pour OLTP

### 1. Connection Pooling avec PgBouncer

**Problème :** 300 connexions PostgreSQL = ~3GB RAM + overhead CPU

**Solution :** PgBouncer en mode **transaction pooling**

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost dbname=mydb

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = scram-sha-256
pool_mode = transaction              # Crucial pour OLTP
max_client_conn = 10000              # Clients peuvent se connecter
default_pool_size = 100              # Seulement 100 connexions vers PostgreSQL
reserve_pool_size = 50
reserve_pool_timeout = 5
```

**Résultat :**
- 10,000 clients → PgBouncer → 100 connexions PostgreSQL
- RAM économisée : (300 - 100) × 10MB = 2GB

---

### 2. Utiliser les Index Efficacement

**Principes pour OLTP :**

#### Index B-Tree sur colonnes de filtrage fréquent
```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

#### Index composites pour requêtes multi-colonnes
```sql
-- Pour : WHERE status = 'pending' AND created_at > NOW() - INTERVAL '1 day'
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
```

#### Index partiels pour sous-ensembles fréquents
```sql
-- Seulement indexer les commandes actives (90% des requêtes)
CREATE INDEX idx_orders_active ON orders(user_id)
WHERE status IN ('pending', 'processing');
```

#### Nouveauté PostgreSQL 18 : Skip Scan Optimization
PostgreSQL 18 peut maintenant utiliser efficacement un index multi-colonnes même si la première colonne n'est pas dans le WHERE :

```sql
-- Index composite
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Requête sans 'status' (1ère colonne)
SELECT * FROM orders WHERE created_at > '2025-01-01';
-- PostgreSQL 18 peut utiliser l'index avec Skip Scan !
```

---

### 3. Prepared Statements

**Principe :** Pré-compiler les requêtes fréquentes pour économiser le temps de parsing et planning.

**Exemple en Python (psycopg3) :**
```python
# Mauvais : Planning à chaque exécution
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))

# Bon : Préparer une fois, exécuter plusieurs fois
cursor.execute("PREPARE get_user AS SELECT * FROM users WHERE email = $1")
cursor.execute("EXECUTE get_user(%s)", (email,))
```

**Gain typique :** 10-30% de réduction de latence sur requêtes simples fréquentes.

---

### 4. Partitionnement pour Tables Volumineuses

**Quand partitionner en OLTP ?**
- Tables > 100 millions de lignes
- Requêtes accèdent toujours à un sous-ensemble (par date, par région, etc.)

**Exemple : Partitionnement par mois**
```sql
-- Table parent
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    total NUMERIC(10,2)
) PARTITION BY RANGE (created_at);

-- Partitions
CREATE TABLE orders_2025_01 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**Avantage :** PostgreSQL accède seulement à la partition pertinente (Partition Pruning), ignorant les autres.

---

## Monitoring : Vérifier que Tout Fonctionne

### 1. Cache Hit Ratio (Objectif : > 99%)

```sql
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 AS cache_hit_ratio
FROM pg_statio_user_tables;
```

**Interprétation :**
- > 99% : Excellent (données en RAM)
- 90-99% : Acceptable
- < 90% : Problème ! Augmenter `shared_buffers` ou investiguer requêtes

---

### 2. Connexions Actives

```sql
SELECT
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle') AS idle,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction
FROM pg_stat_activity;
```

**⚠️ Alerte :** Si `idle_in_transaction` > 10, vous avez des transactions ouvertes qui bloquent ! Vérifier le code applicatif.

---

### 3. Requêtes Lentes (avec pg_stat_statements)

```sql
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

**Action :** Optimiser les requêtes avec `mean_exec_time` > 100ms.

---

### 4. Tables Nécessitant Vacuum

```sql
SELECT
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;
```

**⚠️ Alerte :** Si `dead_ratio` > 10%, autovacuum ne suit pas ! Augmenter agressivité ou lancer VACUUM manuel.

---

## Checklist de Vérification OLTP

Avant de mettre en production, vérifiez :

- [ ] **Mémoire** : `shared_buffers` = 25% RAM
- [ ] **Mémoire** : `effective_cache_size` = 75% RAM
- [ ] **Mémoire** : `work_mem` dimensionné selon max_connections
- [ ] **Connexions** : Connection pooling configuré (PgBouncer)
- [ ] **WAL** : `max_wal_size` ≥ 4GB
- [ ] **Checkpoints** : `checkpoint_timeout` = 15min
- [ ] **Checkpoints** : `checkpoint_completion_target` = 0.9
- [ ] **Autovacuum** : Activé et configuré agressivement
- [ ] **Autovacuum** : `autovacuum_naptime` = 10s
- [ ] **Autovacuum** : `scale_factor` = 0.05
- [ ] **Disques** : `random_page_cost` = 1.1 (SSD/NVMe)
- [ ] **I/O** : `effective_io_concurrency` = 200-300
- [ ] **PostgreSQL 18** : `io_method` = 'async' (si kernel compatible)
- [ ] **Monitoring** : `log_min_duration_statement` configuré
- [ ] **Monitoring** : pg_stat_statements activé
- [ ] **Sécurité** : `password_encryption` = scram-sha-256
- [ ] **Sécurité** : SSL activé
- [ ] **Index** : Index B-Tree sur toutes les colonnes de WHERE/JOIN
- [ ] **Index** : Index partiels pour filtres fréquents
- [ ] **Sauvegarde** : WAL archiving configuré
- [ ] **Sauvegarde** : pg_basebackup testé et automatisé

---

## Erreurs Courantes à Éviter

### ❌ Erreur 1 : Trop de Connexions sans Pooling
**Symptôme :** RAM saturée, performances dégradées
**Solution :** PgBouncer en mode transaction

### ❌ Erreur 2 : work_mem trop Élevé
**Symptôme :** OOM (Out of Memory) kills
**Solution :** Dimensionner selon formule : `(RAM × 0.25) / max_connections / 3`

### ❌ Erreur 3 : Autovacuum Désactivé ou Trop Lent
**Symptôme :** Tables gonflent, performances se dégradent avec le temps
**Solution :** Activer et configurer agressivement

### ❌ Erreur 4 : synchronous_commit = off sur Données Critiques
**Symptôme :** Pertes de transactions récentes après crash
**Solution :** `synchronous_commit = on` pour données critiques

### ❌ Erreur 5 : random_page_cost = 4.0 avec SSD
**Symptôme :** Planificateur évite les index, fait trop de scans séquentiels
**Solution :** `random_page_cost = 1.1`

### ❌ Erreur 6 : Pas de Monitoring
**Symptôme :** Problèmes découverts trop tard
**Solution :** pg_stat_statements + logs + métriques système

---

## Ressources pour Aller Plus Loin

### Documentation Officielle
- [PostgreSQL 18 Documentation - Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html)
- [PostgreSQL Wiki - Tuning Your PostgreSQL Server](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)

### Outils Automatiques de Configuration
- **PGTune** : https://pgtune.leopard.in.ua/
  - Génère une config optimisée selon votre matériel
  - Choisir "Web application" ou "OLTP" pour cas d'usage

### Blogs et Guides
- **Percona Blog** : Articles avancés sur performance PostgreSQL
- **CrunchyData Blog** : Guides pratiques d'administration
- **2ndQuadrant Blog** : Experts PostgreSQL (core developers)

### Extensions Utiles
- **pg_stat_statements** : Analyse de requêtes (essentiel)
- **pg_stat_kcache** : Métriques CPU et I/O par requête
- **pgBadger** : Analyse de logs (graphiques)
- **HypoPG** : Tester des index hypothétiques sans les créer

---

## Conclusion

La configuration OLTP de PostgreSQL vise à :
1. **Maximiser les données en mémoire** (shared_buffers, effective_cache_size)
2. **Gérer efficacement la concurrence** (work_mem, max_connections, pooling)
3. **Optimiser les écritures** (WAL, checkpoints étalés)
4. **Maintenir la base propre** (autovacuum agressif)
5. **Utiliser le stockage moderne efficacement** (random_page_cost, I/O async)

**Points clés à retenir :**
- Commencez avec des valeurs conservatrices
- Monitorez constamment (cache hit ratio, requêtes lentes, autovacuum)
- Ajustez progressivement selon les métriques
- **Connection pooling n'est pas optionnel en OLTP** (PgBouncer)
- Testez en charge réelle avant production

PostgreSQL 18 apporte des améliorations significatives pour OLTP avec le sous-système I/O asynchrone et les optimisations du planificateur. Profitez-en !

---

**Prochaines Étapes :**
1. Appliquer cette configuration sur un environnement de staging
2. Lancer des tests de charge (pgbench, JMeter)
3. Ajuster selon les résultats
4. Activer pg_stat_statements et analyser les requêtes
5. Mettre en place un monitoring continu (Prometheus/Grafana)

Bonne chance avec votre base PostgreSQL OLTP ! 🚀

⏭️ [OLAP (Data warehouse, analytics)](/annexes/configuration-reference/02-configuration-olap.md)
