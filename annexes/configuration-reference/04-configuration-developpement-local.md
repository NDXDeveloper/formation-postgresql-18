🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration PostgreSQL pour Développement Local

## Introduction : PostgreSQL en Développement Local

### Qu'est-ce que le Développement Local ?

Le **développement local** désigne l'environnement de travail d'un développeur sur son propre ordinateur (laptop ou desktop) pour :

- Développer et tester du code
- Expérimenter avec des requêtes SQL
- Déboguer des problèmes
- Tester des migrations de schéma
- Apprendre PostgreSQL

### Différences avec la Production

| Aspect | Production | Développement Local |
|--------|------------|---------------------|
| **Matériel** | Serveur dédié (32-256 GB RAM) | Laptop (8-32 GB RAM) |
| **Performance** | Critique (milliers d'utilisateurs) | Confort (un développeur) |
| **Sécurité** | Maximale | Minimale (pas de données sensibles) |
| **Logs** | Essentiels, ciblés | Très détaillés (déboguer) |
| **Durabilité** | Totale (fsync, WAL) | Optionnelle (accepter perte) |
| **Simplicité** | Configuration complexe | Configuration simple |
| **Coût panne** | Perte financière | Perte de temps dev |

> **Pour les débutants** : En développement local, on privilégie la **simplicité**, les **logs détaillés** pour apprendre, et la **vitesse** plutôt que la robustesse.

---

## Objectifs de Configuration pour Développement Local

### Priorités

1. **Simplicité** : Configuration minimale, facile à comprendre
2. **Feedback rapide** : Voir immédiatement le résultat de ses actions
3. **Logs détaillés** : Comprendre ce que PostgreSQL fait
4. **Performance raisonnable** : Pas critique mais agréable
5. **Facilité de reset** : Pouvoir repartir de zéro facilement

### Non-Priorités

- ❌ Haute disponibilité
- ❌ Sécurité maximale
- ❌ Durabilité absolue
- ❌ Optimisation extrême

---

## Scénarios d'Installation

### Option 1 : Installation Système (Recommandée pour Débuter)

**Pour :** Linux, macOS, Windows

**Avantages :**
- ✅ Simple à installer
- ✅ Démarre automatiquement
- ✅ Intégration avec l'OS

**Inconvénients :**
- ❌ Une seule version à la fois
- ❌ Plus difficile à nettoyer complètement

**Installation :**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# macOS (Homebrew)
brew install postgresql@18
brew services start postgresql@18

# Windows
# Télécharger depuis https://www.postgresql.org/download/windows/
# Installer avec GUI
```

---

### Option 2 : Docker (Recommandée pour Projets Multiples)

**Avantages :**
- ✅ Isolation complète
- ✅ Plusieurs versions en parallèle
- ✅ Facile à supprimer/recréer
- ✅ Portable (même config partout)

**Inconvénients :**
- ❌ Nécessite Docker
- ❌ Légère overhead de performance

**Docker Compose (Recommandé) :**

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:18
    container_name: dev_postgres
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./postgresql.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

volumes:
  postgres_data:
```

**Démarrage :**
```bash
docker-compose up -d
```

**Connexion :**
```bash
docker exec -it dev_postgres psql -U devuser -d devdb
```

---

### Option 3 : Postgres.app (macOS uniquement)

**Avantages :**
- ✅ Interface graphique simple
- ✅ Plusieurs versions en parallèle
- ✅ Pas de configuration nécessaire

**Téléchargement :** https://postgresapp.com/

---

## Configuration Optimale pour Développement Local

### Scénario : Laptop 16 GB RAM, 4-8 CPU, SSD

```ini
# =====================================
# CONFIGURATION POSTGRESQL DÉVELOPPEMENT LOCAL
# Laptop : 16 GB RAM, 4-8 CPU, SSD
# Objectif : Simplicité + Logs détaillés
# =====================================

# ===================================
# MÉMOIRE (conservateur pour laptop)
# ===================================
shared_buffers = 512MB              # Seulement 3% de RAM (pas 25% comme en prod)
effective_cache_size = 4GB          # 25% de RAM (reste pour OS et IDE)
work_mem = 64MB                     # Généreux (peu de connexions)
maintenance_work_mem = 256MB        # Suffisant pour dev
temp_buffers = 32MB                 # Buffers pour tables temporaires

# ===================================
# CONNEXIONS (très peu en dev)
# ===================================
max_connections = 20                # Seulement 20 au lieu de 100-300

# ===================================
# PARALLÉLISATION (limité sur laptop)
# ===================================
max_worker_processes = 4            # Selon CPU
max_parallel_workers_per_gather = 2 # Pas trop pour ne pas saturer
max_parallel_workers = 4
max_parallel_maintenance_workers = 2

# ===================================
# WAL et CHECKPOINTS (performance > durabilité)
# ===================================
wal_level = minimal                 # Pas de réplication en dev
max_wal_size = 1GB                  # Petit (économiser disque)
min_wal_size = 80MB
checkpoint_timeout = 30min          # Espacé (moins d'I/O)
checkpoint_completion_target = 0.9

# IMPORTANT : Accepter risque de perte si crash
fsync = off                         # ⚠️ SEULEMENT EN DEV ! Jamais en prod
synchronous_commit = off            # ⚠️ SEULEMENT EN DEV !
full_page_writes = off              # ⚠️ SEULEMENT EN DEV !

# ===================================
# AUTOVACUUM (peu important en dev)
# ===================================
autovacuum = on                     # Garder activé mais pas urgent
autovacuum_max_workers = 2
autovacuum_naptime = 1min

# ===================================
# PLANIFICATEUR (standard)
# ===================================
random_page_cost = 1.1              # SSD sur laptop
effective_io_concurrency = 200

# ===================================
# STATISTIQUES (standard)
# ===================================
default_statistics_target = 100

# ===================================
# LOGGING (TRÈS DÉTAILLÉ pour apprendre)
# ===================================
logging_collector = on
log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 10MB            # Petit (économiser disque)
log_truncate_on_rotation = on       # Écraser anciens logs

# Préfixe de ligne très détaillé
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '

# Logger TOUTES les requêtes (apprendre SQL)
log_statement = 'all'               # 'all' = toutes les requêtes
log_duration = on                   # Temps d'exécution
log_min_duration_statement = 0      # Logger même requêtes rapides

# Logs de debug
log_connections = on
log_disconnections = on
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0                  # Logger tous fichiers temp
log_autovacuum_min_duration = 0

# ===================================
# MONITORING (activer tout)
# ===================================
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all               # Tracker aussi fonctions SQL

# ===================================
# SÉCURITÉ (minimal en dev)
# ===================================
# Accepter connexions locales sans password
# (dans pg_hba.conf : host all all 127.0.0.1/32 trust)
ssl = off                           # Pas de SSL en dev local
password_encryption = scram-sha-256 # Garder moderne quand même

# ===================================
# TIMEZONE et LOCALE
# ===================================
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'

# ===================================
# EXTENSIONS UTILES EN DEV
# ===================================
shared_preload_libraries = 'pg_stat_statements'
```

---

## Explications Détaillées

### 1. Mémoire : Pourquoi si Peu ?

#### **shared_buffers = 512MB (au lieu de 4GB en production)**

**Raison :**
- Votre laptop a 16 GB RAM mais :
  - OS (macOS/Linux/Windows) : 2-4 GB
  - Navigateur : 2-4 GB
  - IDE (VSCode, IntelliJ) : 1-2 GB
  - Autres applications : 2-4 GB
- **Reste pour PostgreSQL : ~4-6 GB maximum**

**Principe :** Laisser de la RAM pour vos outils de développement.

```
┌─────────────────────────┐
│  RAM Totale: 16 GB      │
├─────────────────────────┤
│  OS: 3 GB               │
│  Navigateur: 3 GB       │
│  IDE: 2 GB              │
│  Applications: 2 GB     │
├─────────────────────────┤
│  Disponible: 6 GB       │
│  PostgreSQL: 512 MB     │  ← shared_buffers
│  OS Cache: 5.5 GB       │  ← Le reste
└─────────────────────────┘
```

---

### 2. Durabilité : Sacrifier pour Performance

#### **fsync = off (⚠️ DANGEREUX EN PRODUCTION)**

**Qu'est-ce que fsync ?**
Forcer l'écriture physique sur disque à chaque COMMIT. C'est lent mais garantit la durabilité.

**Impact de fsync = off :**
- ✅ Performances : **2-5× plus rapide** (surtout insertions/updates)
- ❌ Risque : Perte de transactions récentes si crash/coupure électrique

**Pourquoi acceptable en dev ?**
- Données de test (pas de vraies données clients)
- Facile de régénérer (scripts de seed)
- Gain de productivité > risque de perte

**⚠️ RAPPEL CRITIQUE :** **JAMAIS fsync = off en production !**

---

#### **synchronous_commit = off**

**Impact :**
- Transaction considérée comme validée avant écriture WAL sur disque
- Gain : Latence divisée par 2-3

**Risque :**
- Perte possible des dernières transactions si crash (quelques secondes)

**Acceptable en dev local.**

---

### 3. Logs : Tout Logger pour Apprendre

#### **log_statement = 'all'**

**Effet :** PostgreSQL loge **toutes** les requêtes SQL exécutées.

**Exemple de log :**
```
2025-11-21 10:30:45.123 [12345]: user=devuser,db=devdb,app=psql,client=127.0.0.1 LOG:  statement: SELECT * FROM users WHERE email = 'test@example.com';
2025-11-21 10:30:45.125 [12345]: user=devuser,db=devdb,app=psql,client=127.0.0.1 LOG:  duration: 2.456 ms
```

**Pourquoi utile en dev ?**
- 👀 Voir exactement ce que votre ORM (Django, Rails, SQLAlchemy) génère
- 🐛 Déboguer problèmes de permissions
- 📚 Apprendre SQL en observant

**⚠️ En production :** `log_statement = 'none'` (trop de logs = disque plein)

---

#### **log_min_duration_statement = 0**

Logue le temps d'exécution de **toutes** les requêtes.

**Utilité :** Identifier les requêtes lentes même en dev.

---

### 4. Connexions : Très Peu Nécessaires

#### **max_connections = 20**

**Raison :**
- En dev local, vous êtes seul
- Votre application teste avec 1-5 connexions max
- 20 connexions = largement suffisant

**Économie :** Chaque connexion = ~10 MB RAM → 20 connexions = 200 MB vs 1000 connexions = 10 GB

---

### 5. Autovacuum : Moins Critique

En dev, vous créez/supprimez/recréez des tables fréquemment. L'autovacuum est moins critique qu'en production.

**Configuration simple :**
```ini
autovacuum = on  # Garder activé
autovacuum_naptime = 1min  # Vérifier toutes les minutes (suffisant)
```

---

## Configuration pg_hba.conf pour Développement

Le fichier `pg_hba.conf` contrôle l'authentification. En dev local, on simplifie au maximum.

**Localisation :**
```bash
# Trouver le fichier
psql -U postgres -c "SHOW hba_file;"
```

**Configuration dev local (pg_hba.conf) :**
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Connexions locales : TRUST (pas de password)
local   all             all                                     trust

# Connexions localhost : TRUST
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust

# Connexions réseau local (si besoin) : password
# host    all             all             192.168.1.0/24          scram-sha-256
```

**Explication :**
- `trust` : Pas de mot de passe nécessaire (⚠️ SEULEMENT EN DEV)
- `127.0.0.1` : Localhost uniquement
- Pas besoin de gérer des mots de passe compliqués en dev

**Recharger la config :**
```sql
SELECT pg_reload_conf();
```

---

## Extensions Utiles en Développement

### 1. pg_stat_statements : Analyser les Requêtes

**Installation :**
```sql
CREATE EXTENSION pg_stat_statements;
```

**Configuration (postgresql.conf) :**
```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

**Utilisation :**
```sql
-- Top 10 requêtes les plus lentes
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

### 2. auto_explain : Expliquer Automatiquement

**Installation :**
```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = 100  # Log EXPLAIN pour requêtes > 100ms
auto_explain.log_analyze = on
auto_explain.log_buffers = on
```

**Effet :** PostgreSQL logue automatiquement EXPLAIN ANALYZE pour requêtes lentes.

**Exemple de log :**
```
LOG:  duration: 234.567 ms  plan:
Query Text: SELECT * FROM orders WHERE created_at > '2024-01-01'
Seq Scan on orders  (cost=0.00..5000.00 rows=10000 width=100) (actual time=0.123..200.456 rows=10234)
  Filter: (created_at > '2024-01-01'::date)
  Rows Removed by Filter: 50000
```

**Pourquoi utile ?** Comprendre pourquoi une requête est lente **sans** avoir à lancer EXPLAIN manuellement.

---

### 3. pgcrypto : Génération de Données de Test

```sql
CREATE EXTENSION pgcrypto;

-- Générer des UUID
SELECT gen_random_uuid();

-- Générer des données aléatoires
INSERT INTO users (email, name)
SELECT
    'user' || i || '@example.com',
    'User ' || i
FROM generate_series(1, 1000) AS i;
```

---

### 4. pg_trgm : Recherche Floue

```sql
CREATE EXTENSION pg_trgm;

-- Recherche par similarité
SELECT * FROM products
WHERE name % 'ipone';  -- Trouve "iPhone"
```

---

## Commandes psql Essentielles pour Développeurs

### Connexion et Navigation

```bash
# Connexion
psql -U devuser -d devdb

# Lister bases de données
\l

# Se connecter à une base
\c devdb

# Lister tables
\dt

# Décrire une table
\d users
\d+ users  # Plus détaillé

# Lister index
\di

# Lister fonctions
\df

# Lister extensions
\dx
```

---

### Configuration psql

**Fichier ~/.psqlrc (configuration personnelle) :**

```sql
-- Afficher temps d'exécution
\timing on

-- Format étendu pour requêtes (plus lisible)
\x auto

-- Prompt personnalisé
\set PROMPT1 '%[%033[1;32m%]%n@%/%[%033[0m%]%# '

-- Historique avec timestamps
\set HISTFILE ~/.psql_history- :DBNAME

-- Ignorer doublons dans historique
\set HISTCONTROL ignoredups

-- Afficher NULL clairement
\pset null '¤'

-- Messages d'erreur verbeux
\set VERBOSITY verbose

-- Complétion automatique (majuscules)
\set COMP_KEYWORD_CASE upper
```

**Effet :** Chaque fois que vous lancez `psql`, ces paramètres sont appliqués.

---

### Requêtes Utiles

#### Taille des Tables

```sql
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

#### Voir les Connexions Actives

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity
WHERE datname = current_database();
```

---

#### Tuer une Connexion

```sql
-- Terminer proprement
SELECT pg_terminate_backend(12345);  -- Remplacer par PID

-- Forcer (si ne répond pas)
SELECT pg_cancel_backend(12345);
```

---

## Gestion des Données de Test

### 1. Seed Script (init.sql)

**Créer un fichier init.sql :**

```sql
-- Créer schéma de test
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insérer données de test
INSERT INTO users (email, name) VALUES
    ('alice@example.com', 'Alice'),
    ('bob@example.com', 'Bob'),
    ('charlie@example.com', 'Charlie');

INSERT INTO posts (user_id, title, content) VALUES
    (1, 'First Post', 'Hello World!'),
    (1, 'Second Post', 'PostgreSQL is awesome'),
    (2, 'Bob Post', 'Learning SQL');

-- Créer index
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

**Charger le script :**
```bash
psql -U devuser -d devdb -f init.sql
```

**Avec Docker :**
Placer `init.sql` dans `/docker-entrypoint-initdb.d/` (exécuté au premier démarrage).

---

### 2. Générateur de Données Massives

```sql
-- Générer 100,000 utilisateurs
INSERT INTO users (email, name)
SELECT
    'user' || i || '@example.com',
    'User ' || i
FROM generate_series(1, 100000) AS i;

-- Générer 1,000,000 posts
INSERT INTO posts (user_id, title, content)
SELECT
    (RANDOM() * 100000)::INT + 1,  -- user_id aléatoire
    'Post ' || i,
    'Content for post ' || i
FROM generate_series(1, 1000000) AS i;

-- Créer index après insertion (plus rapide)
CREATE INDEX idx_posts_user_id ON posts(user_id);
```

---

### 3. Reset Complet de la Base

**Script reset.sh :**

```bash
#!/bin/bash

# Supprimer et recréer la base
psql -U postgres -c "DROP DATABASE IF EXISTS devdb;"
psql -U postgres -c "CREATE DATABASE devdb OWNER devuser;"

# Charger le schéma et données
psql -U devuser -d devdb -f init.sql

echo "Base de données réinitialisée avec succès !"
```

**Utilisation :**
```bash
chmod +x reset.sh
./reset.sh
```

---

## Optimisations pour Développement Rapide

### 1. Désactiver Contraintes Temporairement

**Pendant import massif de données :**

```sql
-- Désactiver contraintes
ALTER TABLE posts DISABLE TRIGGER ALL;

-- Importer données
COPY posts FROM '/data/posts.csv' WITH CSV;

-- Réactiver contraintes
ALTER TABLE posts ENABLE TRIGGER ALL;
```

**⚠️ Attention :** Peut créer des incohérences si données invalides.

---

### 2. UNLOGGED Tables pour Tests Temporaires

**Concept :** Tables sans WAL = 2-3× plus rapides.

```sql
-- Table temporaire ultra-rapide
CREATE UNLOGGED TABLE test_data (
    id SERIAL PRIMARY KEY,
    data TEXT
);

-- Charger données rapidement
INSERT INTO test_data (data)
SELECT md5(random()::TEXT)
FROM generate_series(1, 1000000);

-- Nettoyer
DROP TABLE test_data;
```

**⚠️ Risque :** Données perdues si crash PostgreSQL (acceptable pour tests).

---

### 3. EXPLAIN (ANALYZE) pour Comprendre

**Toujours vérifier les plans de requête :**

```sql
-- Plan sans exécution
EXPLAIN SELECT * FROM posts WHERE user_id = 1;

-- Plan avec exécution et statistiques
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM posts WHERE user_id = 1;

-- Format JSON (pour outils de visualisation)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM posts WHERE user_id = 1;
```

**Outils de visualisation :**
- https://explain.dalibo.com/
- https://explain.depesz.com/

---

## Backup et Restore en Développement

### Backup Rapide

```bash
# Dump complet
pg_dump -U devuser devdb > backup.sql

# Dump compressé (plus rapide)
pg_dump -U devuser devdb | gzip > backup.sql.gz

# Dump format custom (plus flexible)
pg_dump -U devuser -Fc devdb > backup.dump
```

---

### Restore

```bash
# Depuis SQL
psql -U devuser devdb < backup.sql

# Depuis gzip
gunzip -c backup.sql.gz | psql -U devuser devdb

# Depuis custom format
pg_restore -U devuser -d devdb backup.dump
```

---

### Snapshot Docker (Très Rapide)

**Avec Docker :**

```bash
# Créer snapshot
docker commit dev_postgres dev_postgres_snapshot

# Restaurer snapshot
docker stop dev_postgres
docker rm dev_postgres
docker run -d --name dev_postgres dev_postgres_snapshot

# Ou avec docker-compose
docker-compose down
docker-compose up -d
```

---

## Intégration avec Outils de Développement

### 1. Connection depuis Applications

**Python (psycopg3) :**
```python
import psycopg

conn = psycopg.connect(
    host="localhost",
    port=5432,
    dbname="devdb",
    user="devuser",
    password="devpassword"
)

cursor = conn.cursor()
cursor.execute("SELECT * FROM users LIMIT 10")
print(cursor.fetchall())
```

---

**Node.js (node-postgres) :**
```javascript
const { Client } = require('pg');

const client = new Client({
  host: 'localhost',
  port: 5432,
  database: 'devdb',
  user: 'devuser',
  password: 'devpassword',
});

await client.connect();
const res = await client.query('SELECT * FROM users LIMIT 10');
console.log(res.rows);
await client.end();
```

---

**Ruby (pg gem) :**
```ruby
require 'pg'

conn = PG.connect(
  host: 'localhost',
  port: 5432,
  dbname: 'devdb',
  user: 'devuser',
  password: 'devpassword'
)

result = conn.exec('SELECT * FROM users LIMIT 10')
result.each { |row| puts row }
conn.close
```

---

### 2. Variables d'Environnement

**Fichier .env :**
```
DATABASE_URL=postgresql://devuser:devpassword@localhost:5432/devdb
PGHOST=localhost
PGPORT=5432
PGDATABASE=devdb
PGUSER=devuser
PGPASSWORD=devpassword
```

**Chargement :**
```bash
# Bash
source .env

# psql (lit automatiquement variables PG*)
psql
```

---

### 3. Migrations de Schéma

**Outils populaires :**

- **Flyway** (Java, multi-langages)
- **Liquibase** (Java, multi-langages)
- **Alembic** (Python)
- **Django Migrations** (Python)
- **Rails Migrations** (Ruby)
- **Knex.js** (Node.js)

**Exemple simple (SQL pur) :**

```sql
-- migrations/001_create_users.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- migrations/002_add_name_to_users.sql
ALTER TABLE users ADD COLUMN name VARCHAR(100);

-- migrations/003_create_posts.sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    title VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Appliquer migrations :**
```bash
for file in migrations/*.sql; do
    echo "Applying $file"
    psql -U devuser -d devdb -f "$file"
done
```

---

## Outils Graphiques pour Développeurs

### 1. pgAdmin (Officiel)

**Installation :**
- https://www.pgadmin.org/download/

**Fonctionnalités :**
- Interface Web complète
- Explorateur de schéma
- Éditeur SQL avec autocomplétion
- EXPLAIN visualisé
- Monitoring

---

### 2. DBeaver (Multi-base)

**Installation :**
- https://dbeaver.io/download/

**Fonctionnalités :**
- Support multi-bases (PostgreSQL, MySQL, etc.)
- Interface moderne
- ER Diagrams
- Génération de données de test
- Import/Export

---

### 3. TablePlus (macOS/Windows/Linux)

**Installation :**
- https://tableplus.com/

**Fonctionnalités :**
- Interface native élégante
- Très rapide
- Édition inline
- Queries favorites

---

### 4. psql avec Plugins

**pgcli : psql amélioré**

```bash
# Installation
pip install pgcli

# Utilisation
pgcli -h localhost -U devuser devdb
```

**Fonctionnalités :**
- Autocomplétion intelligente
- Coloration syntaxique
- Affichage amélioré

---

## Debugging et Troubleshooting

### 1. Logs : Où les Trouver ?

**Localisation logs :**
```sql
SHOW log_directory;
SHOW data_directory;
```

**Typique :**
- Linux : `/var/log/postgresql/`
- macOS (Homebrew) : `/usr/local/var/log/`
- Docker : `docker logs dev_postgres`

**Lire logs en temps réel :**
```bash
# Linux/macOS
tail -f /var/log/postgresql/postgresql-*.log

# Docker
docker logs -f dev_postgres
```

---

### 2. Requêtes Lentes : Identifier

**Activer logging dans session :**
```sql
SET log_min_duration_statement = 100;  -- Log si > 100ms

-- Votre requête
SELECT * FROM posts WHERE ...;
```

**Vérifier logs pour voir le temps.**

---

### 3. Locks : Qui Bloque Quoi ?

```sql
-- Voir les locks actifs
SELECT
    l.pid,
    l.mode,
    l.granted,
    a.usename,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted;
```

---

### 4. Erreurs Courantes et Solutions

#### Erreur : "Too many connections"

**Cause :** Atteint `max_connections`.

**Solution temporaire :**
```sql
-- Tuer connexions inutilisées
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle' AND pid != pg_backend_pid();
```

**Solution permanente :**
```ini
# postgresql.conf
max_connections = 50  # Augmenter si nécessaire en dev
```

---

#### Erreur : "Could not connect to server"

**Causes possibles :**
1. PostgreSQL pas démarré
2. Mauvais port
3. pg_hba.conf bloque

**Vérification :**
```bash
# Est-ce que PostgreSQL tourne ?
ps aux | grep postgres

# Tester connexion
psql -U devuser -h localhost -d devdb

# Vérifier logs
tail /var/log/postgresql/postgresql-*.log
```

---

#### Erreur : "Permission denied"

**Cause :** Droits insuffisants.

**Solution :**
```sql
-- Donner tous les droits (dev uniquement)
GRANT ALL PRIVILEGES ON DATABASE devdb TO devuser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO devuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO devuser;
```

---

## Checklist Développement Local

### Installation
- [ ] PostgreSQL 18 installé (système, Docker, ou Postgres.app)
- [ ] Connexion psql fonctionnelle
- [ ] Extension pg_stat_statements activée
- [ ] Outil graphique installé (pgAdmin, DBeaver, TablePlus)

### Configuration
- [ ] `shared_buffers` = 512MB (conservateur)
- [ ] `work_mem` = 64MB (généreux pour dev)
- [ ] `max_connections` = 20 (suffisant pour dev)
- [ ] `fsync = off` et `synchronous_commit = off` (⚠️ dev seulement)
- [ ] `log_statement = 'all'` (apprendre SQL)
- [ ] `log_min_duration_statement = 0` (temps d'exécution)

### pg_hba.conf
- [ ] `trust` pour connexions localhost (pas de password)
- [ ] Recharger config : `SELECT pg_reload_conf();`

### Données de Test
- [ ] Script `init.sql` créé avec schéma de base
- [ ] Données de test insérées
- [ ] Script `reset.sh` pour réinitialisation rapide

### Outils
- [ ] ~/.psqlrc configuré (\timing, \x auto)
- [ ] pgcli installé (optionnel)
- [ ] Backup/restore testé

---

## Bonnes Pratiques pour Développeurs

### 1. Utiliser des Transactions pour Tester

```sql
-- Commencer transaction
BEGIN;

-- Tester des modifications
UPDATE users SET email = 'new@example.com' WHERE id = 1;
DELETE FROM posts WHERE id = 100;

-- Voir le résultat
SELECT * FROM users WHERE id = 1;

-- Annuler (ne pas enregistrer)
ROLLBACK;

-- Ou valider si OK
-- COMMIT;
```

**Avantage :** Tester sans risque, annuler avec ROLLBACK.

---

### 2. Versionner le Schéma (Git)

**Structure projet :**
```
my_project/
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_add_name.sql
│   └── 003_create_posts.sql
├── seeds/
│   └── test_data.sql
├── docker-compose.yml
└── postgresql.conf
```

**Committer dans Git :** Toute l'équipe a le même schéma.

---

### 3. Documenter les Requêtes Complexes

```sql
-- ============================================
-- Calcul du chiffre d'affaires par produit
-- pour le mois en cours
-- ============================================
SELECT
    p.name AS product_name,
    SUM(oi.quantity * oi.price) AS total_revenue,
    COUNT(DISTINCT o.id) AS num_orders
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
WHERE o.created_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY p.id, p.name
ORDER BY total_revenue DESC;
```

---

### 4. Ne Jamais Copier postgresql.conf de Production vers Dev

**Raison :**
- Config prod optimisée pour serveur 128GB RAM
- Sur laptop 16GB → crash ou lenteur extrême

**Toujours partir d'une config dédiée dev.**

---

## Ressources pour Apprendre PostgreSQL

### Documentation Officielle
- **Tutorial PostgreSQL** : https://www.postgresql.org/docs/18/tutorial.html
- **SQL Language** : https://www.postgresql.org/docs/18/sql.html

### Livres
- *PostgreSQL: Up and Running* (O'Reilly) : Introduction pratique
- *The Art of PostgreSQL* (Dimitri Fontaine) : SQL avancé
- *Mastering PostgreSQL* : Administration complète

### Cours en Ligne
- **Codecademy** : SQL fundamentals
- **Udemy** : PostgreSQL pour développeurs
- **PluralSight** : PostgreSQL Deep Dive

### Communautés
- **Reddit** : r/PostgreSQL
- **Stack Overflow** : Tag [postgresql]
- **Discord** : PostgreSQL Community
- **Mailing list** : pgsql-general

### Blogs
- **Percona Blog** : https://www.percona.com/blog/
- **CrunchyData Blog** : https://www.crunchydata.com/blog
- **2ndQuadrant Blog** : https://www.2ndquadrant.com/en/blog/

---

## Conclusion

La configuration PostgreSQL pour développement local privilégie :

### Principes Clés

1. **Simplicité** : Configuration minimale et compréhensible
2. **Feedback** : Logs détaillés pour apprendre et déboguer
3. **Performance raisonnable** : Sacrifier durabilité pour vitesse
4. **Facilité de reset** : Scripts de seed et backup/restore simples
5. **Ressources modérées** : Laisser de la RAM pour IDE et navigateur

### Différences Critiques Dev vs Production

| Paramètre | Dev Local | Production |
|-----------|-----------|------------|
| **fsync** | off ⚠️ | on |
| **synchronous_commit** | off ⚠️ | on |
| **shared_buffers** | 512MB | 16-32GB |
| **max_connections** | 20 | 200-500 |
| **log_statement** | 'all' | 'none' |
| **log_min_duration_statement** | 0 | 100-1000ms |
| **Authentification** | trust (local) | scram-sha-256 |

### Rappel Essentiel

**⚠️ Ne JAMAIS utiliser la config dev en production !**

Les paramètres comme `fsync = off` sont **dangereux** et causent des pertes de données en cas de crash.

### Prochaines Étapes

1. Installer PostgreSQL (système ou Docker)
2. Appliquer configuration dev locale
3. Créer schéma de test avec `init.sql`
4. Expérimenter avec `psql` et outils graphiques
5. Pratiquer SQL avec données de test
6. Utiliser EXPLAIN pour comprendre les plans
7. Explorer extensions (pg_stat_statements, pgcrypto)

Bon développement avec PostgreSQL ! 🚀🐘

⏭️ [Checklist de Performance](/annexes/checklist-performance/README.md)
